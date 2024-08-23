# イベントの送受信
同じルームにいるプレイヤーは互いにイベントとしてデータを送信することが出来ます。プレイヤー同士の通知、データの同期はこの機能を介して行われるため、マルチプレイヤーゲームの中核を担う機能です。

## イベントの送信
`sendEvent(const MultiplayerEvent& event, ...)`を用いてイベントを送信します。第一引数に`MultiplayerEvent`、第二引数以降に送信するデータを任意の個数連ねることが出来ます。

`MultiplayerEvent`はイベントコードとイベント送信のオプションを持った型です。コンストラクタの第一引数でイベントコード、第二引数でターゲットを指定できます。デフォルトでは自分以外です。

いくつか例を上げます。

- イベントコード 1 で、こんにちは。と送信
```cpp
client.sendEvent(MultiplayerEvent(1),　String(U"こんにちは。"));
```

- イベントコード 2 で、Vec2型, int型, double型のデータを送信
```cpp
client.sendEvent(MultiplayerEvent(2), Cursor::PosF(), 12, 3.4);
```

- イベントコード 3 で、自分含めた全てのプレイヤーに 3.14 を送信
```cpp
client.sendEvent(MultiplayerEvent(3, EventReceiverOption::All), 3.14);
```

- ターゲットの`LocalPlayerID`を指定して送信。この例では`LocalPlayerID`が２と３のプレイヤーが受信する。
```cpp
client.sendEvent(MultiplayerEvent(4, Array<LocalPlayerID>{2, 3}), Circle(10,10,10), RectF(10,10,20,30));
```

- イベントコードのみを送信してもよい。
```cpp
client.sendEvent(MultiplayerEvent(5));
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

をオーバーライドすることで受取ることが出来る。

例えば、先ほどの`sendEvent()`の例に全て対応するなら、

```cpp
class MyClient : public Multiplayer_Photon
{
public:
	using Multiplayer_Photon::Multiplayer_Photon;
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
のようになる。