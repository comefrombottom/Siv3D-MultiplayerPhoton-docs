# イベント送受信によるデータの同期
プレイヤー間でデータを同期するために、それぞれが持つデータに同じ操作を行わせます。データを変更した際はそれを他のプレイヤーに通知し、通知を受けたプレイヤーはそれに応じた同様の操作を行います。

入室してきたプレイヤーには、現在の部屋の情報を教える必要があります。その役割をホストに担わせます。ホストは部屋に一人だけ割り当てられた存在で、通常は部屋を作った人に割り当てられます。ホストが部屋を抜けると別の人がホストとなり、必ずホストが一人いる状態が保たれます。ホストは部屋に人が入ってきたのを`joinRoomEventAction`で検知し、新しいプレイヤーに部屋の共有データ送信します。

??? summary "データの同期サンプル"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"

    //複数プレイヤーで共有するデータ
    class ShareGameData
    {
        int32 m_count = 0;
        Array<bool> m_switches;

    public:
        ShareGameData() : m_switches(5, false) {}

        int32 count() const { return m_count; }
        const Array<bool>& switches() const { return m_switches; }

        void setCount(int32 count) { m_count = count; }
        void setSwitch(size_t index, bool value) { m_switches[index] = value; }

        template <class Archive>
        void SIV3D_SERIALIZE(Archive& archive)
        {
            archive(m_count, m_switches);
        }
    };

    namespace EventCode {
        enum : uint8
        {
            //イベントコードは1から199までの範囲を使う
            sendShareGameData = 1,
            setValue,
            setSwitch,
        };
    }

    class MyClient : public Multiplayer_Photon
    {
    public:
        MyClient()
            : Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::No)
        {
            RegisterEventCallback(EventCode::sendShareGameData, &MyClient::eventReceived_sendShareGameData);
            RegisterEventCallback(EventCode::setValue, &MyClient::eventReceived_setCount);
            RegisterEventCallback(EventCode::setSwitch, &MyClient::eventReceived_setSwitch);
        }

        Optional<ShareGameData> shareGameData;

        //shareGameDataを変更し、他のプレイヤーも同様の変更を行うよう通知する。

        void setCount(int32 count) {
            if(not shareGameData) return;
            shareGameData->setCount(count);
            sendEvent(MultiplayerEvent(EventCode::setValue), count);
        }

        void setSwitch(size_t index, bool value) {
            if (not shareGameData) return;
            shareGameData->setSwitch(index, value);
            sendEvent(MultiplayerEvent(EventCode::setSwitch), index, value);
        }

    private:

        //イベントを受信したらそれに応じた処理を行う

        void eventReceived_sendShareGameData(LocalPlayerID playerID, const ShareGameData& data)
        {
            shareGameData = data;
        }

        void eventReceived_setCount(LocalPlayerID playerID, int32 count)
        {
            if (not shareGameData) return;
            shareGameData->setCount(count);
        }

        void eventReceived_setSwitch(LocalPlayerID playerID, size_t index, bool value)
        {
            if (not shareGameData) return;
            shareGameData->setSwitch(index, value);
        }
        
        void joinRoomEventAction(const LocalPlayer& newPlayer, const Array<LocalPlayerID>& playerIDs, bool isSelf) override
        {
            //入室したら、共有データを初期化
            if (isSelf) {
                shareGameData = ShareGameData();
            }


            //誰かが部屋に入って来た時、ホストはその人にデータを送る
            if (not isSelf and isHost()) {
                sendEvent(MultiplayerEvent(EventCode::sendShareGameData, { newPlayer.localID }), *shareGameData);
            }
        }
    };

    void Main()
    {
        MyClient client;

        Font font(30);
        
        while (System::Update())
        {
            if (client.isActive())
            {
                client.update();
            }
            else {
                client.connect(U"player", U"jp");
            }

            if (client.isInLobby())
            {
                Scene::Rect().draw(Palette::Steelblue);

                if (SimpleGUI::ButtonAt(U"joinRandomOrCreateRoom", Scene::Center(), 300))
                {
                    client.joinRandomOrCreateRoom(U"room");
                }
            }

            if (client.isInRoom())
            {
                Scene::Rect().draw(Palette::Sienna);

                if (SimpleGUI::Button(U"LeaveRoom", Vec2{ 20, 20 }, 160))
                {
                    client.leaveRoom();
                }

                if (client.shareGameData) {
                    int32 count = client.shareGameData->count();

                    //カウントの値を表示
                    RectF rect = font(count).drawAt(Scene::CenterF().moveBy(0,-50));
                    if (SimpleGUI::ButtonAt(U"+", rect.rightCenter().movedBy(50,0)))
                    {
                        //ボタンを押すと値を増やす
                        client.setCount(count + 1);
                    }

                    if (SimpleGUI::ButtonAt(U"-", rect.leftCenter().movedBy(-50, 0)))
                    {
                        //ボタンを押すと値を減らす
                        client.setCount(count - 1);
                    }

                    //スイッチの状態を表示
                    const size_t switchCount = client.shareGameData->switches().size();
                    for (auto [i, value] : Indexed(client.shareGameData->switches())) {
                        const Vec2 pos = Scene::Center().movedBy((i - (switchCount - 1) / 2.0) * 50, 100);
                        Circle circle(pos, 20);
                        if (circle.leftClicked()) {
                            //クリックするとスイッチの状態を反転
                            client.setSwitch(i, not value);
                        }
                        circle.draw(value ? Palette::Lime : Palette::Gray);
                    }
                }
            }
        }
    }

    ```



# プレイヤーデータの同期
`ShareGameData`内に`LocalPlayerID`とプレイヤーデータの`HashTable`を作成します。`joinRoomEventAction()`、`leaveRoomEventAction()`によって`HashTable`を適切に管理します。

??? summary "プレイヤーデータの同期サンプル"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"

    //複数プレイヤーで共有するプレイヤーデータ
    class SharePlayerData {
        Vec2 m_pos = {};
        ColorF m_color = Palette::White;

    public:
        SharePlayerData() = default;

        SharePlayerData(const Vec2& pos, const ColorF& color)
            : m_pos(pos)
            , m_color(color) {}

        const Vec2& pos() const { return m_pos; }
        const ColorF& color() const { return m_color; }

        void setPos(const Vec2& pos) { m_pos = pos; }
        void setColor(const ColorF& color) { m_color = color; }

        template <class Archive>
        void SIV3D_SERIALIZE(Archive& archive)
        {
            archive(m_pos, m_color);
        }
    };

    //複数プレイヤーで共有するデータ
    class ShareGameData
    {
        HashTable<LocalPlayerID, SharePlayerData> m_players;

        int32 m_count = 0;
        Array<bool> m_switches;

    public:
        ShareGameData() : m_switches(5, false) {}

        HashTable<LocalPlayerID, SharePlayerData>& players() { return m_players; }

        SharePlayerData& player(LocalPlayerID playerID) { return m_players[playerID]; }

        int32 count() const { return m_count; }
        const Array<bool>& switches() const { return m_switches; }

        void setCount(int32 count) { m_count = count; }
        void setSwitch(size_t index, bool value) { m_switches[index] = value; }

        template <class Archive>
        void SIV3D_SERIALIZE(Archive& archive)
        {
            archive(m_players, m_count, m_switches);
        }
    };

    namespace EventCode {
        enum : uint8
        {
            //イベントコードは1から199までの範囲を使う
            sendShareGameData = 1,
            setValue,
            setSwitch,
            addPlayer,
            setPlayerPos,
        };
    }

    class MyClient : public Multiplayer_Photon
    {
    public:
        MyClient()
            : Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::No)
        {
            RegisterEventCallback(EventCode::sendShareGameData, &MyClient::eventReceived_sendShareGameData);
            RegisterEventCallback(EventCode::setValue, &MyClient::eventReceived_setCount);
            RegisterEventCallback(EventCode::setSwitch, &MyClient::eventReceived_setSwitch);
            RegisterEventCallback(EventCode::addPlayer, &MyClient::eventReceived_addPlayer);
            RegisterEventCallback(EventCode::setPlayerPos, &MyClient::eventReceived_setPlayerPos);
        }

        Optional<ShareGameData> shareGameData;

        //shareGameDataを変更し、他のプレイヤーも同様の変更を行うよう通知する。

        void setCount(int32 count) {
            if(not shareGameData) return;
            shareGameData->setCount(count);
            sendEvent(MultiplayerEvent(EventCode::setValue), count);
        }

        void setSwitch(size_t index, bool value) {
            if (not shareGameData) return;
            shareGameData->setSwitch(index, value);
            sendEvent(MultiplayerEvent(EventCode::setSwitch), index, value);
        }

        void setPlayerPos(const Vec2& pos) {
            if (not shareGameData) return;
            shareGameData->player(getLocalPlayerID()).setPos(pos);
            sendEvent(MultiplayerEvent(EventCode::setPlayerPos), pos);
        }

    private:

        //イベントを受信したらそれに応じた処理を行う

        void eventReceived_sendShareGameData(LocalPlayerID playerID, const ShareGameData& data)
        {
            shareGameData = data;
        }

        void eventReceived_setCount(LocalPlayerID playerID, int32 count)
        {
            if (not shareGameData) return;
            shareGameData->setCount(count);
        }

        void eventReceived_setSwitch(LocalPlayerID playerID, size_t index, bool value)
        {
            if (not shareGameData) return;
            shareGameData->setSwitch(index, value);
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
        
        void joinRoomEventAction(const LocalPlayer& newPlayer, const Array<LocalPlayerID>& playerIDs, bool isSelf) override
        {
            //入室したら、共有データを初期化
            if (isSelf) {
                shareGameData = ShareGameData();
            }

            if (isSelf and isHost()) {
                //自分がホストの場合、自分のプレイヤーデータを作成
                shareGameData->players().emplace(newPlayer.localID, SharePlayerData(Scene::Center(), RandomColorF()));
            }


            //誰かが部屋に入って来た時、ホストはその人にデータを送る
            if (not isSelf and isHost()) {
                sendEvent(MultiplayerEvent(EventCode::sendShareGameData, { newPlayer.localID }), *shareGameData);

                //新しいプレイヤーデータを作成し、他プレイヤーに送信
                SharePlayerData newPlayerData(Scene::Center(), RandomColorF());
                shareGameData->players().emplace(newPlayer.localID, newPlayerData);
                sendEvent(MultiplayerEvent(EventCode::addPlayer), newPlayer.localID, newPlayerData);
            }
        }

        void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive) override
        {
            shareGameData->players().erase(playerID);
        }
    };

    void Main()
    {
        MyClient client;

        Font font(30);
        
        while (System::Update())
        {
            if (client.isActive())
            {
                client.update();
            }
            else {
                client.connect(U"player", U"jp");
            }

            if (client.isInLobby())
            {
                Scene::Rect().draw(Palette::Steelblue);

                if (SimpleGUI::ButtonAt(U"joinRandomOrCreateRoom", Scene::Center(), 300))
                {
                    client.joinRandomOrCreateRoom(U"room");
                }
            }

            if (client.isInRoom())
            {
                Scene::Rect().draw(Palette::Sienna);

                if (SimpleGUI::Button(U"LeaveRoom", Vec2{ 20, 20 }, 160))
                {
                    client.leaveRoom();
                }

                if (client.shareGameData) {
                    int32 count = client.shareGameData->count();

                    //カウントの値を表示
                    RectF rect = font(count).drawAt(Scene::CenterF().moveBy(0,-50));
                    if (SimpleGUI::ButtonAt(U"+", rect.rightCenter().movedBy(50,0)))
                    {
                        //ボタンを押すと値を増やす
                        client.setCount(count + 1);
                    }

                    if (SimpleGUI::ButtonAt(U"-", rect.leftCenter().movedBy(-50, 0)))
                    {
                        //ボタンを押すと値を減らす
                        client.setCount(count - 1);
                    }

                    //スイッチの状態を表示
                    const size_t switchCount = client.shareGameData->switches().size();
                    for (auto [i, value] : Indexed(client.shareGameData->switches())) {
                        const Vec2 pos = Scene::Center().movedBy((i - (switchCount - 1) / 2.0) * 50, 100);
                        Circle circle(pos, 20);
                        if (circle.leftClicked()) {
                            //クリックするとスイッチの状態を反転
                            client.setSwitch(i, not value);
                        }
                        circle.draw(value ? Palette::Lime : Palette::Gray);
                    }

                    {
                        Vec2 prePos = client.shareGameData->player(client.getLocalPlayerID()).pos();
                        Vec2 nextPos = Cursor::Pos();
                        if (prePos != nextPos) {
                            client.setPlayerPos(nextPos);
                        }
                    }

                    //他プレイヤーのデータを表示
                    for (const auto& [playerID, data] : client.shareGameData->players()) {
                        Circle(data.pos(), 20).draw(data.color());
                    }
                }
            }
        }
    }

    ```

# ホストを用いた衝撃解消
Hostを介してデータ変更の競合を解消します。Host以外のプレイヤーは遅延が発生するため、見かけ上早く動いているような工夫をします。

??? summary "ホストを用いた同期サンプル"
    ```cpp
    # include <Siv3D.hpp> // Siv3D v0.6.15
    # include "Multiplayer_Photon.hpp"
    # include "PHOTON_APP_ID.SECRET"

    struct Badge {
        Circle circle;
        ColorF color;

        Badge& moveBy(const Vec2& v) {
            circle.moveBy(v);
            return *this;
        }

        Badge movedBy(const Vec2& v) const {
            return Badge{ circle.movedBy(v), color };
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
            archive(m_pos, m_color);
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
            : Multiplayer_Photon(std::string(SIV3D_OBFUSCATE(PHOTON_APP_ID)), U"1.0", Verbose::No)
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
            if (isSelf) {
                shareGameData.reset();
            }

            if (isSelf and isHost()) {
                shareGameData = ShareGameData();

                //自分がホストの場合、自分のプレイヤーデータを作成
                shareGameData->players().emplace(newPlayer.localID, SharePlayerData(Scene::Center(), RandomColorF()));
            }


            //誰かが部屋に入って来た時、ホストはその人にデータを送る
            if (not isSelf and isHost()) {
                sendEvent(MultiplayerEvent(EventCode::sendShareGameData, { newPlayer.localID }), *shareGameData);

                //新しいプレイヤーデータを作成し、他プレイヤーに送信
                SharePlayerData newPlayerData(Scene::Center(), RandomColorF());
                shareGameData->players().emplace(newPlayer.localID, newPlayerData);
                sendEvent(MultiplayerEvent(EventCode::addPlayer), newPlayer.localID, newPlayerData);
            }
        }

        void leaveRoomEventAction(LocalPlayerID playerID, bool isInactive) override
        {
            if (not shareGameData) return;

            if (isHost()) {
                
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
                client.connect(U"player", U"jp");
            }

            if (client.isInLobby())
            {
                Scene::Rect().draw(Palette::Steelblue);

                if (SimpleGUI::ButtonAt(U"joinRandomOrCreateRoom", Scene::Center(), 300))
                {
                    client.joinRandomOrCreateRoom(U"");
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
        }
    }
    ```