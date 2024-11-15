# 1115

## C891FJ操作

![NW環境](https://github.com/220TI/Training-Reports/tree/master/1115/HomeNW.png)

 - [ACL](#ACL)
 - [IPSLA](#IPSLA)

### ACL

#### ACLが適している要求
 - IPアドレスやポート番号など、具体的で変更が少ない条件に基づく制御。
 - 複雑な条件やコンテンツの内容に依存しない、シンプルなアクセス制限。
 - VLANに基づく制御

#### ACLが適していない要求
 - IPアドレスが頻繁に変わるサービスや、HTTPSのような暗号化されたトラフィックに対する細かな制御は困難。
 - 特定のアプリケーションやユーザーに基づく制御は、ACLだけでは困難。（ASAではドメインによるユーザ制御が可能。）
 - URLやキーワード、コンテンツの種類に基づく制御は、専用のフィルタリングソリューションが必要となる。

#### ACLはどこに適用したらいいのか？
 - 標準ACLは宛先に近い場所

 送信元の近くに適用すると、意図しないトラフィックが拒否されてしまう。

 - 拡張ACLは送信元に近い場所に適用

 宛先を含めた詳細な条件があるため、早めのフィルタリングが可能であり、不要なトラフィックが流れる範囲を局限出来る。


#### 標準ACL
送信元IPアドレスによる制御のみ可能。

#### 拡張ACL
宛先IPアドレス・プロトコル・確立されたTCPコネクション等のフィルタが可能。


#### 実演
PC1(192.168.1.4)以外からのVTY（仮想回線）によるSSH接続を拒否する
~~~
RT01(config)#ip access-list extended denyssh

RT01(config-ext-nacl)#permit tcp host 192.168.1.4 host 192.168.1.1 eq 22

RT01(config-ext-nacl)#deny tcp any host 192.168.1.1 eq 22

RT01(config-ext-nacl)#permit ip any any

RT01(config-ext-nacl)#exit

RT01(config)#line vty 0 4

RT01(config-line)#access-class denyssh in
~~~

##### SPANを用いてミラーリングし、Wiresharkで確認する

SPANを用いると、スイッチポートのフレームを他ポートにミラーリングすることができる。

GigabitEthernet1～PC2間のフレームをGigabitEthernet0～PC1間にミラーリングし、これをPC1のWiresharkで確認する。

~~~
RT01(config)#monitor session 1 source interface gi 1 both

RT01(config)#monitor session 1 destination interface gi 0
~~~

![SSHのバージョンネゴシエーションも行われずにコネクションが切断される](https://github.com/220TI/Training-Reports/tree/master/1115/wireshark.png)

その他WEB閲覧等は問題なく行える。

### IPSLA

#### 概要・調査

 - 収集されたパフォーマンスデータは、SNMPのMIBに格納される。これにより、SNMPを介してデータをリモートで取得・監視することが可能。

 - IP SLAにおける各種設定は「オペレーション」と呼ばれる。オペレーションは、特定のネットワークパフォーマンスを測定するための個別のタスクやテストを指す。

 - IP SLAは、ネットワーク上でパケットを生成し、その応答を測定することでパフォーマンスを監視する。これにより、ネットワークの状態をリアルタイムで把握できる。

 - threshold（閾値）やtimeoutを設定することで、指定した条件を超過した場合にバックアップパスへ自動的に切り替えることができる。この機能は、IP SLAとObject Trackingを組み合わせることで実現される。

 - IP BASEライセンスでは、Object Tracking機能が制限されており、フル機能の実装が不可。バックアップパスへの自動切り替えなどの高度な機能を利用するためには、上位のライセンス（例：SEC、DATAライセンス）が必要となる。

 - IP SLAを使用することで、以下の指標を監視できる。

        - 遅延

        - ジッター

        - パケット損失

        - 接続性

        - UDP（例：VoIPなど）を監視する場合、IP SLAに対応したIOSデバイスをレスポンダとして用意する必要がある。

 #### 実演

icmp-echoオペレーションをPC2に対し実行し、途中からPC2のNICを無効にしてみる。
 ~~~
 RT01(config)#ip sla 1

 RT01(config-ip-sla)#icmp-echo 192.168.1.3 source-ip 192.168.1.1

 RT01(config-ip-sla-echo)#frequency 10

 RT01(config-ip-sla-echo)#exit

 RT01(config)#ip sla schedule 1 start-time now

RT01(config)#do show ip sla statistics
IPSLAs Latest Operation Statistics

IPSLA operation id: 1
        Latest RTT: NoConnection/Busy/Timeout
Latest operation start time: 17:53:55 JST Fri Nov 15 2024
Latest operation return code: Timeout # PC2のNICを無効にしたことによりタイムアウトとなっている。
Number of successes: 39
Number of failures: 2    # 成功数と失敗数が記録されている。
Operation time to live: 3194 sec
 ~~~