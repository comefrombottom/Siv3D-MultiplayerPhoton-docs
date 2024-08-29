# 部屋への再入室
接続が切れたあと、`.reconnectAndRejoin()`を呼ぶことで再び同じプレイヤーとして入り直すことが出来ます。`.reconnectAndRejoin()`はクライアントが`Disconnected`か`InLobby`の際に呼び出すことが出来ます。成功すると`.joinRoomReturn()`のコールバックが呼ばれ、部屋に入った際は、`.joinRoomEventAction()`が自分含めた部屋の各プレイヤーに呼ばれます。

デフォルトのルームは再入室できる設定になっていません。再入室が出来るようにするには、`RoomCreateOption`の`rejoinGracePeriod()`を設定する必要があります。`rejoinGracePeriod()`で再参加可能な猶予時間を設定することができます。例えば、`RoomCreateOption().rejoinGracePeriod(1min)`とすると、1分間は再入室が可能となります。`RoomCreateOption().rejoinGracePeriod(unspecified)`とすると部屋が存続する限り再入室可能となります。

再入室可能な部屋に通常の`.joinRoom()`で入室することはできません。また、`.leaveRoom()`を用いて明示的に退出した場合は、再入室できません。再入室可能な部屋は、`.disconnect()`で接続したり、予期せず切断されたり、`.leaveRoom(true)`（`willComeBack`を引数にとる）で退出したときです。

`.leaveRoom()`や`.leaveRoom(false)`によって退出した場合、つまりこれ以降そのプレイヤーが参加することがない場合、他のプレイヤーが退出したことを知らせるコールバックである`void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive);`は、引数の`isInactive`を`false`とします。それ以外で退出した場合でかつ部屋の`rejoinGracePeriod()`が0msより大きく設定されているとき、つまり今後また再入室する可能性がある場合`isInactive`を`true`とします。

`isInactive`が`true`で`leaveRoomEventAction()`が呼ばれたあと、そのプレイヤーが`rejoinGracePeriod()`以内に参加することなく期限が切れたとき、もう一度`isInactive`が`false`で`leaveRoomEventAction()`が呼ばれます。


再入室を考慮したサンプル
```cpp
# include <Siv3D.hpp> // Siv3D v0.6.15
# include "Multiplayer_Photon.hpp"
# include "PHOTON_APP_ID.SECRET"

struct Badge{
	Circle circle;
	ColorF color;

	Badge& moveBy(const Vec2& v) {
		circle.moveBy(v);
		return *this;
	}

	Badge movedBy(const Vec2& v) const {
		return Badge{ circle.movedBy(v), color };
	}

	template <class Archive>
	void SIV3D_SERIALIZE(Archive& archive)
	{
		archive(circle, color);
	}
};

//複数プレイヤーで共有するプレイヤーデータ
class SharePlayerData {
	Vec2 m_pos = {};
	ColorF m_color = Palette::White;

	Optional<Badge> m_pickedBadge;

public:
	SharePlayerData() = default;

	SharePlayerData(const Vec2& pos, const ColorF& color)
		: m_pos(pos)
		, m_color(color) {}

	const Vec2& pos() const { return m_pos; }
	const ColorF& color() const { return m_color; }
	const Optional<Badge>& pickedBadge() const { return m_pickedBadge; }

	void setPos(const Vec2& pos) { m_pos = pos; }
	void setColor(const ColorF& color) { m_color = color; }

	void setPickedBadge(const Badge& circle) { m_pickedBadge = circle; }
	void resetPickedBadge() { m_pickedBadge.reset(); }


	template <class Archive>
	void SIV3D_SERIALIZE(Archive& archive)
	{
		archive(m_pos, m_color, m_pickedBadge);
	}
};

//複数プレイヤーで共有するデータ
class ShareGameData
{
	HashTable<LocalPlayerID, SharePlayerData> m_players;

	Array<Badge> m_badges;

public:
	ShareGameData() {
		for (int i = 0; i < 10; ++i) {
			m_badges.push_back(Badge(Circle(RandomVec2(Scene::Rect()), Random(10, 50)), RandomColor()));
		}
	}

	auto& players() { return m_players; }

	SharePlayerData& player(LocalPlayerID playerID) { return m_players[playerID]; }

	const auto& badges() const { return m_badges; }

	Optional<size_t> findBadge(const Vec2& pos) const
	{
		for (auto [i, badge] : m_badges | std::views::enumerate | std::views::reverse)
		{
			if (badge.circle.intersects(pos))
			{
				return i;
			}
		}

		return none;
	}

	void playerPickCircle(LocalPlayerID playerID, const Vec2& pos)
	{
		if (auto i = findBadge(pos))
		{
			auto badge = m_badges[*i];
			m_badges.remove_at(*i);
			player(playerID).setPickedBadge(badge.moveBy(-pos));
		}
	}

	void playerDropCircle(LocalPlayerID playerID, const Vec2& pos)
	{
		if (auto badge = player(playerID).pickedBadge())
		{
			m_badges.push_back(badge->moveBy(pos));
			player(playerID).resetPickedBadge();
		}
	}

	template <class Archive>
	void SIV3D_SERIALIZE(Archive& archive)
	{
		archive(m_players, m_badges);
	}
};

struct PickBadgeLocalChange {
	size_t pickedBadgeIndex;
	Badge badge;
};


namespace EventCode {
	enum : uint8
	{
		//イベントコードは1から199までの範囲を使う
		sendShareGameData = 1,
		addPlayer,
		setPlayerPos,
		pickCircle_toHost,
		dropCircle_toHost,
		playerPickCircle,
		playerDropCircle,
		erasePlayer,
	};
}

class MyClient : public Multiplayer_Photon
{
public:
	MyClient()
		: Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::Yes)
	{
		RegisterEventCallback(EventCode::sendShareGameData, &MyClient::eventReceived_sendShareGameData);
		RegisterEventCallback(EventCode::addPlayer, &MyClient::eventReceived_addPlayer);
		RegisterEventCallback(EventCode::setPlayerPos, &MyClient::eventReceived_setPlayerPos);
		RegisterEventCallback(EventCode::pickCircle_toHost, &MyClient::eventReceived_pickCircle_toHost);
		RegisterEventCallback(EventCode::dropCircle_toHost, &MyClient::eventReceived_dropCircle_toHost);
		RegisterEventCallback(EventCode::playerPickCircle, &MyClient::eventReceived_playerPickCircle);
		RegisterEventCallback(EventCode::playerDropCircle, &MyClient::eventReceived_playerDropCircle);
		RegisterEventCallback(EventCode::erasePlayer, &MyClient::eventReceived_erasePlayer);
	}

	Optional<ShareGameData> shareGameData;

	Optional<PickBadgeLocalChange> pickBadgeLocalChange;

	Optional<Badge> dropBadgeLocalChange;


	//shareGameDataを変更し、他のプレイヤーも同様の変更を行うよう通知する。

	void setPlayerPos(const Vec2& pos) {
		if (not shareGameData) return;
		shareGameData->player(getLocalPlayerID()).setPos(pos);
		sendEvent(MultiplayerEvent(EventCode::setPlayerPos), pos);
	}

	void pickCircle(const Vec2& pos) {
		if (not shareGameData) return;
		if (isHost()) {
			shareGameData->playerPickCircle(getLocalPlayerID(), pos);
			sendEvent(MultiplayerEvent(EventCode::playerPickCircle), getLocalPlayerID(), pos);
		}
		else {
			if (auto i = shareGameData->findBadge(pos))
			{
				pickBadgeLocalChange = PickBadgeLocalChange{ *i, shareGameData->badges()[*i].movedBy(-pos) };
			}

			//ホストに通知
			sendEvent(MultiplayerEvent(EventCode::pickCircle_toHost, EventReceiverOption::Host), pos);
		}
	}

	void dropCircle(const Vec2& pos) {
		if (not shareGameData) return;
		if (isHost()) {
			shareGameData->playerDropCircle(getLocalPlayerID(), pos);
			sendEvent(MultiplayerEvent(EventCode::playerDropCircle), getLocalPlayerID(), pos);
		}
		else {
			if (auto badge = shareGameData->player(getLocalPlayerID()).pickedBadge()) {
				dropBadgeLocalChange = badge->moveBy(pos);
			}

			//ホストに通知
			sendEvent(MultiplayerEvent(EventCode::dropCircle_toHost, EventReceiverOption::Host), pos);
		}
	}

private:

	//イベントを受信したらそれに応じた処理を行う

	void eventReceived_sendShareGameData(LocalPlayerID playerID, const ShareGameData& data)
	{
		shareGameData = data;
	}

	void eventReceived_addPlayer(LocalPlayerID playerID, LocalPlayerID newPlayerID, const SharePlayerData& data)
	{
		if (not shareGameData) return;
		shareGameData->players().emplace(newPlayerID, data);
	}

	void eventReceived_setPlayerPos(LocalPlayerID playerID, const Vec2& pos)
	{
		if (not shareGameData) return;
		shareGameData->player(playerID).setPos(pos);
	}

	void eventReceived_pickCircle_toHost(LocalPlayerID playerID, const Vec2& pos)
	{
		if (not shareGameData) return;
		if (not isHost()) return;
		shareGameData->playerPickCircle(playerID, pos);
		sendEvent(MultiplayerEvent(EventCode::playerPickCircle), playerID, pos);
	}

	void eventReceived_dropCircle_toHost(LocalPlayerID playerID, const Vec2& pos)
	{
		if (not shareGameData) return;
		if (not isHost()) return;
		shareGameData->playerDropCircle(playerID, pos);
		sendEvent(MultiplayerEvent(EventCode::playerDropCircle), playerID, pos);
	}

	void eventReceived_playerPickCircle(LocalPlayerID hostID, LocalPlayerID pickerID, const Vec2& pos)
	{
		if (not shareGameData) return;
		shareGameData->playerPickCircle(pickerID, pos);

		if (pickerID == getLocalPlayerID()) {
			pickBadgeLocalChange.reset();
		}
	}

	void eventReceived_playerDropCircle(LocalPlayerID hostID, LocalPlayerID dropperID, const Vec2& pos)
	{
		if (not shareGameData) return;
		shareGameData->playerDropCircle(dropperID, pos);

		if (dropperID == getLocalPlayerID()) {
			dropBadgeLocalChange.reset();
		}
	}

	void eventReceived_erasePlayer(LocalPlayerID playerID, LocalPlayerID erasePlayerID)
	{
		if (not shareGameData) return;
		shareGameData->players().erase(erasePlayerID);
	}

	void joinRoomEventAction(const LocalPlayer& newPlayer, const Array<LocalPlayerID>& playerIDs, bool isSelf) override
	{

		const bool rejoin = shareGameData ? shareGameData->players().contains(newPlayer.localID) : false;

		//自分が入室した時
		if (isSelf) {
			pickBadgeLocalChange.reset();
			dropBadgeLocalChange.reset();
		}

		//ホストが入室した時かつ再入室でない時、つまり部屋を新規作成した時
		if (isSelf and isHost() and not rejoin) {
			shareGameData = ShareGameData();
			shareGameData->players().emplace(newPlayer.localID, SharePlayerData(Scene::Center(), RandomColorF()));
		}

		//誰かが部屋に入って来た時、ホストはその人にデータを送る
		if (not isSelf and isHost()) {
			sendEvent(MultiplayerEvent(EventCode::sendShareGameData, { newPlayer.localID }), *shareGameData);

			if (not rejoin) {

				//新しいプレイヤーデータを作成し、他プレイヤーに送信
				SharePlayerData newPlayerData(Scene::Center(), RandomColorF());
				shareGameData->players().emplace(newPlayer.localID, newPlayerData);
				sendEvent(MultiplayerEvent(EventCode::addPlayer), newPlayer.localID, newPlayerData);
			}
		}
	}

	void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive) override
	{
		if (not shareGameData) return;

		if (isHost() and not isInactive) {
			
			if (auto& badge = shareGameData->player(playerID).pickedBadge()) {
				const Vec2& pos = shareGameData->player(playerID).pos();
				shareGameData->playerDropCircle(playerID, pos);
				sendEvent(MultiplayerEvent(EventCode::playerDropCircle), playerID, pos);
			}

			shareGameData->players().erase(playerID);
			sendEvent(MultiplayerEvent(EventCode::erasePlayer), playerID);
		}
	}
};

void Main()
{
	MyClient client;
	
	while (System::Update())
	{
		if (client.isActive())
		{
			client.update();
		}
		else {

			if (client.reconnectAndRejoin()) {
				Print << U"reconnectAndRejoin";
			}
			else {
				Print << U"connect";
				client.connect(U"player", U"jp");
			}
		}

		if (client.isInLobby())
		{
			Scene::Rect().draw(Palette::Steelblue);

			if (SimpleGUI::ButtonAt(U"joinRandomOrCreateRoom", Scene::Center(), 300))
			{
				//適当な部屋に入るか、部屋がなければ新規作成する。部屋がある限り再入室可能にし、部屋が5秒間空いたら部屋を削除する。
				client.joinRandomOrCreateRoom(U"", RoomCreateOption().rejoinGracePeriod(unspecified).roomDestroyGracePeriod(5s));
			}
		}

		if (client.isInRoom())
		{
			Scene::Rect().draw(Palette::Sienna);

			if (client.shareGameData) {

				{
					Vec2 prePos = client.shareGameData->player(client.getLocalPlayerID()).pos();
					Vec2 nextPos = Cursor::Pos();
					if (prePos != nextPos) {
						client.setPlayerPos(nextPos);
					}
				}

				if (MouseL.down()) {
					client.pickCircle(Cursor::Pos());
				}

				if (MouseL.up()) {
					client.dropCircle(Cursor::Pos());
				}

				for (auto [i,badge] : client.shareGameData->badges() | std::views::enumerate) {

					if (client.pickBadgeLocalChange and i == client.pickBadgeLocalChange->pickedBadgeIndex) {
						continue;
					}

					badge.circle.draw(badge.color);
				}

				if (auto badge = client.dropBadgeLocalChange) {
					badge->circle.draw(badge->color);
				}

				//プレイヤーのデータを表示
				for (const auto& [playerID, data] : client.shareGameData->players()) {
					if (auto& badge = data.pickedBadge();badge and not client.dropBadgeLocalChange) {
						Circle circle = badge->circle.movedBy(data.pos() - Vec2(2, 2));
						circle.drawShadow(Vec2(5, 5), 10, 2.0, ColorF(0.0, 0.2)).draw(badge->color);
					}

					if (playerID == client.getLocalPlayerID()) {
						if (client.pickBadgeLocalChange) {
							Circle circle = client.pickBadgeLocalChange->badge.circle.movedBy(data.pos() - Vec2(2, 2));
							circle.drawShadow(Vec2(5, 5), 10, 2.0, ColorF(0.0, 0.2)).draw(client.pickBadgeLocalChange->badge.color);

						}
					}


					Circle(data.pos(), 3).draw(data.color());
				}
			}

			if (SimpleGUI::Button(U"LeaveRoom", Vec2{ 20, 20 }, 160))
			{
				client.leaveRoom();
			}
		}

		if (not client.isInLobby() and not client.isInRoom()) {
			//ローディング画面
			size_t t = Floor(fmod(Scene::Time() / 0.1, 8));
			for (size_t i : step(8)) {
				Vec2 n = Circular(1, i * Math::TwoPi / 8);
				Line(Scene::Center() + n * 10, Arg::direction(n * 10)).draw(LineStyle::RoundCap, 4, t == i ? ColorF(1, 0.9) : ColorF(1, 0.5));
			}
		}

		if (client.isActive()) {
			if (SimpleGUI::Button(U"disconnect", Vec2{ 620, 20 }, 160))
			{
				client.disconnect();
			}
		}
	}
}
```