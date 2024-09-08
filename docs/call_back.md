# コールバックを編集する
サーバーに接続、部屋に入室、部屋から退出など、あらゆる操作やイベントの結果を通知するために呼ばれる関数をコールバック関数といいます。Multiplayer_Photon関数を継承した自作クラス`MyClient`のなかでこれらのコールバック関数をオーバーライドすることで、コールバックを編集することが出来ます。

コールバック関数は、`.update()` のタイミングで呼ばれます。

## 操作に対して返すコールバック
client.○○○()という操作に対して○○○Return()を返す。

- `void connectReturn(int32 errorCode, const String& errorString, const String& region, const String& cluster);`

- `void disconnectReturn();`

- `void createRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString);`

- `void joinRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString);`

- `void leaveRoomReturn(int32 errorCode, const String& errorString);`

- `void joinOrCreateRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString);`

- `void joinRandomRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString);`

- `void joinRandomOrCreateRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString);`

## ルームへのプレイヤーの入退室に反応するコールバック

- `void joinRoomEventAction(const LocalPlayer& newPlayer, const Array<LocalPlayerID>& playerIDs, bool isSelf);`

    自身がルームに入室したり、部屋に他のプレイヤーが入ってきた際に通知する。入ってきた人を含めた、部屋にいる人全員に通知される。
    
    `joinRoomReturn()`と紛らわしいが、`joinRoomReturn()`はあくまで自分が`.joinRoom()`を行った時のみ自分に対して返ってくる。

- `void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive);`

    同じルームにいた他のプレイヤーが退室した際に通知する。部屋にいる人全員に通知される。抜けた人は受け取らない。

    `leaveRoomReturn()`はあくまで自身が`.leaveRoom()`をしたときに自分に対して返ってくるもの。

## イベント受信
- `void customEventAction(LocalPlayerID playerID, uint8 eventCode, Deserializer<MemoryViewReader>& reader);`

## その他
- `void connectionErrorReturn(int32 errorCode);`

    意図的でなく切断が切れた際に通知する。

- `void onRoomListUpdate();`

    ロビーから取得できるルームリストが更新された際に通知する。

- `void onHostChange(LocalPlayerID newHostPlayerID, LocalPlayerID oldHostPlayerID);`

    部屋のホストが変わった際に通知する。

- `void onRoomPropertiesChange(const HashTable<String, String>& changes);`


## サンプルコード

試しにチャットルームのサンプルに各コールバックを追加してみます。編集したコールバック以外のPrintを抑えるために`Verbose::No`としています。

??? summary "サンプルコード"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"


    class MyClient : public Multiplayer_Photon
    {
    public:
        MyClient()
            :Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)),U"1.0",Verbose::No)
        {}
    private:

        void connectReturn(int32 errorCode, const String& errorString, const String& region, const String& cluster) override{
            if (errorCode)
            {
                Print << U"connectReturn error!!! " << errorString;
            }
            else
            {
                Print << U"connectReturn!!!";
            }
        }

        void disconnectReturn() override{
            Print << U"disconnectReturn!!!";
        }

        void createRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString) override{
            if (errorCode)
            {
                Print << U"createRoomReturn error!!! " << errorString;
            }
            else
            {
                Print << U"createRoomReturn!!!";
            }
        }

        void joinRoomReturn(LocalPlayerID playerID, int32 errorCode, const String& errorString) override{
            if (errorCode)
            {
                Print << U"joinRoomReturn error!!! " << errorString;
            }
            else
            {
                Print << U"joinRoomReturn!!!";
            }
        }

        void leaveRoomReturn(int32 errorCode, const String& errorString) override{
            if (errorCode)
            {
                Print << U"leaveRoomReturn error!!! " << errorString;
            }
            else
            {
                Print << U"leaveRoomReturn!!!";
            }
        }

        void joinRoomEventAction(const LocalPlayer& newPlayer, const Array<LocalPlayerID>& playerIDs, bool isSelf) override{
            Print << U"joinRoomEventAction!!!";
        }

        void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive) override{
            Print << U"leaveRoomEventAction!!!";
        }

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