# 基礎のプログラム
まっさらな状態からコードを描いていきます。
## クライアントの作成
クライアントとは、サーバーと通信を行うためのものです。マルチプレイヤー通信はすべてこのクライアントを介して行われます。
### インクルード
<Siv3D.hpp> に続いて、"Multiplayer_Photon.hpp" と "PHOTON_APP_ID.SECRET" をインクルードします。

### Multiplayer_Photon の継承
Multiplayer_Photon を継承したクラス MyClient（名前は任意）を作成し、using Multiplayer_Photon::Multiplayer_Photon; で Multiplayer_Photon のコンストラクタも継承します。

### Photon App ID の格納
const std::string secretAppID{ SIV3D_OBFUSCATE(PHOTON_APP_ID) }; とすることで、実行時に Photon App ID が secretAppID に格納されます。直接 const std::string secretAppID{ PHOTON_APP_ID }; とすると、ビルドした実行ファイルのバイナリを解析した際に、Photon App ID がそのまま表れてしまいますが、SIV3D_OBFUSCATE() で包むことで、多少の難読化が施されます。

### MyClient の作成
MyClient オブジェクトを作成します。コンストラクタには Photon App ID, アプリケーションのバージョン、詳細なデバッグ表示の有無、の 3 つのパラメータを渡します。
Photon App ID が同じでも、アプリケーションのバージョンが異なるプログラムとは通信ができません。ゲームのバージョンアップ後に、新旧のバージョン同士で通信してしまうことを防ぐことができます。
詳細なデバッグ表示を有効 (Verbose::Yes) にすると、Multiplayer_Photon クラスの protected メンバ変数 m_verbose が true になり、Multiplayer_Photon の各種コールバック関数が呼ばれた際に、詳細な情報を Print 経由で出力するようになります。開発中は有効にしておくとデバッグに便利です。リリース時には Verbose::No を選択すると、一切 Print しなくなります。
```cpp
# include <Siv3D.hpp> // Siv3D v0.6.15
# include "Multiplayer_Photon.hpp"
# include "PHOTON_APP_ID.SECRET"

// Multiplayer_Photon を継承したクラス
class MyClient : public Multiplayer_Photon
{
public:

	// Multiplayer_Photon のコンストラクタを継承
	using Multiplayer_Photon::Multiplayer_Photon;
};

void Main()
{
	// ウィンドウを　1280x720 にリサイズ
	Window::Resize(1280, 720);

	// Photon App ID
	// 実行ファイルに App ID が直接埋め込まれないよう、SIV3D_OBFUSCATE() でラップ
	const std::string secretAppID{ SIV3D_OBFUSCATE(PHOTON_APP_ID) };

	// サーバーと通信するためのクラス
	// - Photon App ID
	// - 今回作ったアプリケーションのバージョン（これが異なるプログラムとは通信できない）
	// - Print によるデバッグ出力の有無
	MyClient client{ secretAppID, U"1.0", Verbose::Yes };

	while (System::Update())
	{
        
	}
}
```

チャットルームのサンプルコードのように、自作クラスのコンストラクタで各引き数を渡してもよい。
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
};

void Main()
{
	Window::Resize(1280, 720);

	MyClient client;

	while (System::Update())
	{
        
	}
}
```

## 接続・維持・切断

### サーバーに接続する
MyClient の `.connect()` でサーバーに接続します。引数として自身の名前（ユーザ名）を設定します。このあと、ユーザ名にランダムな数字をつなげた文字列がユーザ ID として自動的に割り当てられます。第二引数に`U"jp"`などのリージョンを渡すことでサーバーを指定できます。ない場合は利用可能なもののうち最も速いものが選ばれます。

### サーバーと同期する
`.connect()` すると `.isActive()` が `true` を返すようになります。この間は 60FPS の頻度で `.update()` を呼び、サーバと同期をとり続ける必要があります。.update() が数秒以上呼ばれないと、サーバから切断される場合があります。サーバと接続していない時に `.update()` を呼んでも何も起こりません。
この先のチュートリアルで登場する「～すると呼ばれる関数」は、基本的に `.update()` のタイミングで呼ばれます。

### サーバから切断する
MyNetwork のデストラクタで自動的にサーバから切断するため、明示的な `.disconnect()` は不要です。

```cpp
# include <Siv3D.hpp> // OpenSiv3D v0.6.15
# include "Multiplayer_Photon.hpp"
# include "PHOTON_APP_ID.SECRET"

class MyClient : public Multiplayer_Photon
{
public:
	MyClient()
		:Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)),U"1.0",Verbose::Yes)
	{}
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
		if (SimpleGUI::Button(U"Connect", Vec2{ 1000, 20 }, 160, (not client.isActive())))
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

の3つの状態を主にとり、加えてそれらの移行状態を合わせて7つの状態を取りえます。
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

ロビーには、複数の部屋を作成することができ、またその中から部屋を選んで入室することが出来ます。マルチプレイヤー通信は同部屋の中にいるプレイヤー間でのみ行われ、ロビーで通信することはできません。