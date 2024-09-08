# ルームプロパティ
各ルームには、`HasTable<uint8,String>`型のプロパティを持たせることができます。

### ルームプロパティの主な操作
- `String getRoomProperty(uint8 key)`
- `HashTable<uint8,String> getRoomProperties()`
- `void setRoomProperty(uint8 key, StringView value)`

ルームプロパティを削除することはできません。代わりに`setRoomProperty()`で空文字列を設定することで対応してください。

### 初期化
`RoomCreateOption`の`properties()`を用いてルームプロパティの初期値を設定することが出来ます。

### ロビーからルームプロパティの取得
ロビーにいるプレイヤーは `Array<RoomInfo> getRoomList()` によって`RoomInfo`を取得し、`RoomInfo::properties`としてルームプロパティを閲覧することが出来ます。

### コールバック
- `void onRoomPropertiesChange(const HashTable<String,String>& changes)`


によってプロパティの変更を検知することが出来ます。引数には差分が送られます。