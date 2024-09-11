# ランダムマッチング
`joinRandomRoom()`及び`joinRandomOrCreateRoom()`では基本的なランダムマッチングをすることが出来ます。これらの関数は引数に`propertyFilter`、`expectedMaxPlayers`、`matchmakingMode`を取ります。

## フィルター
`propertyFilter`と`expectedMaxPlayers`はランダムマッチング時のフィルターとして機能します。

各ルームのルームプロパティを参照し、`propertyFilter`で与えた`{key,value}`が全て一致するルームをランダムマッチングの対象とします。`propertyFilter`が公開されたルームプロパティを網羅している必要はありません。

`expectedMaxPlayers`はルームに設定された`maxPlayers`を参照し、一致するもののみを対象とします。0が設定された場合は無視されます。

## MatchmakingMode
`MatchmakingMode`は以下で与えられる列挙型で、デフォルトは`MatchmakingMode::FillOldestRoom`です。
```cpp
/// @brief ランダム入室時のマッチメイキングモード
enum class MatchmakingMode : uint8
{
	/// @brief 古いルームからうめていくように入室
	FillOldestRoom,

	/// @brief 順次均等に配分するように入室
	Serial,

	/// @brief ランダムに入室
	Random,
};
```