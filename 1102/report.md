# 11.02

## C891FJ検証作業

### 想定するNW構成

    AirTerminal5 -->|DHCP| C891FJ -->|DHCP| PC

### 目的
上記経路でのネットワーク疎通
    ⇒ ping(8.8.8.8)の疎通で評価する。2重NATになるのでゲーム等一部のオンラインコンテンツについては想定しない。

### 課題
1. 2重NATになる
2. AirTerminal5からリースされるIPアドレスのリース時間の確認
    ⇒ 固定割り当てが可能。
いずれも、異常があった際に逐次調査とする...

### 作業

### AirTerminal5 -> C891FJへのDHCP固定
C891FJへ192.168.3.254をリースすることとする。

#### WAN用ポート2つのアップ
~~~
RT01(config)#int gi 8
RT01(config-if)#no shutdown
RT01(config-if)#
*Nov  2 05:27:42.596: %LINK-3-UPDOWN: Interface GigabitEthernet8, changed state to down
*Nov  2 05:27:45.928: %LINK-3-UPDOWN: Interface GigabitEthernet8, changed state to up
*Nov  2 05:27:46.928: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet8, changed state to up

RT01(config)#int fa 0
RT01(config-if)#no shutdown
RT01(config-if)#
*Nov  2 05:30:31.955: %LINK-3-UPDOWN: Interface FastEthernet0, changed state to up
*Nov  2 05:30:32.955: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0, changed state to up
~~~

#### DHCPクライアント（C891FJ側）の設定
~~~
RT01(config-if)#ip address dhcp
RT01(config-if)#
*Nov  2 08:09:10.023: %DHCP-6-ADDRESS_ASSIGN: Interface GigabitEthernet8 assigned DHCP address 192.168.3.254, mask 255.255.255.0, hostname RT01
~~~

#### C891FJ～インターネットへの疎通確認（8.8.8.8）
~~~
RT01(config)#do ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 24/27/36 ms
~~~

#### C891FJのPC側IFへIPアドレスのアサイン
~~~
RT01(config)#int fa 0
RT01(config-if)#ip address 192.168.1.1 255.255.255.0
~~~

#### DHCPサーバの設定
~~~
RT01(config)#ip dhcp pool DHCPPOOL
RT01(dhcp-config)#network 192.168.1.0 255.255.255.0
RT01(dhcp-config)#default-router 192.168.1.1
RT01(dhcp-config)#lease 10
RT01(dhcp-config)#exit
RT01(config)#ip dhcp excluded-address 192.168.1.1

RT01(config)#do show ip dhcp pool

Pool DHCPPOOL :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 1
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.3          192.168.1.1      - 192.168.1.254     1
~~~
PC側 ipconfig /all 確認
~~~.ps1
イーサネット アダプター イーサネット 3:

   接続固有の DNS サフィックス . . . . .:
   説明. . . . . . . . . . . . . . . . .: Realtek USB GbE Family Controller
   物理アドレス. . . . . . . . . . . . .: 00-E0-4C-68-03-3D
   DHCP 有効 . . . . . . . . . . . . . .: はい
   自動構成有効. . . . . . . . . . . . .: はい
   IPv4 アドレス . . . . . . . . . . . .: 192.168.1.2(優先)
   サブネット マスク . . . . . . . . . .: 255.255.255.0
   リース取得. . . . . . . . . . . . . .: 2024年11月2日 17:44:06
   リースの有効期限. . . . . . . . . . .: 2024年11月3日 17:44:05
   デフォルト ゲートウェイ . . . . . . .:
   DHCP サーバー . . . . . . . . . . . .: 192.168.1.1
   NetBIOS over TCP/IP . . . . . . . . .: 有効
   ~~~

#### スタティックルートの設定
現状のルーティングテーブルを確認
~~~
RT01(config)#do sh ip route

Gateway of last resort is 192.168.3.1 to network 0.0.0.0

S*    0.0.0.0/0 [254/0] via 192.168.3.1
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, FastEthernet0
L        192.168.1.1/32 is directly connected, FastEthernet0
      192.168.3.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.3.0/24 is directly connected, GigabitEthernet8
L        192.168.3.254/32 is directly connected, GigabitEthernet8
~~~

~~~.ps1
PS C:\Users\hoge> tracert 8.8.8.8

8.8.8.8 へのルートをトレースしています。経由するホップ数は最大 30 です

  1  192.168.3.215  レポート: 宛先ホストに到達できません。
  ~~~

  PCにデフォルトルートが無かったので追加してみる
  ~~~.ps1
IPv4 ルート テーブル
===========================================================================
アクティブ ルート:
ネットワーク宛先        ネットマスク          ゲートウェイ       インターフェイス  メトリック
        127.0.0.0        255.0.0.0            リンク上         127.0.0.1    331
        127.0.0.1  255.255.255.255            リンク上         127.0.0.1    331
  127.255.255.255  255.255.255.255            リンク上         127.0.0.1    331
       172.16.0.0      255.255.0.0            リンク上        172.16.0.1    281
       172.16.0.1  255.255.255.255            リンク上        172.16.0.1    281
   172.16.255.255  255.255.255.255            リンク上        172.16.0.1    281
      192.168.1.0    255.255.255.0            リンク上       192.168.1.2    291
      192.168.1.2  255.255.255.255            リンク上       192.168.1.2    291
    192.168.1.255  255.255.255.255            リンク上       192.168.1.2    291
        224.0.0.0        240.0.0.0            リンク上         127.0.0.1    331
        224.0.0.0        240.0.0.0            リンク上        172.16.0.1    281
        224.0.0.0        240.0.0.0            リンク上       192.168.1.2    291
  255.255.255.255  255.255.255.255            リンク上         127.0.0.1    331
  255.255.255.255  255.255.255.255            リンク上        172.16.0.1    281
  255.255.255.255  255.255.255.255            リンク上       192.168.1.2    291
===========================================================================
固定ルート:
  なし 

PS C:\Users\hoge> route add 0.0.0.0 mask 0.0.0.0 192.168.1.1
 OK!
  ~~~

  #### C891FJ -> PCのPingがうまくいかない
  ~~~.ps1
--- PCからは疎通が取れる
  92.168.1.1 に ping を送信しています 32 バイトのデータ:
192.168.1.1 からの応答: バイト数 =32 時間 =1ms TTL=255
192.168.1.1 からの応答: バイト数 =32 時間 =1ms TTL=255
192.168.1.1 からの応答: バイト数 =32 時間 =1ms TTL=255
192.168.1.1 からの応答: バイト数 =32 時間 =1ms TTL=255

192.168.1.1 の ping 統計:
    パケット数: 送信 = 4、受信 = 4、損失 = 0 (0% の損失)、
ラウンド トリップの概算時間 (ミリ秒):
    最小 = 1ms、最大 = 1ms、平均 = 1ms
~~~
~~~
--- C891FJからのPingが通らない
RT01(config)#do ping 192.168.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
~~~
~~~.wireshark
---WireSharkでのキャプチャ結果
2923	7429.008234	192.168.1.2	8.8.8.8	ICMP	74	Echo (ping) request  id=0x0001, seq=96/24576, ttl=128 (no response found!)
2924	7433.669089	RealtekSemic_68:03:3d	Cisco_75:a9:9a	ARP	42	Who has 192.168.1.1? Tell 192.168.1.2
2925	7433.670461	Cisco_75:a9:9a	RealtekSemic_68:03:3d	ARP	60	192.168.1.1 is at 6c:ab:05:75:a9:9a
~~~

ファイアウォールの受信の規則でICMPv4を許可する。（作業後無効にする）

PC ⇔ C891FJのPing疎通を確認。
~~~
192.168.1.1 の ping 統計:
    パケット数: 送信 = 2、受信 = 2、損失 = 0 (0% の損失)、
ラウンド トリップの概算時間 (ミリ秒):
    最小 = 1ms、最大 = 1ms、平均 = 1ms

RT01#ping 192.168.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
~~~

##### NAPTの設定
~~~
RT01(config)#int gi 8
RT01(config-if)#ip nat outside
RT01(config-if)#exit
RT01(config)#int fa 0
RT01(config-if)#ip nat inside
RT01(config-if)#exit
RT01(config)#ip nat inside source list 10 interface gigabitEthernet 8 o
RT01(config)#$de source list 10 interface gigabitEthernet 8 overload
RT01(config)#exit

RT01(config)#do sh access-lists
Standard IP access list 10
    10 permit 192.168.3.0, wildcard bits 0.0.0.255
    20 permit any
~~~

#### ping8.8.8.8実施
~~~.ps1
PS C:\Users\hoge> ping 8.8.8.8

8.8.8.8 に ping を送信しています 32 バイトのデータ:
8.8.8.8 からの応答: バイト数 =32 時間 =47ms TTL=116
8.8.8.8 からの応答: バイト数 =32 時間 =41ms TTL=116

8.8.8.8 の ping 統計:
    パケット数: 送信 = 2、受信 = 2、損失 = 0 (0% の損失)、
ラウンド トリップの概算時間 (ミリ秒):
    最小 = 41ms、最大 = 47ms、平均 = 44ms
~~~
~~~
RT01(config)#do sh ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 192.168.3.254:1   192.168.1.2:1      8.8.8.8:1          8.8.8.8:1
RT01(config)#do sh ip nat statistics
Total active translations: 1 (0 static, 1 dynamic; 1 extended)
Peak translations: 1, occurred 00:02:52 ago
Outside interfaces:
  GigabitEthernet8
Inside interfaces:
  FastEthernet0
Hits: 14  Misses: 0
CEF Translated packets: 14, CEF Punted packets: 0
Expired translations: 1
Dynamic mappings:
-- Inside Source
[Id: 1] access-list 10 interface GigabitEthernet8 refcount 1

Total doors: 0
Appl doors: 0
Normal doors: 0
Queued Packets: 0
~~~
PC -> C891FJ -> AirTermimnal5 -> 8.8.8.8の疎通に成功

## 所感・課題
 - 名前解決が出来ていない。
 - ACLについて、Permit any頼りで実施しているので要検討。
 - teralogをそのままコミットしてしまい、実質ハードコーディングとなってしまったので、セキュアなGit使用を修得したい。
 - 簡単なハンズオンを網羅した後は2重NAT対策も行いたい。