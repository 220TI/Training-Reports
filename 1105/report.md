1105

# LPIC 2学習

## トピック
 - [システムロギング](#システムロギング)
 - [ハード監視](#ハード監視)

## 作業
 - トピックまとめ
 - Ping-t問題演習(主題204記憶装置へのアクセス方法の調整)

### システムロギング
参画予定の案件においてもログ確認やディスク確認等があり、
LPIC 2試験においては主題213に該当するとのことだが、
現場での作業を想定すると不十分に感じたので別途掘り下げる...

ログの実務的な見方等を修得出来ればと思ったが、
tail -fに一工夫する程度や、監視ツールの使用が現実的らしい。

#### rsyslog
/etc/rsyslog.conf にルール等が設定されている。

#### logrotate
/etc/logrotate.conf に周期等が設定されている。

#### strace,ltace
straceは、トレース対象のプログラムが発行するシステムコールと受信したシグナルを出力するコマンドです。

> ターミナルを2つ使います。ターミナル1でテストプログラムを起動します。[ltraceコマンドの使い方](https://hana-shin.hatenablog.com/entry/2022/12/02/212751)

VScodeでブレークポイントを設けて実行し、ターミナルで確認する。
~~~.bash
[root@localhost project1]# ps aux | grep test
root        2240 21.3  0.1   4436   816 ?        Ss   20:38   0:11 /root/test/project1/test
~~~

ltrace実行
~~~.bash
[root@localhost project1]# ltrace -p 2240
Cannot attach to pid 2240: Operation not permitted
~~~
プロセスにアタッチができない。

ライブラリが静的リンクになっていないか確認
~~~.bash
[root@localhost project1]# ldd test
        linux-vdso.so.1 (0x00007ffe447b9000)
        libc.so.6 => /lib64/libc.so.6 (0x00007efe990c2000)
        /lib64/ld-linux-x86-64.so.2 (0x00007efe99498000)
~~~
straceも失敗する。
~~~.bash
[root@localhost project1]# strace -p 4785
strace: attach: ptrace(PTRACE_SEIZE, 4785): 許可されていない操作です
~~~

プロセス名で指定したら成功した...
~~~.bash
[root@localhost project1]# ltrace test
strrchr("test", '/')                                     = nil
...（略）
~~~

STATがtsだとプロセスにアタッチできないかも...↓
~~~.bash
[root@localhost project1]# ps -p 4785 -o stat
STAT
ts
~~~

straceやltraceをPID指定で実行すると、既存のプロセスにアタッチするが、プロセス名で指定すると、新規にプロセスを起動する。

⇒ STATがts（停止状態）である既存のプロセスにはアタッチできない。
~~ブレークポイントで停止させて観測したことによる事象。~~

⇒ と思ったが、無限ループさせたプログラム（STATがSs（割り込み可能なスリープ））であってもPIDによるアタッチが不可であった。
~~~.bash
[root@localhost project1]# ps -p 5190 -o stat
STAT
Ss
[root@localhost project1]# ltrace -p 5190
Cannot attach to pid 5190: Operation not permitted
[root@localhost project1]# strace -p 5190
strace: attach: ptrace(PTRACE_SEIZE, 5190): 許可されていない操作です
~~~

#### ps
上記等で多く触り、初見のオプションもあったのでまとめ

##### プロセス名で指定(-Cオプション)
~~~.bash
[root@localhost project1]# ps -C test
    PID TTY          TIME CMD
   2240 ?        00:00:51 test
~~~

##### STAT
| STAT | Discription|
| :---: | :---------- |
| R | 実行可能状態/実行状態|
| D | 割り込み不能な待機状態。IO待ち|
| S | 割り込み可能な待機状態。プログラムの指示で自発的にスリープしている。 |
| T | STOPシグナルで停止している。|
| t | トレース中にデバッガによって停止されている。|
| Z |ゾンビ状態。親プロセスによる終了確認を待っている。|

### ハード監視
主題203や204など
構築・作成よりも表示・確認系の問題の演練を優先...

#### デバイスファイル対応表
| デバイスファイル | 説明                        |
|-----------------|-----------------------------|
| /dev/sda        | SCSI/SATA HDDの1番目       |
| /dev/sdb        | SCSI/SATA HDDの2番目       |
| /dev/sdc        | SCSI/SATA HDDの3番目       |
| /dev/sdd        | SCSI/SATA HDDの4番目       |
| /dev/hda        | IDEのPrimary Master        |
| /dev/hdb        | IDEのPrimary Slave         |
| /dev/hdc        | IDEのSecondary Master      |
| /dev/hdd        | IDEのSecondary Slave       |
| /dev/sr0        | SCSI/SATA CD-ROMの1番目    |
| /dev/sr1        | SCSI/SATA CD-ROMの2番目    |
| /dev/st0        | SCSIテープの1番目          |
| /dev/st1        | SCSIテープの2番目          |

#### udev
デバイスにアクセスするための/devの下のデバイスファイルを動的に作成、削除する仕組みを提供する。

カーネルから検知・切断したデバイスの情報（uevent）を受けて

/lib/udev/rules.d/hoge.rules　← デフォルト・ルール
/etc/udev/rules.d/hoge.rules　← カスタマイズ・ルール

のルールに従って/dev下のデバイスファイルを作成あるいは削除する。

##### udevadm
udevの管理コマンド。
monitor･test･trigger等のサブコマンドがある。

#### hdparm
ハードディスクのパラメータを表示、設定する。

SCSIデバイス用としてsdparmが存在する。

Usage:  hdparm  [options] [device ...]

設定されている転送モードなどを確認
~~~.bash
[root@localhost ~]# hdparm  /dev/sda1

/dev/sda1:
 multcount     = 128 (on)
 （略）
~~~
ドライブから直接詳細情報を得る。(-Iオプション)
~~~.bash
[root@localhost ~]# hdparm -I /dev/sda1

/dev/sda1:

ATA device, with non-removable media
(略)
~~~
-t:バッファキャッシュオフで読み込み速度を計測

-T:バッファキャッシュのみで読み込み速度を計測
~~~.bash
[root@localhost ~]# hdparm -t /dev/sda1

/dev/sda1:
 Timing buffered disk reads: 222 MB in  3.00 seconds =  73.91 MB/sec
[root@localhost ~]# hdparm -T /dev/sda1

/dev/sda1:
 Timing cached reads:   9646 MB in  1.99 seconds = 4849.60 MB/sec
 ~~~

典型的な使用例として、読み込み速度の計測・スピンアップ/ダウン・DMA転送の有効・無効などが頻出。

#### SSD（NANDメモリ）について
ファイルを削除しても、データブロックが解放されるだけでデータが消去されるわけではなく、NANDメモリはデータの上書きが出来ない。

データを消去してから書き込みを行うことになるので、パフォーマンスが低下する。

下記で改善を図る。
 
 - TRIMコマンド(fstrim)の利用
    不要データが記録されたセクタをドライブに通知する。
 - discardオプションをつけてファイルシステムをマウントする。（非推奨）
    書き込み時に自動的にTRIMされる。

ガベージコレクションやウェアレベリング等のソリューションもあるが、管理者が明示的に設定するようなものではない。

#### AHCI
SATAデバイスと通信するための規格。
SATAとの関係の理解に難儀している。
 > Advanced Host Controller Interface (AHCI) は、Serial ATA Revision 2.0 と密接な関連があるがホストコントローラーの独立した規格でありシリアルATA規格には含まれない。(Wikipedia)

[この画像をかみ砕いていきたい](https://ascii.jp/elem/000/000/817/817807/2/)

#### NVMe
PCI ExpressバスからSSDに接続するための規格。
マザーボード・システムボードに直接挿す。

/dev/nvme0n1 といった規則のデバイスファイル名となる。

## 課題
- トラブルシューティングや情報収集という観点で更に学習する。
- サーバにはSASストレージが搭載されていることが多いと思うので、SASについて深堀する。（HDDとSSDの中間の性能であり、コスパが良く採用されているというイメージ...）
- fstrimを実施しようとしたところ失敗。VirtualBOXが起因しているか・・・？
~~~.bash
[root@localhost ~]# fstrim /
fstrim: /: the discard operation is not supported
~~~

