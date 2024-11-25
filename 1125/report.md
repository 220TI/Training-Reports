- [VLAN](#vlan)
  - [管理VLAN](#管理vlan)
  - [トランクポート](#トランクポート)
    - [トランキングプロトコル](#トランキングプロトコル)
      - [トランキングプロトコルの指定](#トランキングプロトコルの指定)
      - [モードの指定](#モードの指定)
    - [ネイティブVLAN](#ネイティブvlan)
    - [DTP](#dtp)
      - [モードの組み合わせ](#モードの組み合わせ)
  - [VTP（VLAN Trunking Protocol）](#vtpvlan-trunking-protocol)
    - [動作モード](#動作モード)
    - [VTPプルーニング](#vtpプルーニング)
    - [リビジョン番号](#リビジョン番号)
    - [DTPとの関係](#dtpとの関係)
    - [VTP使われていない説](#vtp使われていない説)
  - [VLAN間ルーティングとDHCP](#vlan間ルーティングとdhcp)
    - [L2SW1](#l2sw1)
    - [L3SW1](#l3sw1)


# VLAN

## 管理VLAN
デフォルトではVLAN1(デフォルトVLAN)がアサインされている。管理VLANにのみSSHやTelnetを許可する等の運用をすることで管理用のVLANとすることが可能。

一般トラフィック用のVLANと分離する等の処置が必要。

## トランクポート

### トランキングプロトコル
- IEEE 802.1Q:

送信元MACアドレスとタイプの間に4バイトのタグフィールドを挿入し、再度フレームチェックシーケンス（FCS）を計算する

- ISL:

Cisco独自の技術。イーサネットフレーム全体を新たなヘッダでカプセル化する方式を採用しており、通常のイーサネットフレームとは異なる形式でデータを転送する。

#### トランキングプロトコルの指定
~~~
Switch(config-if)#switchport trunk encapsulation { isl | dot1q }
~~~
トランキングプロトコルの指定であり、モードの指定ではないことに注意。

#### モードの指定
~~~
Switch(config-if)#switchport mode {access | trunk | dot1q-tunnel | private-vlan}
~~~

### ネイティブVLAN
IEEE 802.1Qでサポートされている特別なVLAN。ネイティブVLANに属するフレームにはタグを付けないため、スイッチは受信したフレームがネイティブVLANに属していると判断する。これにより、VLAN未対応のデバイスとの互換性が保たれる。

~~~
(config-if)#switchport trunk native vlan {vlan-id}
~~~

### DTP
Cisco独自のプロトコルで、スイッチ同士を接続した際にポートが自動的にトランクポートかアクセスポートとして動作するかを決定する

~~~
(config-if)#switchport {nonegotiate | mode dynamic {auto | desirable}}
~~~
対向機器がCisco以外の場合は`nonegotiate`とし、手動でトランクポートにする。

イーサネットのネゴシエーションとの混同に注意。下記のようなconfigが出題される。
~~~
L2SW1#sh run interface gi 0/0
Building configuration...

Current configuration : 102 bytes
!
interface GigabitEthernet0/0
 switchport mode access
 switchport nonegotiate
 negotiation auto //これはイーサネット
~~~


#### モードの組み合わせ
| 管理モード           | access         | dynamic auto    | dynamic desirable | trunk           |
|----------------------|----------------|-----------------|-------------------|-----------------|
| access               | アクセスポート   | アクセスポート    | アクセスポート     | ×（エラー）      |
| dynamic auto         | アクセスポート   | アクセスポート    | トランクポート     | トランクポート    |
| dynamic desirable    | アクセスポート   | トランクポート    | トランクポート     | トランクポート    |
| trunk                | ×（エラー）      | トランクポート    | トランクポート     | トランクポート    |

## VTP（VLAN Trunking Protocol）
Ciscoが開発したプロトコルで、ネットワーク内のスイッチ間でVLAN情報を自動的に伝播させるために使用される。VLAN設定を一元管理でき、手動で各スイッチに設定を適用する手間を省くことが可能となる。

デフォルトのドメイン名は空であり、これを既存のVTPドメインに接続するとVTPアドバタイズに含まれるVTPドメイン名を学習してVLANを同期する。

### 動作モード
| モード                      | 説明                                                           |
|----------------------------|----------------------------------------------------------------|
| **サーバーモード**         | VLANの作成、削除、変更が可能。VTPドメイン内の他のスイッチに情報を伝達 |
| **クライアントモード**     | VLAN情報の受信は可能だが、変更は不可                           |
| **トランスペアレントモード** | VLAN情報を転送せず、独自にVLANを管理                           |

トランスペアレントモード以外になっていると、新規にVLANを作ることが出来ない。VTPを使わない場合は特に注意。

### VTPプルーニング
特定のスイッチに所属していないVLANのトラフィックは、そのスイッチを通過する必要がないため、当該VLANフレームをブロードキャストしないようにすることができる。
~~~
Switch(config)# vtp pruning
~~~

### リビジョン番号
VLAN情報のバージョン管理を行う。各スイッチはVTPのリビジョン番号を使用して、どのVLAN設定が最新であるかを判断する。VLANの設定変更の度に1増加する。

### DTPとの関係
VTPドメインが異なると、VLAN情報の整合性を確保できないため、DTPのネゴシエーションが行えない。

~~~
L2SW1(config)#vtp domain abc　//ドメイン名abcに設定
Changing VTP domain name from NULL to abc
L2SW1(config)#
*Nov 25 05:16:33.210: %SW_VLAN-6-VTP_DOMAIN_NAME_CHG: VTP domain name changed to abc.
L2SW1(config)#int gi 0/0
L2SW1(config-if)#switchport mode dynamic desirable

L3SW1>
*Nov 25 05:17:17.529: %SW_VLAN-4-VTP_USER_NOTIFICATION: VTP protocol user notification: MD5 digest checksum mismatch on receipt of equal revision summary on trunk: Gi0/0
L3SW1(config)#vtp domain ABC //ドメイン名ABCに設定
Changing VTP domain name from abc to ABC
L3SW1(config)#
*Nov 25 05:17:50.115: %SW_VLAN-6-VTP_DOMAIN_NAME_CHG: VTP domain name changed to ABC.
L3SW1(config)#int gi 0/0
L3SW1(config-if)#switchport mode dynamic desirable
L3SW1(config-if)#
*Nov 25 05:18:12.812: %DTP-5-DOMAINMISMATCH: Unable to perform trunk negotiation on port Gi0/0 because of VTP domain mismatch.
// VTPドメイン不一致によるエラーをDTPが出力している。
~~~

### VTP使われていない説
Cisco試験においては出題されるが、現場においては使われていないとされている理由が多い。
- リビジョン数の高いスイッチを接続してしまった場合、意図せずVLANに変更が生じてしまう。(通称:VTP爆弾)
- マルチベンダーに対応していないので、柔軟性に欠ける。
- VXLANやSDNなど、より柔軟で抽象的な技術が普及している。



## VLAN間ルーティングとDHCP
![pic1](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1125/1125_1.png)

L3SW1によるVLAN間ルーティングを可能にするとともに、DHCPによりVLAN毎に適切なセグメントのIPアドレスをリースできるようにする。

### L2SW1
~~~
# VLAN10,20設定
L2SW1(config)#int vlan 10
L2SW1(config-if)#no shutdown

# それぞれインターフェースに割り当てる。
L2SW1(config)#int gi 0/1
L2SW1(config-if)#switchport access vlan 10

# トランクポート設定
L2SW1(config)#int gi 0/0
L2SW1(config-if)#switchport trunk encapsulation dot1q
L2SW1(config-if)#switchport mode trunk
L2SW1(config-if)#switchport trunk allowed vlan 1,10,20
~~~

### L3SW1
~~~
# VLAN10,20設定
L3SW1(config)#vlan 10
L3SW1(config-vlan)#no shutdown

# トランクポート設定
L3SW1(config)#int gi 0/0
L3SW1(config-if)#switchport trunk encapsulation dot1q
L3SW1(config-if)#switchport mode trunk
L3SW1(config-if)#switchport trunk allowed vlan 1,10,20

# 各VLANにIPアドレスを設定
L3SW1(config)#int vlan 10
*Nov 25 09:32:19.486: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to down
L3SW1(config-if)#ip address 192.168.1.254 255.255.255.0
L3SW1(config-if)#no shutdown
L3SW1(config-if)#
*Nov 25 09:32:49.946: %LINK-3-UPDOWN: Interface Vlan10, changed state to up
*Nov 25 09:32:50.946: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to up
L3SW1(config-if)#exit
L3SW1(config)#int vlan 20
*Nov 25 09:32:58.707: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan20, changed state to down
L3SW1(config-if)#ip address 192.168.2.254 255.255.255.0
L3SW1(config-if)#no shutdown
L3SW1(config-if)#
*Nov 25 09:33:32.138: %LINK-3-UPDOWN: Interface Vlan20, changed state to up
*Nov 25 09:33:33.138: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan20, changed state to up

# ルーティングテーブルの確認
L3SW1#sh ip route

Gateway of last resort is not set

      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, Vlan10
L        192.168.1.254/32 is directly connected, Vlan10
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, Vlan20
L        192.168.2.254/32 is directly connected, Vlan20

# DFGWのアドレスを配布しないようにする
L3SW1(config)#ip dhcp excluded-address 192.168.1.254
L3SW1(config)#ip dhcp excluded-address 192.168.2.254

# VLAN毎のDHCPプールを作成
L3SW1(config)#ip dhcp pool VLAN10
L3SW1(dhcp-config)#network 192.168.1.0 255.255.255.0
L3SW1(dhcp-config)#default-router 192.168.1.254
L3SW1(dhcp-config)#exit
L3SW1(config)#ip dhcp pool VLAN20
L3SW1(dhcp-config)#network 192.168.2.0 255.255.255.0
L3SW1(dhcp-config)#default-router 192.168.2.254
L3SW1(dhcp-config)#exit

#DHCP リース確認
Pool VLAN10 :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 1
 Excluded addresses             : 1
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased/Excluded/Total
 192.168.1.2          192.168.1.1      - 192.168.1.254     1     / 1     / 254

Pool VLAN20 :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 1
 Excluded addresses             : 1
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased/Excluded/Total
 192.168.2.2          192.168.2.1      - 192.168.2.254     1     / 1     / 254
~~~

PCアドレス確認
~~~
inserthostname-here:~$ ip route
default via 192.168.1.254 dev eth0  metric 202
192.168.1.0/24 dev eth0 scope link  src 192.168.1.1
~~~
PC1～PC2疎通確認
~~~
inserthostname-here:~$ traceroute 192.168.2.1
traceroute to 192.168.2.1 (192.168.2.1), 30 hops max, 46 byte packets
 1  192.168.1.254 (192.168.1.254)  3.399 ms  2.986 ms  2.817 ms
 2  192.168.2.1 (192.168.2.1)  4.235 ms  4.218 ms  3.503 ms
~~~