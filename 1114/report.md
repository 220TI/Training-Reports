# 1114

## 現場スキルキャッチアップ

### ASDM不具合調査
Ubuntu～ASAv間のパケットをキャプチャするも、TCPコネクションに問題はなく、TLSで通信が出来ていたということまでしか確認できなかった。

フォーラム等に倣い、ASAv⇔ExternalConnector⇔ホストオンリーアダプターの経路とし、ホストOSのブラウザでASAvにアクセスするとASDMのダウンロードボタンが表示された。

![lab](https://github.com/220TI/Training-Reports/tree/master/1114/lab.png)

![ホストオンリー](https://github.com/220TI/Training-Reports/tree/master/1114/hostonly.png)

![ASDMWEB](https://github.com/220TI/Training-Reports/tree/master/1114/ASDM_WEB.png)

![ホストオンリー](https://github.com/220TI/Training-Reports/tree/master/1114/ASDM.png)

### ASAvキャッチアップ

#### L2/L3ファイアウォール
- ルーテッドモード
- トランスペアレントモード

#### nameif
インターフェースに対して名前を付与することができる。
FWの各インターフェースは"inside""outside""dmz"といった役割を担うことが多いため、名前を付与することで直感的な管理が行えるようになる。

#### security-level
 - 0(低)～100(高)の値を設定できる
 - 高いセキュリティレベルのI/Fから低いセキュリティレベルのI/Fへの通信（発信）は、暗黙的に許可される。

 - 低いセキュリティレベルのI/Fから高いセキュリティレベルのI/Fへの戻りの通信（レスポンス）は、ステートフルインスペクションにより暗黙的に許可される。ただし、新規の通信を開始することはデフォルトでは許可されない。

 - same-security-traffic permit inter-interface を有効にした場合、同じセキュリティレベルのI/F間の通信（発信と戻り）が暗黙的に許可される。

 - 低いセキュリティレベルのI/Fから高いセキュリティレベルのI/Fへ新規の通信（発信）を行うためには、ACLで明示的に許可する必要がある。
