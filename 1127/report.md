- [Ethercannel](#ethercannel)
  - [SW~サーバ間のEtherchannel構築](#swサーバ間のetherchannel構築)
    - [SW設定](#sw設定)
    - [サーバ設定](#サーバ設定)
      - [チームとチームの設定(connection)を作成](#チームとチームの設定connectionを作成)
      - [チームのconnectionを編集](#チームのconnectionを編集)
      - [チームにIFを追加](#チームにifを追加)
      - [IFの設定を切り離す](#ifの設定を切り離す)
      - [チームデバイスの管理](#チームデバイスの管理)
      - [パケットキャプチャ](#パケットキャプチャ)

# Ethercannel
モード対応早見表

| モード           | on   | auto (PAgP) | desirable (PAgP) | passive (LACP) | active (LACP) |
|-----------------|------|-------------|------------------|----------------|---------------|
| **on**          | ○    | ×           | ×                | ×              | ×             |
| **auto (PAgP)** | ×    | ×           | ○                | -              | -             |
| **desirable (PAgP)** | × | ○         | ○                | -              | -             |
| **passive (LACP)** | ×  | -           | -                | ×              | ○             |
| **active (LACP)** | ×   | -           | -                | ○              | ○             |

Cisco独自プロトコルであるDTPと同じく、"desirable""とauto"というモードがあると覚える。

## SW~サーバ間のEtherchannel構築

> NICチーミングを使用すると、複数の物理ネットワークインターフェイスと仮想ネットワークインターフェイスを、NICチームと呼ばれる1つの論理仮想アダプタに結合できます。 
この設計では、スイッチとサーバ間のEtherChannel接続を示します。
この場合、スイッチ側から見ると、EtherChannelは、ピア側から実行されているプロトコルに応じて、ONモードまたはLACPアクティブ/パッシブモードのいずれかで設定できます。(EtherChannel設計 - Cisco)
![pic1](https://www.cisco.com/c/dam/en/us/support/docs/lan-switching/gigabit-etherchannel-gec/222012-etherchannel-designs-08.png)

現場ではRHELが想定されるが、CML2にRH系のOSが無いのでAlmalinux8のisoをインポートする。

[CML2を使いこなす。（その12：カスタムイメージの導入）](https://qiita.com/sakai00kou/items/343d3e5782c164367dde)

### SW設定
~~~
SW1(config)#int range gi 0/0-1
SW1(config-if-range)#channel-group 1 mode on
Creating a port-channel interface Port-channel 1

SW1(config-if-range)#
*Nov 27 08:44:30.209: %LINK-3-UPDOWN: Interface Port-channel1, changed state to up
*Nov 27 08:44:31.209: %LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel1, changed state to up
~~~

### サーバ設定
~~~
[cisco@almalinux ~]$ less /etc/redhat-release
AlmaLinux release 8.10 (Cerulean Leopard)

[cisco@almalinux ~]$ uname -a
Linux almalinux 4.18.0-553.16.1.el8_10.x86_64 #1 SMP Thu Aug 8 07:11:46 EDT 2024 x86_64 x86_64 x86_64 GNU/Linux
~~~

#### チームとチームの設定(connection)を作成
~~~
[cisco@almalinux ~]$ sudo nmcli c add type team ifname team0 con-name team-team0
~~~
#### チームのconnectionを編集
~~~
[cisco@almalinux ~]$ sudo nmcli c mod team-team0 ipv4.method manual \
> ipv4.address "192.168.1.10/24"
~~~
このようにコマンドが長い場合は折り返されてしまうので、バックスラッシュで改行をする。

#### チームにIFを追加
~~~
[cisco@almalinux ~]$ sudo nmcli c type team-slave ifname eth0 \
\ con-name team-slave-eth0 master team-team0
#(eth1も同様)
~~~

#### IFの設定を切り離す
~~~
[cisco@almalinux ~]$ sudo nmcli c del "System eth0"
Connection 'System eth0' (5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03) successfully deleted.
[cisco@almalinux ~]$ [ 8240.405706] team0: Port device eth0 added
[ 8240.407708] IPv6: ADDRCONF(NETDEV_CHANGE): team0: link becomes ready
# チームの設定が適用された
~~~
~~~
[cisco@almalinux ~]$ nmcli
team0: connected to team-team0
        "team0"
        team, 52:54:00:1A:58:31, sw, mtu 1500
        inet4 192.168.1.10/24
        route4 192.168.1.0/24 metric 350
        inet6 fe80::d4d:ca7f:6267:9805/64
        route6 fe80::/64 metric 1024

eth0: connected to team-slave-eth0
        "Red Hat Virtio"
        ethernet (virtio_net), 52:54:00:1A:58:31, hw, mtu 1500
        master team0

eth1: connected to team-slave-eth1
        "Red Hat Virtio"
        ethernet (virtio_net), 52:54:00:1A:58:31, hw, mtu 1500
        master team0
~~~

#### チームデバイスの管理
teamdctlコマンドで確認できる。
~~~
[cisco@almalinux ~]$ sudo teamdctl team0 state
setup:
  runner: roundrobin # 両系でラウンドロビンされている
ports:
  eth0
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
~~~

#### パケットキャプチャ
![pic1](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1127/1127_1.png)

サーバ⇒SW1のパケットは分散されているが、逆は分散されていない。

サーバからのパケットもチーミングにより単一の送信元（MACアドレス）から送られることになっていたので、ロードバランスアルゴリズムを”src-dst-ip”に変更する。

~~~
SW1(config)#port-channel load-balance src-dst-ip
~~~