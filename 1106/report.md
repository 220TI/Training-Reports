1106

# LPIC2学習
引き続き、オペレーション・保守に係るトピックを中心に学習する。

## トピック
 - [ユーザ管理](#ユーザ管理)
 - [システムの起動](#システムの起動)

## ユーザ管理
rootで作業していたため、一般ユーザでの作業を癖付ける。

### ユーザ追加
useradd
~~~.bash
[root@localhost ~]# useradd TI
~~~
passwd
~~~.bash
[root@localhost ~]# passwd TI
ユーザー TI のパスワードを変更。
新しいパスワード:
新しいパスワードを再入力してください:
passwd: すべての認証トークンが正しく更新できました。
~~~

### sudo権限付与
#### /etc/sudoersについて
/etc/sudoersにて、ユーザグループwheelがすべてのコマンドをroot権限で実行できるように定義されている。
~~~.bash
## Same thing without a password
# %wheel        ALL=(ALL)       NOPASSWD: ALL
~~~

特定のグループに一部の権限を与えることも可能。
~~~.bash
## Allows members of the users group to mount and unmount the
## cdrom as root
# %users  ALL=/sbin/mount /mnt/cdrom, /sbin/umount /mnt/cdrom
~~~

#### ユーザTIをグループwheelに追加
~~~.bash
[root@localhost ~]# usermod -G wheel TI
[root@localhost ~]# id TI
uid=1000(TI) gid=1000(TI) groups=1000(TI),10(wheel)
~~~

#### sudo実行
~~~.bash
[TI@localhost ~]$ less /etc/sudoers
/etc/sudoers: 許可がありません
[TI@localhost ~]$ sudo less /etc/sudoers

あなたはシステム管理者から通常の講習を受けたはずです。
これは通常、以下の3点に要約されます:

    #1) 他人のプライバシーを尊重すること。
    #2) タイプする前に考えること。
    #3) 大いなる力には大いなる責任が伴うこと。

[sudo] TI のパスワード:
~~~

sudoの場合はエイリアスが使えない。
~~~.bash
[TI@localhost ~]$ sudo ll /root
[sudo] TI のパスワード:
sudo: ll: コマンドが見つかりません
~~~

"sudo␣"のエイリアスを張るとエイリアスが判定されるようになるとのこと。

    ⇒ [sudoコマンドでaliasを使えるようにする](https://qiita.com/homoluctus/items/ba1a6d03df85e65fc85a)
## システムの起動
主題202

### Systemd
参考:[systemdエッセンシャル](https://speakerdeck.com/moriwaka/systemd-intro)

#### 用語集

 | word    | discription |
 | ------  | ---------   |
 | unit    | systemdにおける役割の単位 |
 | target  | 依存関係等の定義 |
 | service | プロセスを実行しサービスを提供する |
 | unit file | unitの設定ファイル。これをロードしてメモリ上のunitを作成する。 |

#### unit間の依存関係
 - Wants 

 依存関係が解決出来なくても稼働する。

 - Requires

 依存関係が解決出来なければ稼働しない。

#### unitの定義ファイルのディレクトリ(反映優先順位降順)

 - /etc/systemd（管理者による設定）
 - /run/systemd（実行時の自動設定など）
 - /usr/lib/systemd（rpmパッケージで提供）

#### systemd-delta コマンド

変更された設定ファイルを表示する。
上記[ユニットの定義ファイルのディレクトリ(反映優先順位降順)](#ユニットの定義ファイルのディレクトリ(反映優先順位降順))によるオーバーライドを表示できる。

#### systemd-analyze plotコマンド（svg出力）
unitの初期化開始・終了のタイミングを図示する。

[1106ディレクトリに添付](https://github.com/220TI/Training-Reports/blob/master/1106/systemd-analyze.svg)

#### systemctl コマンド
- systemctl -t [service|target|etc...]

ユニットを状態に関わらず表示

- systemctl list-units -type [service|target|etc...]

アクティブなユニットのみ表示

- systemctl list-unit-files

サービスユニットがブート時に自動的に起動するかどうかの情報(STATE)

- systemctl list-dependencies

ユニットの依存関係をツリー表示。

- systemctl is-enabled <unit>

ユニットの自動起動設定を返す。シェルスクリプトに組み込んで判定に用いる等の活用法がある。

- 普段使っている一部のコマンドがsystemctlへのシンボリックリンクになっている。
~~~.bash
 [TI@localhost ~]$ file /sbin/halt
/sbin/halt: symbolic link to ../bin/systemctl
~~~

#### daemon

 ⇒ daemonにするメリット

    - 端末から切り離され、シグナルを受け付けない。
        
        ⇒ 反対に、端末で稼働させた"tail -f"等はシグナルを受け付ける。

    -  stdin,stdout,stderrが/dev/nullにリダイレクトされる。
        
        ⇒ 画面に不要な情報が出力されない。

    - ユーザがログアウトしてもプロセスが終了しない。

## 課題
- システムの起動～根幹的な機能の管理～アプリ稼働管理など様々な機能を担っているのでキャッチアップに難儀。
- getty/udev/journaldなど様々な機能がsystemdの配下で稼働しているので、これらを交えて学ぶと得点率が上がると思慮。
- fork()やexec()などの知見があると好ましいので、C言語の学習も余力を見ながら実施したい。