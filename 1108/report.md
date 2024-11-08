# 1108

## 現場スキルキャッチアップ

- [TFTPサーバを用いてのC891FJconfigバックアップ](#TFTPサーバを用いてのC891FJconfigバックアップ)
    - [TFTPサーバ側作業](#TFTPサーバ側作業)
    - [C891FJ側作業](#C891FJ側作業)
    - [ハマったこと](#ハマったこと)
- [C891FJへのSSH接続](#C891FJへのSSH接続)
    - [スイッチポートにVLANを割り当て、アクセスポートとする。](#スイッチポートにVLANを割り当て、アクセスポートとする。)
- [TFTPサーバにバックアップしていたrunning-configと現在の差異(diff)を取る。](#TFTPサーバにバックアップしていたrunning-configと現在の差異(diff)を取る。)

### TFTPサーバを用いてのC891FJconfigバックアップ

#### TFTPサーバ側作業

##### TFTPサーバ・クライアントのインストール
~~~.bash
sudo dnf -y install tftp-server tftp
~~~

##### クライアントの書き込みを許可する起動オプション(-c)を付与する。
~~~.bash
sudo vi /usr/lib/systemd/system/tftp.service

[Unit]
Description=Tftp Server
Requires=tftp.socket
Documentation=man:in.tftpd

[Service]
ExecStart=/usr/sbin/in.tftpd -c -s /var/lib/tftpboot
StandardInput=socket

[Install]
Also=tftp.socket
~~~

##### firewalldにてtftpを許可
~~~.bash
sudo firewall-cmd -add-service=tftp --zone=public --permanent
~~~

##### tftpサービス起動・自動起動設定
tftp.socketを有効にすると対応するtftp.serviceが有効になる。

~~~.bash
sudo systemctl start tftp.socket

sudo systemctl enable tftp
Created symlink /etc/systemd/system/sockets.target.wants/tftp.socket �� /usr/lib/systemd/system/tftp.socket.

# sockets.target.wantsからtftp.socketへのシンボリックリンクが貼られる。
# これによりtftp.serviceと同様に自動起動される。(sockets.target ⇒ tftp.socket ⇒ tftp.serviceの依存関係)
~~~

##### ~~TFTPルートディレクトリの権限変更~~

**tftpdの-cオプションのみでTFTP送信は可能であり、本作業は不要であった。**

~~~.bash
sudo chown nobody:nobody /var/lib/tftpboot

sudo chmod -R 777 /var/lib/tftpboot
~~~

##### C891FJへの疎通確認
~~~.bash
ping 192.168.3.254
PING 192.168.3.254 (192.168.3.254) 56(84) bytes of data.
64 bytes from 192.168.3.254: icmp_seq=1 ttl=255 time=3.31 ms
~~~

#### C891FJ側作業

##### TFTPサーバへの疎通確認
~~~
RT01(config)#do ping 192.168.3.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.3, timeout is 2 seconds:
...!!
Success rate is 40 percent (2/5), round-trip min/avg/max = 4/40/76 ms
~~~

##### TFTPサーバへのrunning-config送信
~~~
RT01#copy running-config tftp:
Address or name of remote host []? 192.168.3.3
Destination filename [rt01-confg]?
!!
2134 bytes copied in 0.248 secs (8605 bytes/sec)
~~~

##### TFTPサーバ側での確認
~~~.bash
[TI@localhost root]$ ll /var/lib/tftpboot/
���v 8
-rw-rw-rw-  1 nobody nobody 2134 11��  7 21:32 rt01-confg
~~~

#### ハマったこと

~~~
%Error opening tftp://192.168.3.3/rt01-confg (No such file or directory)
~~~
⇒systemdユニットファイルに-cオプションが適用されていなかったためC891FJによるTFTP送信に失敗した。

~~~
%Error opening tftp://192.168.3.3/rt01-confg (Timed out)
~~~
⇒systemdユニットファイルの設定が誤っており、tftpが稼働できていなかった。

### C891FJへのSSH接続

C891FJのスイッチポート（GigabitEthernet0~7）をPCと繋ぎ、SSH接続が出来るようにする。

#### スイッチポートにVLANを割り当て、アクセスポートとする。
##### VLAN10作成
~~~
RT01(config)#vlan 10
RT01(config-vlan)#exit
~~~

##### VLAN10へIPアドレスを割り当てる。
~~~
RT01(config)#int vlan 10
RT01(config-if)#ip address 192.168.1.1 255.255.255.0
RT01(config-if)#no shutdown
~~~

##### スイッチポート（GigabitEthernet0）をアクセスポートにする。
~~~
RT01(config-if)#int gi 0
RT01(config-if)#switchport mode access
~~~

##### VLAN10をNATのインサイドにする。
~~~
RT01(config)#int vlan 10
RT01(config-if)#ip nat inside
RT01(config-if)#
*Nov  8 05:52:42.128: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to up
RT01(config-if)#exit
~~~

~~~RT01(config)#do sh ip int brie
Interface                  IP-Address      OK? Method Status                Protocol
Async3                     unassigned      YES unset  down                  down
BRI0                       unassigned      YES NVRAM  administratively down down
BRI0:1                     unassigned      YES unset  administratively down down
BRI0:2                     unassigned      YES unset  administratively down down
FastEthernet0              unassigned      YES manual down                  down
GigabitEthernet0           unassigned      YES unset  up                    up
GigabitEthernet1           unassigned      YES unset  down                  down
GigabitEthernet2           unassigned      YES unset  down                  down
GigabitEthernet3           unassigned      YES unset  down                  down
GigabitEthernet4           unassigned      YES unset  down                  down
GigabitEthernet5           unassigned      YES unset  down                  down
GigabitEthernet6           unassigned      YES unset  down                  down
GigabitEthernet7           unassigned      YES unset  down                  down
GigabitEthernet8           192.168.3.254   YES DHCP   up                    up
NVI0                       unassigned      YES unset  up                    up
Vlan1                      unassigned      YES manual down                  down
Vlan10                     192.168.1.1     YES manual up                    up
~~~

#### 自動ログアウトの設定を変更
Console接続/SSH接続共に、時間経過でログアウトされてしまうのでこれを無効にする。現場ではポリシーに応じて設定することが好ましい。
~~~
# Console
RT01(config)#line con 0
RT01(config-line)#exec-timeout 0
RT01(config-line)#exit

# SSH/Telnet
RT01(config)#line vty 0 4
RT01(config-line)#exec-timeout 0
~~~

### TFTPサーバにバックアップしていたrunning-configと現在の差異(diff)を取る。

#### TFTPサーバからバックアップをフラッシュメモリに取得する。

~~~ 
RT01#copy tftp: flash:
Address or name of remote host []? 192.168.3.3
Source filename []? rt01-confg
Destination filename [rt01-confg]? backup-config
Accessing tftp://192.168.3.3/rt01-confg...
Loading rt01-confg from 192.168.3.3 (via GigabitEthernet8): !
[OK - 2134 bytes]

2134 bytes copied in 0.548 secs (3894 bytes/sec)

RT01#dir
Directory of flash:/

  407  -rw-         660   Nov 8 2024 05:49:06 +00:00  vlan.dat
  408  -rw-        2134   Nov 8 2024 06:34:42 +00:00  backup-config
    1  -rw-        3068  Jul 18 2019 10:35:22 +00:00  cpconfig-8xx.cfg
    2  -rw-    89610452  Oct 19 2019 03:31:48 +00:00  c800-universalk9-mz.SPA.155-3.M6a.bin
    3  drw-           0  Jul 18 2019 10:36:00 +00:00  ccpexp
  406  -rw-       22747  Jul 18 2019 10:39:24 +00:00  home.html
~~~

#### show archive config differencesを用いてdiffを取る。

~~~
RT01#show archive config differences flash:backup-config system:run
RT01#$e config differences flash:backup-config system:running-config
!Contextual Config Diffs:
+username hoge password 0 huga
interface FastEthernet0
 +no ip address
interface GigabitEthernet0
 +switchport access vlan 10
+interface Vlan10
 +ip address 192.168.1.1 255.255.255.0
 +ip nat inside
 +ip virtual-reassembly in
+ip ssh version 2
line con 0
 +exec-timeout 0 0
 +length 0
line vty 0 4
 +exec-timeout 0 0
 +login local
interface FastEthernet0
 -ip address 192.168.1.1 255.255.255.0
line vty 0 4
 -login
~~~

本日実施した作業による差異が分かる。

### CML

#### ハマったこと

##### Ethernet0に使用できるPCIeスロットがありません
[VMware WorkstationでCisco Modeling Labs 2（CML2）を起動しようとした際に、よく遭遇するエラーメッセージ「Ethernet0に使用できるPCIeスロットがありません」について](https://pythonjp.ikitai.net/entry/2024/10/07/001107)
 自宅だとUSB有線LAN、宅外だと無線LANなので要調整

或いは、仮想マシンのバージョンによるものか
##### 仮想AMD-V/RVI はこのプラットフォームではサポートされていません。
仮想化をネストしようとしまっているものと思われる。

- 本PCでHyper-Vを使っていた時の名残でWindowハイパーバイザプラットフォームが有効になっていたので無効にした。
- VirtualBOXもアンインストール。
- 下記ps1も実施
~~~.ps1
bcdedit /set hypervisorlaunchtype off
~~~
- セキュリティのコア分離を無効にした。

ラボの稼働を確認した