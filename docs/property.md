# プレイヤープロパティ・ルームプロパティ
ルーム及びルームに参加中のプレイヤーには、各々`HasTable<String,String>`型のプロパティを持たせることができます。プレイヤープロパティは自身のもののみを編集することが出来ます。

### プレイヤープロパティの主な操作
- `String getPlayerProperty(LocalPlayerID localPlayerID, StringView key)`
- `HashTable<String, String> getPlayerProperties(LocalPlayerID localPlayerID)`
- `void setPlayerProperty(StringView key, StringView value)`
- `void removePlayerProperty(StringView key)`
### ルームプロパティの主な操作
- `String getRoomProperty(StringView key)`
- `HashTable<String,String> getRoomProperties()`
- `void setRoomProperty(StringView key, StringView value)`
- `void removeRoomProperty(StringView key)`

## 初期化
`RoomCreateOption`の`properties()`を用いてルームプロパティの初期値を設定することが出来ます。

## ルームプロパティの公開
ルームプロパティのうち指定したキーのプロパティをロビーから閲覧可能にすることが出来ます。
- `void setVisibleRoomPropertyKeys(const Array<String>& keys)`
- `Array<String> getVisibleRoomPropertyKeys();`

`RoomCreateOption`の`visibleRoomPropertyKeys()`で初期値を設定することが出来ます。

キーが登録されたプロパティは、`Array<RoomInfo> getRoomList()`によって`RoomInfo`を取得した際、`RoomInfo::properties`として取得できます。

## コールバック
- `void onRoomPropertiesChange(const HashTable<String,String>& changes)`
- `void onPlayerPropertiesChange(LocalPlayerID playerID, const HashTable<String, String>& changes)`

によってプロパティの変更を検知することが出来ます。引数には差分が送られます。