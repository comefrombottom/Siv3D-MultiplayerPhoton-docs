# 動かしてみよう！
## 簡単なチャットルームを作成

??? summary "サンプルコード"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"

    class MyClient : public Multiplayer_Photon
    {
    public:
        MyClient()
            :Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)),U"1.0",Verbose::Yes)
        {}
    private:

        //sendEventによって送られてきたイベントに反応し呼ばれる。
        void customEventAction(const LocalPlayerID playerID, const uint8 eventCode, Deserializer<MemoryViewReader>& reader) override
        {
            if (eventCode == 1)
            {
                String text;
                reader(text);
                Print << getUserName(playerID) << U":" << text;
            }
        }
    };

    void Main()
    {
        Window::Resize(1280, 720);

        MyClient client;

        TextEditState playerName{ U"siv3d-kun" };

        TextEditState sendText;

        Font font(20);

        while (System::Update())
        {
            if (client.isActive())
            {
                client.update();
            }

            ClientState state = client.getClientState();

            Vec2 stateTextPos(1000, 5);

            //クライアントは7つの状態を持つ
            switch (state)
            {
            case s3d::ClientState::Disconnected:
                font(U"Disconnected").draw(stateTextPos);
                break;
            case s3d::ClientState::ConnectingToLobby:
                font(U"ConnectingToLobby").draw(stateTextPos);
                break;
            case s3d::ClientState::InLobby:
                Scene::Rect().draw(Palette::Steelblue);
                font(U"InLobby").draw(stateTextPos);

                //ロビー内に存在するルームを列挙
                font(U"Rooms").draw(Vec2{ 650, 5 });
                {
                    auto roomNameList = client.getRoomNameList();
                    double y = 0;
                    for (size_t i = 0; i < roomNameList.size(); ++i)
                    {
                        //クリックで入室
                        if (SimpleGUI::Button(roomNameList[i], Vec2{ 650, (y += 40) }, 330, client.isInLobby()))
                        {
                            client.joinRoom(roomNameList[i]);
                        }
                    }
                }
                break;
            case s3d::ClientState::JoiningRoom:
                font(U"JoiningRoom").draw(stateTextPos);
                break;
            case s3d::ClientState::InRoom:
                Scene::Rect().draw(Palette::Sienna);
                font(U"InRoom").draw(stateTextPos);
                font(U"Room name : ", client.getCurrentRoomName()).draw(Arg::topRight(Scene::Width() - 20, Scene::Height() - 90));

                SimpleGUI::TextBox(sendText, Vec2{ 20, Scene::Height()-50}, 1000, unspecified, client.isInRoom());

                //ボタンを押すかエンターキーを押すと文字列を送信
                if (SimpleGUI::Button(U">>>", Vec2{ 1040, Scene::Height() - 50 }, 160, client.isInRoom()) or sendText.enterKey)
                {
                    //イベントコード1で文字列を送信
                    client.sendEvent(MultiplayerEvent(1), sendText.text);
                    Print << U"自分：" << sendText.text;
                    sendText.clear();
                }

                break;
            case s3d::ClientState::LeavingRoom:
                font(U"LeavingRoom").draw(stateTextPos);
                break;
            case s3d::ClientState::Disconnecting:
                font(U"Disconnecting").draw(stateTextPos);
                break;
            default:
                break;
            }


            //コントロールパネル

            double y = 0;
            if (SimpleGUI::Button(U"Disconnect", Vec2(1000, y+=40), unspecified, client.isActive()))
            {
                client.disconnect();
            }

            SimpleGUI::TextBox(playerName, Vec2(1000, y += 40), 200, unspecified, state == ClientState::Disconnected);

            if (SimpleGUI::Button(U"Connect", Vec2(1000, y+=40), unspecified, state == ClientState::Disconnected))
            {
                //名前、リージョンを指定して接続
                client.connect(playerName.text, U"jp");
            }

            if (SimpleGUI::Button(U"Create Room", Vec2{ 1000, (y += 40) }, 160, client.isInLobby()))
            {
                //部屋名は被ってはいけないのでランダムな文字列を付加
                const RoomName roomName = (client.getUserName() + U"'s room-" + ToHex(RandomUint32()));

                //ロビー内に部屋を作成
                client.createRoom(roomName);
            }

            if (SimpleGUI::Button(U"Leave Room", Vec2{ 1000, (y += 40) }, 160, client.isInRoom()))
            {
                client.leaveRoom();
            }

            if (SimpleGUI::Button(U"ClearPrint()", Vec2{ 1000, (y += 40) }, 160))
            {
                ClearPrint();
            }
            
        }
    }
    ```

上記サンプルコードを貼り付け、実行してみましょう。`Connect`を押し、少しまって画面が青くなったら接続成功です。その後`Create Room`を押すと部屋を作成と同時に入室でき、画面が茶色くなったら部屋にいる状態です。画面下のテキストボックスからチャットを打てますが、今は一人しかいないため意味はないです。

### 一人で通信を確認する

一度実行Windowを閉じて、プロジェクトのAppフォルダを開きます。先ほど作った実行ファイル（.exeファイル）を複数回ダブルクリックして起動すれば、一台のパソコンで通信が行えているのを確認できます。

一人目が部屋を作成したら、二人目はRoomsの下に部屋の名前で入室ボタンが表示されるはずなので、そこから同じ部屋に入れます。

![alt text](image-6.png)
