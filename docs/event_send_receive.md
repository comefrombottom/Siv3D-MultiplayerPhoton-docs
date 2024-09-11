# イベントの送受信
同じルームにいるプレイヤーは互いにイベントとしてデータを送信することが出来ます。プレイヤー同士の通知、データの同期はこの機能を介して行われるため、マルチプレイヤーゲームの中核を担う機能です。

## イベントの送信
`sendEvent(const MultiplayerEvent& event, ...)`を用いてイベントを送信します。第一引数に`MultiplayerEvent`、第二引数以降に送信するデータを任意の個数連ねることが出来ます。

`MultiplayerEvent`はイベントコードとイベント送信のオプションを持った型です。コンストラクタの第一引数でイベントコード、第二引数でターゲットを指定できます。デフォルトでは自分以外です。

イベントコードは1から199までの整数を指定することが出来ます。

いくつか例を上げます。

- イベントコード 1 で、こんにちは。と送信
    ```cpp
    client.sendEvent(MultiplayerEvent(1),　String(U"こんにちは。"));
    ```

    `MultiplayerEvent`は省略して書ける。

    ```cpp
    client.sendEvent({ 1 },　String(U"こんにちは。"));
    ```

- イベントコード 2 で、Vec2型, int型, double型のデータを送信
    ```cpp
    client.sendEvent({ 2 }, Cursor::PosF(), 12, 3.4);
    ```

- イベントコード 3 で、自分含めた全てのプレイヤーに 3.14 を送信

    ```cpp
    client.sendEvent(MultiplayerEvent(3, ReceiverOption::All), 3.14);
    ```
    ２引数の場合も`MultiplayerEvent`は省略して書ける。
    ```cpp
    client.sendEvent({ 3, ReceiverOption::All }, 3.14);
    ```

- ターゲットの`LocalPlayerID`を指定して送信。この例では`LocalPlayerID`が２と３のプレイヤーが受信する。
    ```cpp
    client.sendEvent({ 4, Array<LocalPlayerID>{2, 3} }, Circle(10,10,10), RectF(10,10,20,30));
    ```

- イベントコードのみを送信してもよい。
    ```cpp
    client.sendEvent({ 5 });
    ```

## 送信可能な型
`sendEvent()`は内部で`Serializer<MemoryWriter>{}`を使用しています。つまり、Siv3Dで規定されたシリアライズ可能な型は全て送信可能です。また、独自に作った型も、以下のようにメンバ関数を定義することでシリアライズに対応し、送信可能な型にすることが出来ます。
```cpp
// ユーザ定義型
struct MyData
{
	String word;

	Point pos;

	// シリアライズに対応させるためのメンバ関数を定義する
	template <class Archive>
	void SIV3D_SERIALIZE(Archive& archive)
	{
		archive(word, pos);
	}
};
```

## イベントの受信
イベントはコールバック関数である、

`void customEventAction(LocalPlayerID playerID, uint8 eventCode, Deserializer<MemoryViewReader>& reader);`

をオーバーライドすることで受け取ることが出来ます。

例えば、先ほどの`sendEvent()`の例に全て対応するなら、

```cpp
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
            Print << U"イベントコード1を受信：" << getUserName(playerID) << U":" << text;
        }

        if (eventCode == 2)
        {
            Vec2 v;
            int32 n;
            double d;
            reader(v, n, d);
            Print << U"イベントコード2を受信：" << getUserName(playerID) << U":" << v << U"," << n << U"," << d;
        }

        if (eventCode == 3)
        {
            double d;
            reader(d);
            Print << U"イベントコード3を受信：" << getUserName(playerID) << U":" << d;
        }

        if (eventCode == 4)
        {
            Circle circle;
            RectF rect;
            reader(circle, rect);
            Print << U"イベントコード4を受信：" << getUserName(playerID) << U":" << circle << U"," << rect;
        }

        if (eventCode == 5)
        {
            Print << U"イベントコード5を受信：" << getUserName(playerID);
        }
    }
}
```
のようになります。

## RegisterEventCallback
customEventAction()のオーバーロードによる受信は、全てのイベントが一つの関数を呼ぶため、customEventAction()が肥大化してしまう問題があります。そこで、独自のメンバ関数を登録し、イベントコードに応じて別の関数が呼ばれるようにできる仕組みが用意されています。

引数に`LocalPlayerID`と送信した型を連ねたメンバ関数を独自定義し、`RegisterEventCallback()`にイベントコードと関数ポインタを与え、コンストラクタなどではじめに登録しておきます。これにより、`RegisterEventCallback()`によってイベントコードが登録されているイベントは、対応したメンバ関数を呼び出すようになります。登録されていないイベントコードのイベントが送られてきた場合は通常通り`customEventAction()`が反応します。

`RegisterEventCallback()`を用いて先ほどのイベント受信コードを書き換えると、

```cpp
class MyClient : public Multiplayer_Photon
{
public:
	MyClient()
		: Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::No)
	{
		RegisterEventCallback(1, &MyClient::eventReceived_1);
		RegisterEventCallback(2, &MyClient::eventReceived_2);
		RegisterEventCallback(3, &MyClient::eventReceived_3);
		RegisterEventCallback(4, &MyClient::eventReceived_4);
		RegisterEventCallback(5, &MyClient::eventReceived_5);
	}

private:

	void eventReceived_1(LocalPlayerID playerID, const String& text) {
		Print << U"イベントコード1を受信：" << getUserName(playerID) << U":" << text;
	}

	void eventReceived_2(LocalPlayerID playerID, const Vec2& v, int32 n, double d) {
		Print << U"イベントコード2を受信：" << getUserName(playerID) << U":" << v << U"," << n << U"," << d;
	}

	void eventReceived_3(LocalPlayerID playerID, double d) {
		Print << U"イベントコード3を受信：" << getUserName(playerID) << U":" << d;
	}

	void eventReceived_4(LocalPlayerID playerID, const Circle& circle, const RectF& rect) {
		Print << U"イベントコード4を受信：" << getUserName(playerID) << U":" << circle << U"," << rect;
	}

	void eventReceived_5(LocalPlayerID playerID) {
		Print << U"イベントコード5を受信：" << getUserName(playerID);
	}
};
```

となります。


??? summary "実際に動かせるサンプル"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"


    class MyClient : public Multiplayer_Photon
    {
    public:
        MyClient()
            : Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::No)
        {
            RegisterEventCallback(1, &MyClient::eventReceived_1);
            RegisterEventCallback(2, &MyClient::eventReceived_2);
            RegisterEventCallback(3, &MyClient::eventReceived_3);
            RegisterEventCallback(4, &MyClient::eventReceived_4);
            RegisterEventCallback(5, &MyClient::eventReceived_5);
        }

    private:

        void eventReceived_1(LocalPlayerID playerID, const String& text) {
            Print << U"イベントコード1を受信：" << getUserName(playerID) << U":" << text;
        }

        void eventReceived_2(LocalPlayerID playerID, const Vec2& v, int32 n, double d) {
            Print << U"イベントコード2を受信：" << getUserName(playerID) << U":" << v << U"," << n << U"," << d;
        }

        void eventReceived_3(LocalPlayerID playerID, double d) {
            Print << U"イベントコード3を受信：" << getUserName(playerID) << U":" << d;
        }

        void eventReceived_4(LocalPlayerID playerID, const Circle& circle, const RectF& rect) {
            Print << U"イベントコード4を受信：" << getUserName(playerID) << U":" << circle << U"," << rect;
        }

        void eventReceived_5(LocalPlayerID playerID) {
            Print << U"イベントコード5を受信：" << getUserName(playerID);
        }
    };

    void Main()
    {
        Window::Resize(1280, 720);

        MyClient client;

        while (System::Update())
        {
            client.update();

            if (client.isDisconnected()) {
                client.connect(U"player", U"jp");
            }

            if (client.isInLobby())
            {
                client.joinRandomOrCreateRoom(U"room");
            }

            if (client.isInRoom())
            {
                Scene::Rect().draw(Palette::Sienna);
            }
            else {
                //待機画面のクルクル
                size_t t = Floor(fmod(Scene::Time() / 0.1, 8));
                for (size_t i : step(8)) {
                    Vec2 n = Circular(1, i * Math::TwoPi / 8);
                    Line(Scene::Center() + n * 10, Arg::direction(n * 10)).draw(LineStyle::RoundCap, 4, t == i ? ColorF(1, 0.9) : ColorF(1, 0.5));
                }
            }


            //コントロールパネル

            double y = 0;

            if (SimpleGUI::Button(U"sendEvent 1", Vec2{ 1000, (y += 40) }, 160))
            {
                client.sendEvent({ 1 }, String(U"こんにちは。"));
            }

            if (SimpleGUI::Button(U"sendEvent 2", Vec2{ 1000, (y += 40) }, 160))
            {
                client.sendEvent({ 2 }, Cursor::PosF(), 12, 3.4);
            }

            if (SimpleGUI::Button(U"sendEvent 3", Vec2{ 1000, (y += 40) }, 160))
            {
                client.sendEvent({ 3, ReceiverOption::All }, 3.14);
            }

            if (SimpleGUI::Button(U"sendEvent 4", Vec2{ 1000, (y += 40) }, 160))
            {
                client.sendEvent({ 4, Array<LocalPlayerID>{2, 3} }, Circle(10, 10, 10), RectF(10, 10, 20, 30));
            }

            if (SimpleGUI::Button(U"sendEvent 5", Vec2{ 1000, (y += 40) }, 160))
            {
                client.sendEvent({ 5 });
            }

            if (SimpleGUI::Button(U"ClearPrint()", Vec2{ 1000, (y += 40) }, 160))
            {
                ClearPrint();
            }

        }
    }

    ```