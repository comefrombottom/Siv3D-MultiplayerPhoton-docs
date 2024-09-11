# 基礎のプログラム
## クライアントの作成
クライアントとは、サーバーと通信を行うためのものです。マルチプレイヤー通信はすべてこのクライアントを介して行われます。

### 独自のクライアントクラスを作成
`<Siv3D.hpp>` に続いて、"Multiplayer_Photon.hpp" と "PHOTON_APP_ID.SECRET" をインクルードし、`Multiplayer_Photon` を継承したクラス `MyClient`（名前は任意）を作成します。コンストラクタで`init()`関数（あるいは`Multiplayer_Photon`の引数付きコンストラクタ）を呼び出し中身を作成します。

### Photon App ID の格納
`std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID))` を`init()`に渡しましょう。直接 `std::string(PHOTON_APP_ID)` とすると、ビルドした実行ファイルのバイナリを解析した際に、Photon App ID がそのまま表れてしまいますが、`SIV3D_OBFUSCATE()` で包むことで、多少の難読化が施されます。

### MyClient の構築
`init()`には Photon App ID, アプリケーションのバージョン、詳細なデバッグ表示の有無、の 3 つのパラメータを渡します。
Photon App ID が同じでも、アプリケーションのバージョンが異なるプログラムとは通信ができません。ゲームのバージョンアップ後に、新旧のバージョン同士で通信してしまうことを防ぐことができます。
詳細なデバッグ表示を有効 (`Verbose::Yes`) にすると、`Multiplayer_Photon` クラスの protected メンバ変数 `m_verbose` が `true` になり、`Multiplayer_Photon` の各種コールバック関数が呼ばれた際に、詳細な情報を Print 経由で出力するようになります。開発中は有効にしておくとデバッグに便利です。リリース時には Verbose::No を選択すると、一切 Print しなくなります。

```cpp
# include <Siv3D.hpp> // Siv3D v0.6.15
# include "Multiplayer_Photon.hpp"
# include "PHOTON_APP_ID.SECRET"

class MyClient : public Multiplayer_Photon
{
public:
	MyClient() {
		//init()を呼び出し、クライアントの中身を構築する。
		// - Photon App ID 
		//   実行ファイルに App ID が直接埋め込まれないよう、SIV3D_OBFUSCATE() でラップ
		// - 今回作ったアプリケーションのバージョン（これが異なるプログラムとは通信できない）
		// - Print によるデバッグ出力の有無
		init(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::Yes);
	}
};

void Main()
{
	// ウィンドウを　1280x720 にリサイズ
	Window::Resize(1280, 720);

	// サーバーと通信するためのクライアントを作成
	MyClient client;

	while (System::Update())
	{
        
	}
}
```

### デバッグ出力先を変更
`Multiplayer_Photon`のコンストラクタの第3引数に`Verbose`の代わりに`Console`などの文字列を受取る関数、関数オブジェクトを与えると、デバッグ出力の出力先を変更することができます。(デフォルトでは`Print`)　この場合、第4引数で`Verbose`を指定できます。

各自でにデバッグ出力をする場合は、`.debugLog()`を用いましょう。指定した出力先に合わせて出力でき、`Verbose::No`の時出力しなくなります。

```cpp
class MyClient : public Multiplayer_Photon
{
public:
	//デバッグ情報をConsole出力
	MyClient()
	{
		init(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Console);

		//デフォルトではPrint出力するが、Consoleを指定したのでConsole出力
		//Verbose::Noの時は何もしない。
		debugLog(U"MyClient created");
	}
};
```


## 接続・維持・切断

### サーバーに接続する
`MyClient` の `.connect()` でサーバーに接続します。引数として自身の名前（ユーザ名）を設定します。このあと、ユーザ名にランダムな数字をつなげた文字列がユーザ ID として自動的に割り当てられます。第二引数に`U"jp"`などのリージョンを渡すことでサーバーを指定できます。ない場合は利用可能なもののうち最も速いものが選ばれます。

### サーバーと同期する
`.connect()` すると `.isActive()` が `true` を返すようになります。この間は 60FPS の頻度で `.update()` を呼び、サーバと同期をとり続ける必要があります。`.update()` が数秒以上呼ばれないと、サーバから切断される場合があります。サーバと接続していない時に `.update()` を呼んでも何も起こりません。
この先のチュートリアルで登場する「コールバック関数」は、基本的に `.update()` のタイミングで呼ばれます。

### サーバから切断する
`MyClient` のデストラクタで自動的にサーバから切断するため、明示的な `.disconnect()` は不要です。

```cpp
# include <Siv3D.hpp> // OpenSiv3D v0.6.15
# include "Multiplayer_Photon.hpp"
# include "PHOTON_APP_ID.SECRET"

class MyClient : public Multiplayer_Photon
{
public:
	MyClient()
	{
		init(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::Yes);
	}
};

void Main()
{
	Window::Resize(1280, 720);
	MyClient client;

	while (System::Update())
	{
		// サーバーと接続している場合、サーバーと同期する
		client.update();

		// サーバーに接続するボタン
		if (SimpleGUI::Button(U"Connect", Vec2{ 1000, 20 }, 160, client.isDisconnected()))
		{
			// ユーザ名
			const String userName = U"siv3d-kun";

			// サーバーに接続する
			client.connect(userName);
		}
	}

	// サーバーから切断する
	// Multiplayer_Photon のデストラクタで自動的に切断されるため、明示的に呼ぶ必要はない
	// network.disconnect();
}
```
## クライアントの状態
クライアントは、

- サーバーに接続されていない状態 `Disconnected`
- サーバーに接続され、ロビーにいる状態 `InLobby`
- 部屋にいる状態 `InRoom`

の3つの状態を主にとり、それらの移行状態を合わせて7つの状態を取りえます。
```cpp
enum class ClientState {
	Disconnected,
	ConnectingToLobby,
	InLobby,
	JoiningRoom,
	InRoom,
	LeavingRoom,
	Disconnecting,
};
```
クライアントの状態は`.getClientState()`で取得できます。

`ClientState`はフォーマットに対応しているため、そのままPrint出力などが可能です。
```cpp
Print << client.getClientState();
```

## ルームの作成・入室
ロビーには、複数の部屋を作成することができ、またその中から部屋を選んで入室することが出来ます。マルチプレイヤー通信は同部屋の中にいるプレイヤー間でのみ行われ、ロビーで通信することはできません。

- `.createRoom(RoomNameView roomName)` : ロビー内に部屋を作成し、入室する。
- `.joinRoom(RoomNameView roomName)` : ロビー内にすでにある部屋に対して入室する。
- `.leaveRoom()` : 今いる部屋から退出する。

※`RoomName`は`String`,`RoomNameView`は`StringView`のエイリアス

`roomName`はRoomIDとして機能し、同じ名前のルームを複数作ることはできません。

### ルーム作成オプション

ルーム作成オプションを表す型として`RoomCreateOption`が用意されています。

- `isVisible()` : ロビーのルーム一覧に表示されるか。デフォルトで`true`
- `isOpen()` : 他の人が入室できるか。デフォルトで`true`
- `maxPlayers()` : ルームに入れる最大人数。0 を指定すると無制限になる。デフォルトで 0
- `roomDestroyGracePeriod()` : ルームに誰もいなくなってからルームが破棄されるまでの猶予時間。最大5分(300000ms)。デフォルトで0ms

などのオプション指定するときに使用します。`.createRoom()`の第二引数に指定して使います。
```cpp
//ロビーから見えず、最大2人までの部屋を作成
client.createRoom(U"room#" + ToHex(RandomUint32()), RoomCreateOption().isVisible(false).maxPlayers(2));
```

### ロビーから部屋の一覧を取得
- `.getRoomNameList()` : `Array<RoomName>`でルームの名前一覧を取得する。

- `.getRoomList()` : `Array<RoomInfo>`でルームの情報一覧を取得する。`RoomInfo`型はロビーから所得可能なルームの情報をメンバ変数に持っています。名前以上の情報が欲しい場合はこちらを使用します。(`maxPlayers` : 最大人数, `playerCount` : 現在の人数, ...など)

### その他
- `.joinOrCreateRoom()` : 指定した名前のルームに参加を試み、無かった場合にルームの作成を試みます。

- `.joinRandomRoom()` : 既存のルームの中からランダムに参加を試みます。

- `joinRandomOrCreateRoom()` : ランダムなルームに参加を試み、参加できるルームが無かった場合にルームの作成を試みます。

