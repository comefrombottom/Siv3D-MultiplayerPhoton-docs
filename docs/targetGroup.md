# イベントターゲットグループ
`MultiplayerEvent`の第二引数には`EventReceiverOption`、`Array<LocalPlayerID>`のほかに`TargetGroup`を指定することが出来ます。`TargetGroup`はuint8型のラッパーであり、1以上255以下の整数を使用することが出来ます。
コンストラクタは`TargetGroup(uint8 targetGroup)`です。

例えば
```cpp
sendEvent(MultiplayerEvent(1,TargetGroup(1)),String(U"こんにちは！"));
```

とすることで、グループ１に所属する他のプレイヤーにイベントを送信することが出来ます。

なお、0を指定すると全てのプレイヤーが対象になります。プレイヤーはグループ0にはじめから全員所属していて、退出することはできません。

### グループへの加入・退出

`joinEventTargetGroup()`、`leaveEventTargetGroup()`を用いて自身のイベントターゲットグループを管理できます。`joinAllEventTargetGroups()`を用いると1~255全てのグループに所属でき、`leaveAllEventTargetGroups()`で全てのグループから退出します。