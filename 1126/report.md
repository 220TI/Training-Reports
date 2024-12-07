- [STP](#stp)
  - [役割の選定](#役割の選定)
    - [ブリッジプライオリティ](#ブリッジプライオリティ)
      - [ブリッジプライオリティの指定](#ブリッジプライオリティの指定)
    - [ポートプライオリティ](#ポートプライオリティ)
  - [状態遷移](#状態遷移)
    - [収束・コンバージェンス](#収束コンバージェンス)
    - [STPタイマー](#stpタイマー)
  - [CST (Common Spanning Tree)](#cst-common-spanning-tree)
  - [PVST+ (Per VLAN Spanning Tree Plus)](#pvst-per-vlan-spanning-tree-plus)
  - [RSTP（Rapid Spanning Tree Protocol）](#rstprapid-spanning-tree-protocol)
  - [MST (Multiple Spanning Tree)](#mst-multiple-spanning-tree)
  - [portfast](#portfast)
    - [bpduguard](#bpduguard)
    - [BPDUフィルタリング](#bpduフィルタリング)
    - [BPDUフィルタリングとBPDUガードの違い](#bpduフィルタリングとbpduガードの違い)
  - [ルートガード](#ルートガード)
    - [動作](#動作)
    - [設定例](#設定例)
  - [UDLD](#udld)
    - [動作モード](#動作モード)
    - [設定例](#設定例-1)
  - [VLAN間ロードバランシング](#vlan間ロードバランシング)

# STP

| プロトコル | 規格        | Ciscoでの実装        | インスタンスの数      | インスタンスごとの負荷分散 |
|------------|-------------|----------------------|-----------------------|---------------------------|
| STP        | IEEE802.1d  | PVST+               | VLANごとに1つ         | 可                       |
| RSTP       | IEEE802.1w  | Rapid PVST+         | VLANごとに1つ         | 可                       |
| MST        | IEEE802.1s  | MST (Rapid PVST+で動作) | 複数のVLANで1つ       | 可                       |
| CST        | IEEE802.1q(トランク)  | 無し                | 全体で1つ             | 不可                     |


## 役割の選定

### ブリッジプライオリティ

ブリッジIDの比較: BPDUにはスイッチのブリッジIDが含まれ、以下の要素で構成される。

- ブリッジプライオリティ: デフォルト値は32768。

PVSTやMSTのブリッジプライオリティの16bit中下位12bitには拡張システムID(VLANIDやsysID)が割り当てられるため、10進数4096(2^12)の倍数での設定となる。（13bit～16bitでの設定となる。）

- MACアドレス

#### ブリッジプライオリティの指定
~~~
(config)#spanning-tree vlan {VLAN番号} {priority {プライオリティ値} | root {primary | secondary} [diameter {diameter値}]}
~~~
`root {primary | secondary}`については、意図しない結果となる可能性がある。(例:既にプライオリティ0のブリッジがある場合はroot primaryによる指定が不可。)

### ポートプライオリティ
- ポートプライオリティのデフォルト128

1. ルートポートの決定

ルートポートは、ルートブリッジまでの最短経路を提供するポートであり、スイッチごとに1つだけ存在する。

パスコストの計算: 各スイッチはルートブリッジまでの経路コストを計算する

コストはポートの速度に基づき、以下のように設定される（例）：

10 Mbps: 100

100 Mbps: 19

1 Gbps: 4

10 Gbps: 2


最小コストの選択: ルートブリッジまでのパスコストが最も低いポートをルートポートに設定。

同一コストの場合: 接続先スイッチのブリッジIDを比較し、値が小さい方を選択。

1. 指定ポートの決定

指定ポートは、リンク上でトラフィックを転送するポートとして選出される。

ルートパスコストの比較: 接続されたスイッチ間でパスコストを比較し、値が低いポートを指定ポートに設定。

同一コストの場合: ブリッジIDを比較し、値が小さいスイッチ側のポートを指定ポートに設定。

尚、ルートブリッジのポートは全て指定ポートとなる。

4. 非指定ポートの決定

ルートポートと指定ポート以外のポートは、非指定ポートとなり、トラフィックを転送しない。**BPDUの受信は行う。**

## 状態遷移
### 収束・コンバージェンス

| ポート状態           | 機能                                            | 収束時間                       |
|-----------------------|------------------------------------------------|--------------------------------|
| **ブロッキング**      | ループを防止。BPDUを受信するが、データフレームは転送しない。| 通常、20秒間この状態を維持。   |
| **リスニング**        | トポロジ変更の検出。MACアドレスを学習しない。 | 約15秒。                       |
| **ラーニング**        | MACアドレスを学習。データフレームは転送しない。| 約15秒。                       |
| **フォワーディング**  | データフレームを転送し、MACアドレスも学習。   | 即時（リスニング、ラーニング後に遷移）。 |
| **無効**             | 管理者が明示的にポートを無効化。               | なし（管理者操作による）。     |

### STPタイマー
| タイマー名          | 機能                                                                                      | デフォルト値 |
|----------------------|-------------------------------------------------------------------------------------------|--------------|
| **Hello Time**      | BPDUを2秒ごとに送信する間隔を設定。                                                        | 2秒          |
| **Max Age**         | BPDUを保持する時間。この時間を超えても新しいBPDUが届かない場合、障害発生とみなして再計算を開始。 | 20秒         |
| **Forward Delay**   | ListeningおよびLearning状態に留まる時間。BPDUの通知遅延を考慮して設定。                        | 15秒         |
## CST (Common Spanning Tree)
全てのVLANで単一のインスタンスを構成。

トランキングプロトコルであるIEE802.1qで定義されている。

## PVST+ (Per VLAN Spanning Tree Plus)
- VLANごとにインスタンスを構成し、柔軟な負荷分散が可能。

- VLAN数が多い場合、CPUや帯域負荷が増加。

- プライオリティ値は、設定値とVLANIDの合計となる。

## RSTP（Rapid Spanning Tree Protocol）

- 代替ポートとバックアップポートがある
- コンバージェンスが早い
- ポートの状態がSTPと異なる
## MST (Multiple Spanning Tree)
-  複数のVLANをグループ化し、インスタンスを削減。

- プライオリティ値は、設定値とインスタンスID(sysid)の合計となる。

## portfast
  エンドポイントを接続した際、通常のコンバージェンスを待たずに素早くフォワーディングになるようにする。

`(config)#spanning-tree portfast trunk`とすることでトランクポートでも使用可能。

### bpduguard
portfastが有効になっているポートでBPDUを受信したとき、err-disabled状態にする。

~~~
# グローバルコンフィギュレーションモードでの有効化
(config)# spanning-tree portfast bpduguard default

# ポートでの有効化
(config-if)# spanning-tree bpduguard enable
~~~


### BPDUフィルタリング
BPDUフィルタリングは、PortFastが設定されているポートでBPDUの送受信を無効にする。

- ポート単位で有効化: BPDUフィルタリングをポート単位で有効にした場合、そのポートではPortFastの設定の有無に関係なく、BPDUの送受信が完全に停止する。実質的には、そのポートでのSTPの無効化を意味する。

- グローバルで有効化: グローバルでBPDUフィルタリングを有効化した場合、PortFastが設定されたポートのみが対象となる。この場合、BPDUフィルタリングが有効なポートがBPDUを受信すると、そのPortFastの設定が解除され、BPDUフィルタリングが無効化されることで、通常のBPDU送受信が再開される。

### BPDUフィルタリングとBPDUガードの違い

- BPDUフィルタリングは、STPに参加しないデバイスへのBPDU送信を防止するこ。

BPDUガードは、STPトポロジに参加しようとする不正なデバイスや誤接続デバイスを検出し、ポートを無効化することでネットワークを保護する。

## ルートガード
特定のポートがスパニングツリーのルートブリッジとして選出されることを防ぐ。これにより、SWの接続等に伴い意図しないルートブリッジの変更を防ぐことが出来る。

### 動作
BPDU受信時の挙動:

1. 上位のルートブリッジとして認識されるBPDUを受信した場合、該当ポートを無効化 。

2. 無効化されたポートは、上位BPDUの受信が停止すると自動的に復旧。

### 設定例
~~~
Switch(config)# interface gigabitethernet 1/0/1
Switch(config-if)# spanning-tree guard root
~~~

## UDLD
（Unidirectional Link Detection：単方向リンク検出）

リンクが双方向通信可能であるかを確認するプロトコル。特にSTP を使用する環境で、トポロジーループを未然に防ぐために重要。

一方向通信とは

- 光ファイバーケーブルの片側断線:
光ファイバーケーブルは送信用(Tx)と受信(Rx)用の2本で構成されており、片側が断線すると、受信はできても送信ができない状態になる。

- スイッチポートの誤設定:
スイッチの設定ミスにより、一方的な通信しかできない場合がある。(例:片側ポートのみshutdown)

### 動作モード
- Normalモード:
リンクに問題が検出された場合、ログメッセージが記録されるが、ポートの動作はundetermined(不定)でありSTPの動作に依存する。

- Aggressiveモード(推奨):
リンクに問題が検出された場合、UDLDは再試行を試みる。それでも問題が解消されない場合、ポートの動作はerr-disabledとなる。

### 設定例

グローバルコンフィギュレーションモードでUDLDを有効化。光ファイバーケーブルでのみ有効化される。
~~~
Switch(config)# udld {enable | aggressive}
~~~

特定のインターフェースで有効化
~~~
Switch(config)# interface gigabitethernet 1/0/1
Switch(config-if)# udld port [aggressive]
~~~

## VLAN間ロードバランシング
各VLAN(或いはMSTインスタンス)でのブロッキングポートを変更することで負荷分散が可能。
