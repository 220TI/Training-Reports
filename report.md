- [ロギング・加工](#ロギング加工)
  - [表示例](#表示例)
    - [先頭の記号](#先頭の記号)
  - [ログの保管](#ログの保管)
  - [ログの永続化](#ログの永続化)
  - [ログのリモート保存](#ログのリモート保存)
    - [バッファの確認](#バッファの確認)
  - [ログの重大レベル](#ログの重大レベル)
- [オペレーションTips](#オペレーションtips)
  - [domain-lockup等の強制終了](#domain-lockup等の強制終了)
  - [showコマンドの実行時間を記録する](#showコマンドの実行時間を記録する)
  - [showコマンドの行番号を表示する](#showコマンドの行番号を表示する)

# ロギング・加工

## 表示例
~~~
*Mar  1 00:00:00.273: %VIRTIO-3-INIT_FAIL: Failed to initialize device, PCI 0/8/0/1002 , device is disabled, not supported
~~~
### 先頭の記号
- *: システムクロックが同期されていない状態（日付と時刻が正確ではない状態）
- ,: システムクロックが同期されたことがあるが、現在は同期されていない状態
- 何もない: システムクロックが同期されている状態（日付と時刻が信頼できる状態）

## ログの保管
ログは機器の内部バッファにため込まれるが、上限を超えると古いメッセージから上書きされる。

## ログの永続化
機器を再起動してもログを保持するために、RAM以外に保存することができる。
~~~
RT2(config)#logging persistent size 1048576 filesize 262144
~~~


## ログのリモート保存
機器内部のバッファだけでなく、外部のSyslogサーバにログを送信することで、長期間のログ保存や一元管理が可能となる。

Syslogサーバの設定
以下のコマンドでSyslogサーバのIPアドレスを指定する。
~~~
RT2(config)#logging host <SyslogサーバのIPアドレス>
~~~
また、Syslogに送信するログのレベルも設定できる。

~~~
RT2(config)#logging trap informational
~~~

### バッファの確認
`#show logging`の最下部で確認できるため、includeを用いてフィルタしようとしたが、大文字小文字のマッチを考慮する必要があった。
~~~
RT2#sh logging | include buffer
# 何も表示されない

RT2#sh logging | include Buffer
    Buffer logging:  level debugging, 30 messages logged, xml disabled,
Log Buffer (8192 bytes):
# 大文字と小文字が区別されている
~~~
このケースを考慮するのであれば、任意の一文字にマッチする"."を活用するといいかもしれない。
~~~
RT2#sh logging | include .uffer
    Buffer logging:  level debugging, 30 messages logged, xml disabled,
Log Buffer (8192 bytes):
~~~

ただし、少なくともrunning-configは殆ど小文字の模様(?)

バッファのサイズやロギングするプライオリティは下記のコマンドで変更できる。ログフィルター用のdiscriminatorエントリもここで作ることができる。
~~~
RT2(config)#logging buffered ?
  <0-7>              Logging severity level # レベル
  <4096-2147483647>  Logging buffer size　　# バッファサイズ
  alerts             Immediate action needed           (severity=1)
  critical           Critical conditions               (severity=2)
  debugging          Debugging messages                (severity=7)
  discriminator      Establish MD-Buffer association
  emergencies        System is unusable                (severity=0)
  errors             Error conditions                  (severity=3)
  filtered           Enable filtered logging
  informational      Informational messages            (severity=6)
  notifications      Normal but significant conditions (severity=5)
  warnings           Warning conditions                (severity=4)
  xml                Enable logging in XML to XML logging buffer
  ~~~
  ただし、メモリにも限りがあるので空き容量を確認する。
  ~~~
  RT2#sh version | section mem
Cisco IOSv (revision 1.0) with  with 460001K/62464K bytes of memory.
  ~~~

## ログの重大レベル
(RFC 3164)
| 数字 | レベル    |                                 |
|------|----------|--------------------------------|
| 0    | Emergency| system is unusable             |
| 1    | Alert    |action must be taken immediately   |
| 2    | Critical | critical conditions             |
| 3    | Error    | error conditions                   |
| 4    | Warning  | warning conditions               |
| 5    | Notice   | normal but significant condition  |
| 6    | Informational| informational messages     |
| 7    | Debug| debug-level messages               |

# オペレーションTips

## domain-lockup等の強制終了
`Ctrl + Shit + 6`

## showコマンドの実行時間を記録する
~~~
router(config)#line console 0
router(config-line)#exec prompt timestamp
~~~
~~~
RT2#sh run
Load for five secs: 0%/0%; one minute: 0%; five minutes: 0%
Time source is hardware calendar, *05:45:30.107 UTC Fri Nov 29 2024
~~~
teralogに残す際に便利。

## showコマンドの行番号を表示する
~~~
RT2#sh run  linenum
Load for five secs: 0%/0%; one minute: 0%; five minutes: 0%
Time source is hardware calendar, *05:46:59.807 UTC Fri Nov 29 2024

Building configuration...

Current configuration : 3114 bytes
     1 : !
     2 : ! Last configuration change at 05:45:26 UTC Fri Nov 29 2024
     3 : !
     4 : version 15.9
~~~