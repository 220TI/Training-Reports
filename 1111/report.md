# 1111

## 現場スキルキャッチアップ

### インターネットVPN(GRE)
"Tunnel"の理解を旨とし、サイト間VPNで学習コストの低いとされるGREについて学習する。

#### 調査

##### GRE
- ペイロードにGREヘッダと新しいIPヘッダを付与し、カプセル化する。
- マルチキャストに対応している。

    ⇒ OSPFのhelloメッセージ等を転送できる。
- 暗号化は出来ないため、IPsecと併用されることが多い。
- Tunnnel
 - サイト間を仮想的に接続する
 - カプセル化することで、サポートされていないプロトコルを使用することができる。（例:IPv6 over IPv4 など）
 - Tunnelに使用しているインターフェースがダウンした場合、Tunnelもダウンする。

##### ループバックアドレス
TunnelインターフェースにIPアドレスを割り当てずに、ループバックアドレスを用いる。

以下のようなメリットがある。
- ループバックアドレスはデバイスを代表するIPアドレスとして扱うことができ、管理コストを抑えることが出来る。
- 他のインターフェースとは異なり、機器が稼働さえしていれば常にUP状態を保つ。

#### 構築

##### CMLのそれぞれの端末にTeraTermで接続
WebConsoleのIPアドレスにSSH接続した後、コマンドでラボ内のノードに接続することができる。
~~~
open /Lab/Node/Line
~~~
Ctrl + ^ + ] でノードから切断。

##### Alpine Linux
CMLでエンドポイントをデプロイする際に最もリソース効率が良いと思われるディストリビューション。コンテナや組み込みで用いられているらしい。

ネットワークconfigファイルの記述要領がRedHat系と異なる。

~~~./etc/network/interfaces
auto eth0
iface eth0 inet static
        address 10.1.1.0
        netmask 255.255.255.0
        gateway 10.1.1.12
~~~
ネットワークのリスタート
~~~.ash
/etc/network # /etc/init.d/networking restart
~~~

##### 初期設定済みconfigを流し込む

 - RIPv2の設定
 - R11及びR21にてTunnnelインターフェースを作成する。

running-configをCONFIGタブにコピペして流し込むことが出来る。

##### PC1からPC2へping実施

~~~.ash
/etc/network # ping 10.2.1.100
PING 10.2.1.100 (10.2.1.100): 56 data bytes
64 bytes from 10.2.1.100: seq=0 ttl=60 time=6.304 ms
~~~

##### パケットキャプチャ
Tunnel間のICMPパケットはデータ長が長く、GREヘッダ・新しいIPヘッダが付与されていることがわかる。

PC1～R12間の例
~~~
Frame 9: 98 bytes on wire (784 bits), 98 bytes captured (784 bits)
Ethernet II, Src: RealtekU_05:50:f8 (52:54:00:05:50:f8), Dst: RealtekU_12:f4:d6 (52:54:00:12:f4:d6)
Internet Protocol Version 4, Src: 10.2.1.100, Dst: 10.1.1.100
Internet Control Message Protocol
~~~

R11~ISP間の例
~~~
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: RealtekU_11:69:55 (52:54:00:11:69:55), Dst: RealtekU_0a:3e:e8 (52:54:00:0a:3e:e8)
Internet Protocol Version 4, Src: 100.1.1.11, Dst: 100.2.2.21 # 新しいIPヘッダ
Generic Routing Encapsulation (IP) # GREヘッダ
Internet Protocol Version 4, Src: 10.1.1.100, Dst: 10.2.1.100
Internet Control Message Protocol
~~~

また、RIPのマルチキャストアドレス宛て(Dst: 224.0.0.9)のデータを転送出来ていることもわかる。
~~~
Frame 4: 170 bytes on wire (1360 bits), 170 bytes captured (1360 bits)
Ethernet II, Src: RealtekU_10:72:c1 (52:54:00:10:72:c1), Dst: RealtekU_09:09:8b (52:54:00:09:09:8b)
Internet Protocol Version 4, Src: 100.1.1.11, Dst: 100.2.2.21
Generic Routing Encapsulation (IP)
Internet Protocol Version 4, Src: 10.1.0.11, Dst: 224.0.0.9
User Datagram Protocol, Src Port: 520, Dst Port: 520
Routing Information Protocol
~~~