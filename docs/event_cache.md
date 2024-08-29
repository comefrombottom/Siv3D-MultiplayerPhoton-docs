# イベントキャッシュ
送信したイベントは通常その時参加しているプレイヤーのみ受信します。イベントキャッシュを使用すると、送信したあとに遅れて参加したプレイヤーも受信できるようになります。イベントキャッシュをするには、`MultiplayerEvent`の第二引数に、
- `EventReceiverOption::Others_CacheUntilLeaveRoom`
- `EventReceiverOption::Others_CacheForever`
- `EventReceiverOption::All_CacheUntilLeaveRoom`
- `EventReceiverOption::All_CacheForever`

のいずれかを指定します。

## キャッシュの種類
キャッシュには`○○○_CacheUntilLeaveRoom`と`○○○_CacheForever`の二種類があります。

- `○○○_CacheUntilLeaveRoom`はイベントキャッシュが送信したプレイヤーに紐づけられます。そのため、送信したプレイヤーが部屋から退出すると、このイベントキャッシュも消滅します。
- `○○○_CacheForever`はイベントキャッシュがルームに紐づけられます。そのため、送信したプレイヤーが退出しても残り続けます。

## イベントキャッシュの削除
`.removeEventCache(uint8 eventCode = 0)`を用いてキャッシュされたイベントを削除することが出来ます。第一引数ににイベントコードを指定することで、特定のイベントコードのイベントキャッシュを削除できます。イベントコードに0を入れるとイベントコードに関わらず削除します。

`.removeEventCache(uint8 eventCode, const Array<LocalPlayerID>& targets)`のオーバーロードではイベントキャッシュが紐づいているプレイヤーの`LocalPlayerID`を指定することが出来ます。`targets`に`{0}`を指定するとルームに紐づいたイベントキャッシュを指定します。（`○○○_CacheUntilLeaveRoom`を用いると送信したプレイヤーに紐づき、`○○○_CacheForever`を用いるとルームに紐づきます。）