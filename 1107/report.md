# 1107

## トピック
 - [LPIC2学習](#LPIC2学習)
    - [ネットワーク構成](#ネットワーク構成)
 - [現場スキルキャッチアップ](#現場スキルキャッチアップ)
    - [LINUX](#LINUX)
    - [ネットワーク学習](#ネットワーク学習)
    - [VMware調査](#VMware調査)
    - [WindowsServer2022](#WindowsServer2022)

## LPIC2学習

### ネットワーク構成

#### ifconfigコマンド
現在非推奨となっているnet-toolsパッケージの一部。

ioctl()を用いてnet_device構造体を制御

open()を用いて/proc/net/devを参照して表示

#### ipコマンド

##### ip link show
ifconfigの後継

データリンク情報の表示
~~~.bash
[TI@localhost root]$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:4c:93:7b brd ff:ff:ff:ff:ff:ff
~~~

IP情報の表示
~~~.bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:4c:93:7b brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 85397sec preferred_lft 85397sec
    inet6 fe80::5462:953f:b783:fe60/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
~~~

ipコマンドはipctl()ではなく、Netlink(カーネルのサブシステム)を用いる。

##### IPv6アドレスに関する設定
~~~.bash
[TI@localhost root]$ ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 fe80::5462:953f:b783:fe60/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
~~~

#### 仮想インターフェース

ハイパーバイザの仮想NICよりも追加が容易で、再起動が不要。

##### ifconfigコマンドの場合
~~~.bash
[TI@localhost root]$ ifconfig enp0s3:1 192.168.1.100
(ifconfig未導入のため結果無し)
~~~
~~~.bash
[TI@localhost root]$ ifconfig enp0s3:sub1 192.168.1.100
(ifconfig未導入のため結果無し)
~~~

物理インターフェースの後ろに区切り番号":"を付ける。

##### ipコマンドの場合
ifconfigコマンドと異なり区切り番号の指定は無くても良い。

~~~.bash
[TI@localhost root]$ sudo ip addr add 192.168.1.100/24 dev enp0s3 label enp0s3:0
[TI@localhost root]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:4c:93:7b brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 83591sec preferred_lft 83591sec
    inet 192.168.1.100/24 scope global enp0s3:0
       valid_lft forever preferred_lft forever
~~~


上記で追加した仮想インターフェースを無効にする。
~~~.bash
[TI@localhost root]$ sudo ip addr del 192.168.1.100 dev enp0s3
Warning: Executing wildcard deletion to stay compatible with old scripts.
         Explicitly specify the prefix length (192.168.1.100/32) to avoid this warning.
         This special behaviour is likely to disappear in further releases,
         fix your scripts! # プレフィックス長による警告
[TI@localhost root]$ ip addr
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:4c:93:7b brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 83514sec preferred_lft 83514sec
~~~

### 無線LAN(WLAN)
#### iwconfig/iwlist
現在非推奨となっているWireless Toolsパッケージの一部。

#### iw
上記の後継。
~~~.bash
[TI@localhost ~]$ sudo dnf install iw -y
~~~

## 現場スキルキャッチアップ

### LINUX

#### sosreport
sosパッケージの一部。

Redhatにサポートを依頼する際に提供する設定ファイルやログファイル等が生成される。
~~~.bash
[TI@localhost ~]$ sudo sosreport
[sudo] TI のパスワード:
Please note the 'sosreport' command has been deprecated in favor of the new 'sos' command, E.G. 'sos report'. # 現在は"sos report"と分離されている。
Redirecting to 'sos report '
(略)
Your sosreport has been generated and saved in:
	/var/tmp/sosreport-localhost-2024-11-06-xpdlfxc.tar.xz
~~~
実際にRedhatにサポートを依頼する場合は問題番号を入力する。
~~~.bash
Optionally, please enter the case id that you are generating this report for []:
~~~

#### ls -ltr
ls -a や ll をよく用いていたのでキャッチアップ。

manにてltrそれぞれのオプションを確認。
~~~.bash
-l     use a long listing format
-t     sort by modification time, newest first
-r, --reverse
              reverse order while sorting
~~~
多くの情報（権限など）を表示 ⇒ 更新順（降順）でソート ⇒ 逆順（昇順）表示という操作となる。

最下に最新の更新が表示される。
~~~.bash
[TI@localhost ~]$ ls -ltr /etc/yum.repos.d/
合計 72
-rw-rw-r--. 1 root root   203  5月 15 05:38 pgadmin4.repo
-rw-r--r--. 1 root root  2666  5月 22 10:49 almalinux.repo
-rw-r--r--. 1 root root   928  5月 22 10:49 almalinux-saphana.repo
-rw-r--r--. 1 root root   873  5月 22 10:49 almalinux-sap.repo
-rw-r--r--. 1 root root   871  5月 22 10:49 almalinux-rt.repo
-rw-r--r--. 1 root root  1041  5月 22 10:49 almalinux-resilientstorage.repo
-rw-r--r--. 1 root root   963  5月 22 10:49 almalinux-powertools.repo
-rw-r--r--. 1 root root   885  5月 22 10:49 almalinux-plus.repo
-rw-r--r--. 1 root root   905  5月 22 10:49 almalinux-nfv.repo
-rw-r--r--. 1 root root   943  5月 22 10:49 almalinux-ha.repo
-rw-r--r--. 1 root root 13297  7月 22 03:17 pgdg-redhat-all.repo
-rw-r--r--  1 root root 13587  8月  7 02:45 pgdg-redhat-all.repo.rpmnew
~~~

### ネットワーク学習
CML(Cisco Modeling Labs)の調査
 - PacketTracerとことなり、実際のCiscoIOSが導入された機器をエミュレートすることができる。
 - サポートされているハイパーバイザがVMware製品のみ。VMware有償化と聞いていたので諦めかけていたが、無償で個人利用できる製品があるらしい...↓
 - 本日、購入・ダウンロードのみ実施。

### VMware調査
 - VMware Workstation ProやVMware Fusion Proが個人利用に限り無償化したらしい...
[VMware Workstation Pro と VMware Fusion Pro の個人利用版が無償になったので使ってみた (2024 年 5 月)](https://qiita.com/sanjushi003/items/b4ba2687f99412fd7c38)

上記の通りBROADCOMに登録しダウンロード。

- Windows Server 2022を入れてみる。

memo: OSのインストールをした後は、isoのDVD/CDをアンマウントすることを心がける。

 ⇒ マウントされたままだとvMotionやデプロイに失敗する。

### WindowsServer2022

#### パフォーマンスモニタ
 [Windows Server のパフォーマンスを監視する](https://learn.microsoft.com/ja-jp/training/modules/monitor-windows-server-performance/)

カウンターを追加してグラフ監視が可能。
##### データコレクターセット
パフォーマンスのロギング。

- 閾値でアラートを生成させることができる。
- 取得の開始・停止タイミングも定義可能。
- 取得を開始すると、パフォーマンスモニタファイルが[生成される。](https://github.com/220TI/Training-Reports/blob/master/1107/DataCollector.png)