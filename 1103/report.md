## 作業予定

1. [Hyper-VにTFTPサーバ構築](#Hyper-VにTFTPサーバ構築)
        ･ AlmaLinux8構築
            - 構築のためインターネットへの接続
        ･ TFTP導入
2. [C891FJのrunning-configをTFTPサーバにコピー](#C891FJのrunning-configをTFTPサーバにコピー)
        ･ C891FJ⇔TFTPサーバの疎通

[（一部のmd記法キャッチアップ）](#一部のmd記法キャッチアップ)

## 作業

### Hyper-VにTFTPサーバ構築

#### yum update
ゲストOS-Hyper-Vの外部SW-AirTerminal5の疎通が取れなかったので、下記を参照の上デフォルトGWを追加
[Wi-Fi接続してるのに、インターネットにアクセスできない](https://zenn.dev/yukitezuka/scraps/0f9bffd34dacc0)

yum updateをしようとしたところ、Baseの名前解決が出来ない。
調査したところ、/etc/resolv.confが存在しなかった。
NetworkManagerで追加
~~~.bash
nmcli conn mod ens192 +ipv4.dns 8.8.8.8
~~~
NetworkManagerを再起動
~~~.bash
systemctl restart NetworkManager
~~~
/etc/resolv.confの生成及びyum updateの成功を確認。

#### TFTP導入
どのTFTPサービスがいいの？
...

### C891FJのrunning-configをTFTPサーバにコピー
[これをやりたい](https://www.cisco.com/c/ja_jp/support/docs/ios-nx-os-software/ios-software-releases-122-mainline/46741-backup-config.html)

### 一部のmd記法キャッチアップ
文書内リンクについてキャッチアップをし、本頁に盛り込んだ。
[ymmt2005/grpc-tutorial](https://github.com/ymmt2005/grpc-tutorial/blob/main/README.md?plain%3D1)を参考にした。
Qiitaもmdの状態で参照できるようなので今後は参考にしたい...
