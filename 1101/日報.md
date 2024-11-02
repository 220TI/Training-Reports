# 11.01

## Git復習（研修報告用）
### git init
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git init
Initialized empty Git repository in C:/Users/hoge/Desktop/TrainingReports/.git/
~~~

### git add
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git add .\日報.txt
PS C:\Users\hoge\Desktop\TrainingReports> git add .\pktimg1101.png
~~~

### git status
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   pktimg1101.png
        new file:   "\346\227\245\345\240\261.txt"
~~~
### git commit 
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git commit .\日報.txt
PS C:\Users\hoge\Desktop\TrainingReports> git commit .\pktimg1101.png
~~~
### GitHubのリポジトリにpushする
#### git remote add
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git remote add origin https://github.com/220TI/Training-Reports.git
~~~

#### git push
~~~.ps1
PS C:\Users\hoge\Desktop\TrainingReports> git push -u origin master
* remote: Repository not found.
fatal: repository 'https://github.com/220TI/Training-Reports.git/' not found *
~~~
GitHubとの疎通或いは認証に失敗。
[「Repository not found」になるGithub Private Repositoryの扱い方](https://qiita.com/hosikiti/items/03bba19abec4b789d7a5)
を参照し、クライアント鍵ペアを作成。GitHubに公開鍵を登録。

### SW-PC間のコンソール接続～SSH接続

#### SW01の管理VLANにIPアドレスを設定しないとSWにSSH接続が出来ない。
~~~.pkt
SW01(config)#int vlan 1
SW01(config-if)#ip address 192.168.1.2 255.255.255.0
~~~

#### 作業のログ取得を失念したので、show running config及びshow crypt keyを確認し、下記手順を網羅していることを確認。
1. ホスト名を設定する
2. ドメイン名を設定する
3. RSA暗号鍵を生成する
4. ユーザアカウントを作成する
5. ローカル認証を設定する
6. SSHの接続許可を設定する
7. SSHのバージョンを設定する

~~~.pkt
SW01#show running config

interface Vlan1
(略)
hostname SW01
!
enable secret 5 $1$foobar
!
ip ssh version 2
ip domain-name test.com
!
username hoge privilege 1 password 0 huga

 ip address 192.168.1.2 255.255.255.0
!
line con 0
!
line vty 0 4
 login local
 transport input ssh
line vty 5 15
 login
 ~~~

 ~~~.pkt
 SW01#show crypto key mypubkey rsa 
% Key pair was generated at: 0:1:14 UTC 3月 1 1993
Key name: SW01.test.com
(略)
~~~

### NTP
~~~.pkt
SW01(config)#do show clock
*0:18:21.275 UTC Mon Mar 1 1993
---時間が誤っている。

SW01(config)#do show ntp status
Clock is unsynchronized, stratum 16, no reference clock
nominal freq is 250.0000 Hz, actual freq is 249.9990 Hz, precision is 2**24
reference time is 00000000.00000000 (00:00:00.000 UTC Mon Jan 1 1990)
clock offset is 0.00 msec, root delay is 0.00  msec
root dispersion is 0.00 msec, peer dispersion is 0.00 msec.
loopfilter state is 'FSET' (Drift set from file), drift is - 0.000001193 s/s system poll interval is 4, never updated.

SW01(config)#do show clock
*0:52:36.329 UTC Mon Mar 1 1993
SW01(config)#do show clock
*0:52:50.403 UTC Mon Mar 1 1993
---少し待つと反映される。
SW01(config)#do show clock
11:32:7.162 UTC Fri Nov 1 2024
~~~

## JP1/AJS3構築調査
### サーバ群
 - Manager
 - Agent
 - Viewクライアント(PC)