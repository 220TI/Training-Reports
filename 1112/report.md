# 1112

## 現場スキルキャッチアップ
 - [VRRP](#VRRP)
    - [概要・調査](#概要・調査)
    - [構築](#構築)
 - [CML内にUbuntuのデスクトップ環境を用意する](#CML内にUbuntuのデスクトップ環境を用意する)
### VRRP

#### 概要・調査
DFGW冗長化のためのFHRPの一つ。
Cisco独自のFHRPと異なり、RFCで定義されベンダフリーである。

##### VRID
- グループ番号
- 仮想ルータのMACアドレスの下2桁にグループ番号が用いられているので、異なる仮想ルータでVRIDが重複すると障害が発生する。

##### プライオリティ
 - 0～255の値をとる(今回のIOSvだと1~254であった)
 - 大きいほうが優先度が高い

##### VRRPアドバタイズメント（Hello）
 - マスタールータのみが送信する。
 - バックアップルータがアドバタイズメントを受信できずに送信間隔の3倍の時間が経過すると、次に優先度の高いルータがマスタールータになる。

##### プリエンプト
　- 有効時にバックアップルータが自身より低いプライオリティを含むアドバタイズメントを受信すると、アドバタイズメントを送信する。（マスタールータに昇格しようとする）

##### 状態
 - Init（初期）
 - Backup
 - Master

#### 構築

##### それぞれのIFでVRRPを有効にする
~~~
RT1(config)#int gi 0/1
RT1(config-if)#vrrp 1 ip 192.168.1.254
~~~

~~~
RT2(config)#int gi 0/1
RT2(config-if)#vrrp 1 ip 192.168.1.254
~~~

##### RT1がマスタールータとなるようにプライオリティを設定する
~~~
RT1(config)#int gi 0/1
RT1(config-if)#vrrp 1 priority 254

RT2(config)#int gi 0/1
RT2(config-if)#vrrp 1 priority 1
~~~

##### show vrrp確認
~~~
 RT1#sh vrrp
GigabitEthernet0/1 - Group 1
  State is Master
  Virtual IP address is 192.168.1.254
  Virtual MAC address is 0000.5e00.0101
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 254
  Master Router is 192.168.1.1 (local), priority is 254
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.007 sec

  RT2(config)#do sh vrrp
GigabitEthernet0/1 - Group 1
  State is Backup
  Virtual IP address is 192.168.1.254
  Virtual MAC address is 0000.5e00.0101
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 1
  Master Router is 192.168.1.1, priority is 254
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.996 sec (expires in 3.267 sec)
~~~

 ##### PC1から仮想IPアドレスへ疎通確認
~~~
  / # ping 192.168.1.254
PING 192.168.1.254 (192.168.1.254): 56 data bytes
64 bytes from 192.168.1.254: seq=0 ttl=255 time=4.571 ms
~~~

##### 切り替えテスト
SW1～RT2間のコネクションを切断する。
~~~
/ # ping 192.168.1.254
PING 192.168.1.254 (192.168.1.254): 56 data bytes
64 bytes from 192.168.1.254: seq=0 ttl=255 time=2.941 ms
64 bytes from 192.168.1.254: seq=1 ttl=255 time=3.317 ms
64 bytes from 192.168.1.254: seq=2 ttl=255 time=2.943 ms
64 bytes from 192.168.1.254: seq=3 ttl=255 time=3.347 ms
64 bytes from 192.168.1.254: seq=4 ttl=255 time=3.271 ms
64 bytes from 192.168.1.254: seq=5 ttl=255 time=2.957 ms
64 bytes from 192.168.1.254: seq=6 ttl=255 time=3.594 ms
# ここでコネクションを切断
# seq=12から復帰する。
64 bytes from 192.168.1.254: seq=12 ttl=255 time=2.636 ms

# 以下、RT2
*Nov 12 05:32:37.474: %VRRP-6-STATECHANGE: Gi0/1 Grp 1 state Backup -> Master

RT2#sh vrrp
GigabitEthernet0/1 - Group 1
  State is Master # RT2がマスターに昇格している。
  Virtual IP address is 192.168.1.254
  Virtual MAC address is 0000.5e00.0101
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 1
  Master Router is 192.168.1.2 (local), priority is 1
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.996 sec
~~~

RT2がマスタールータになると、アドバタイズメントを送信するようになる。
~~~
Frame 16: 60 bytes on wire (480 bits), 60 bytes captured (480 bits)
Ethernet II, Src: IETF-VRRP-VRID_01 (00:00:5e:00:01:01), Dst: IPv4mcast_12 (01:00:5e:00:00:12)
Internet Protocol Version 4, Src: 192.168.1.2, Dst: 224.0.0.18
Virtual Router Redundancy Protocol
Version 2, Packet type 1 (Advertisement)
Virtual Rtr ID: 1
Priority: 1 (Non-default backup priority)
Addr Count: 1
Auth Type: No Authentication (0)
Adver Int: 1
Checksum: 0x1b56 [correct]
Checksum Status: Good
IP Address: 192.168.1.254
~~~

##### 復帰・プリエンプト確認
RT1が復帰し、RT2がバックアップに降格する。
~~~
RT2#
*Nov 12 05:37:39.272: %VRRP-6-STATECHANGE: Gi0/1 Grp 1 state Master -> Backup

RT2#sh vrrp
GigabitEthernet0/1 - Group 1
  State is Backup
  Virtual IP address is 192.168.1.254
  Virtual MAC address is 0000.5e00.0101
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 1
  Master Router is 192.168.1.1, priority is 254
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.996 sec (expires in 3.425 sec)
~~~

### CML内にUbuntuのデスクトップ環境を用意する
ASAvのASDM(GUI)に接続するため、Ubuntuのデスクトップ環境を用意する。

External Connectorにブリッジインターフェースをアタッチし、Ubuntuと接続するとインターネットと導通する。

~~~
sudo apt update
~~~

~~~
sudo apt upgrade
~~~

~~~
sudo apt-get -y install ubuntu-desktop
~~~