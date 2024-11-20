# 1120
- [1120](#1120)
  - [CCNP学習](#ccnp学習)
    - [ループバックインターフェース/アドレス](#ループバックインターフェースアドレス)
      - [ループバックインターフェース](#ループバックインターフェース)
      - [ループバックアドレス](#ループバックアドレス)
    - [BGP](#bgp)
      - [キャッチアップ・問題演習](#キャッチアップ問題演習)
        - [BGPの状態遷移](#bgpの状態遷移)
        - [BGPメッセージタイプ](#bgpメッセージタイプ)
        - [パス属性](#パス属性)
        - [ベストパス選択優先順位](#ベストパス選択優先順位)
        - [ルートマップ](#ルートマップ)
        - [アドレスファミリ](#アドレスファミリ)
        - [BGPピアグループ](#bgpピアグループ)
        - [BGPのリセット](#bgpのリセット)
        - [確認コマンド](#確認コマンド)
        - [RIB-Failure](#rib-failure)
## CCNP学習

### ループバックインターフェース/アドレス

BGP等で、ループバック**インターフェース**を用いるシナリオを多々見かけるので今一度正確に理解する。

#### ループバックインターフェース

仮想的に作成される論理インターフェースであり、装置が動作している限りダウンしない。BGPに用いている物理インターフェースがダウンし、BGPセッションが切断されることのないように、`(config-router)# neighbor [addr] update-source [I/F] `コマンドを用いてループバックインターフェースがアサインされることがある。

複数作成可能。

#### ループバックアドレス
全くの別物。

単にホスト自身を示すIPアドレス。

127.0.0.1~127.255.255.255の値を取る。


参考:[ループバックアドレスの範囲と用途について](https://ja.stackoverflow.com/questions/95507/%E3%83%AB%E3%83%BC%E3%83%97%E3%83%90%E3%83%83%E3%82%AF%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9%E3%81%AE%E7%AF%84%E5%9B%B2%E3%81%A8%E7%94%A8%E9%80%94%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)

### BGP
Border Gateway Protocol

自律システム (AS) 間でルーティング情報を交換するために使用されるEGP 。

経路情報 (NLRI: Network Layer Reachability Information) の広告を行う。経路広告の目的は、特定のネットワークに到達可能であることを他のASに知らせることである。

BGPネイバー（ピア）は手動で設定する必要がある。

#### キャッチアップ・問題演習

##### BGPの状態遷移
1. Idle: 初期状態

- Idle（Admin）:`(config-router)#neighbor shutdown`によりピアがシャットダウンされている。

2. Connect: TCP接続待ち

3. Active: TCP接続試行中

4. Open Sent: Openメッセージ送信済み

5. Open Confirm: Openメッセージ確認済み

6. Established: セッション確立完了

##### BGPメッセージタイプ

- Open: セッション確立時に使用

- Update: 経路情報の更新に使用。Updateメッセージによって、新しい経路が広告されるか、既存の経路が取り消される。

- Notification: エラー通知に使用

- Keepalive: セッション維持に使用。デフォルトは60秒

- Route Refresh: 全経路情報の再要求に使用

##### パス属性

- 周知必須属性 (Well-known mandatory)

全てのBGPルータがサポートし、必ず広告しなければならない属性。これには、AS Path、Next Hop、Originなどが含まれる。

- 周知任意属性 (Well-known discretionary)

周知任意属性は、全てのBGPルータがサポートするが、広告が必須ではない属性。これには、Local Preference、Atomic Aggregateなどが含まれる。

- オプション推移的属性 (Optional transitive)

オプション推移的属性は、特定のBGPルータでサポートされていなくても、その属性を他のピアに引き継ぐことが可能な属性。これには、Aggregator、Communityなどが含まれる。

- オプション非推移的属性 (Optional non-transitive)

オプション非推移的属性は、特定のBGPルータでサポートされていない場合、その属性を他のピアに引き継がない属性。これには、MEDなどが含まれる。

##### ベストパス選択優先順位

1. Weight（Cisco独自、大きい値が優先）Well-Known Discretionary

WeightはCisco独自の属性で、特定のルートに対する優先度を設定する。大きい値が優先される。この属性はルータ内部(試験ではローカルと云われる)のみで使用され、他のルータには影響を与えない。

2. Local Preference（大きい値が優先）Well-Known Discretionary

Local PreferenceはAS内での経路選択に使用され、大きい値が優先される。これはAS内の全てのルータに対して広告される属性である。

通常、出口ポイントの選択に使用される。

3. ローカルで生成されたルート Well-Known Mandatory

ネットワークコマンドで生成されたルートや、aggregate-addressコマンドで生成されたルートは、他のルートよりも優先される。

4. AS_Path長（短いほど優先）Well-Known Mandatory

AS_Pathは通過したASのリストであり、短いAS Pathを持つ経路が優先される。

ループ防止のため、AS_Pathに自身のASNが含まれていた場合はルートを破棄する。　

ただし、MPLSトランジットAS等を挟んで同一ASと接続する場合はループとならないため破棄させないように設定をする必要がある。

- 両端の同一ASで設定する場合(allowas-in):
~~~
R1(config)#router bgp 65001
R1(config-router)#neighbor 192.168.0.1 allowas-in

R3(config)#router bgp 65001
R3(config-router)#neighbor 192.168.1.1 allowas-in
~~~

- 内側のASがAS_Pathを書き換える場合(as-override):
~~~
R2(config)#router bgp 65002
R2(config-router)#neighbor 192.168.1.2 as-override
~~~



5. Origin（IGP > EGP > Incomplete）Well-Known Mandatory

Origin属性は経路の生成元を示し、IGP (Interior Gateway Protocol) が最も優先され、次にEGP、最後にIncomplete（不明または手動再配布）となる。

6. MED（小さい値が優先） Optional Non-Transitive

MED (Multi-Exit Discriminator) は異なるAS間で使用され、他のASに対して特定の経路を選択する際の好ましさを示す。小さい値が優先される。

通常、複数の接続ポイントがある場合に使用される。

7. eBGP > iBGP 

eBGP経由で学習した経路はiBGP経由で学習した経路よりも優先される。これは、外部からの経路を内部の経路よりも好むという基本的なBGPのポリシーに基づいている。

8. IGPメトリック（小さい値が優先） 

複数のパスが同じ場合は、IGPメトリックが小さい経路が選択される。これにより、内部ネットワーク内で最も効率的なパスが選ばれる。

9. 最後に、ルータIDが小さいBGPピアから伝えられた経路を優先する。


##### ルートマップ
BGPでは、経路制御を柔軟に行うために、Route-MapやACLを組み合わせて使用することが多い。

特定の条件に一致する経路の属性を変更したり、広告する経路を制御するために利用する。例えば、特定のネイバーに対してのみ特定のプレフィックスを広告する場合や、MEDやLocal Preferenceの設定を動的に変更する場合に使用する。

~~~
(config)#route-map SET-LOCALPREF permit 10 //シーケンス10番目のエントリーに対して許可

(config-route-map)#match ip address 1 //ACL1に該当した場合のアクションを指定

(config-route-map)#set local-preference 200

(config)#router bgp 65001

(config-router)#neighbor 192.168.1.1 route-map SET-LOCALPREF out
~~~

##### アドレスファミリ
MP-BGP (Multiprotocol BGP) に関する問題として出題される。
異なるタイプの経路情報を単一のBGPセッションで交換できるようにするための機能。IPv4、IPv6、VPNv4、VPNv6など、様々なプロトコルをサポートする。

アドレスファミリの設定手順

IPv6の有効化（必要な場合）

`(config)# ipv6 unicast-routing`

BGPプロセスの起動

`(config)# router bgp <as-number>`

アドレスファミリの設定

`(config-router)# address-family <address-family-type>`

例：IPv6ユニキャストの場合

`(config-router)# address-family ipv6 unicast`

アドレスファミリ固有の設定
アドレスファミリごとに必要な設定を行う。

ネイバーのアクティブ化

`(config-router-af)# neighbor <neighbor-address> activate`

ネットワークの広告

`(config-router-af)# network <network-address>`

アドレスファミリの終了

`(config-router-af)# exit-address-family`


##### BGPピアグループ

複数のピアに対して同じ設定を繰り返すことが多い場合、ピアグループを利用することで設定効率が向上する。

ピアグループの主な特徴とルール

- ローカルでのみ有効: ピアグループはローカルルータ内でのみ有効である。

- 設定の効率化: 設定をグループごとにまとめられるため、設定効率が向上する。

- UPDATEメッセージの最適化: UPDATEメッセージがグループごとに生成されるようになるため、ルータの負荷が減少する。

- アウトバウンドポリシーの一貫性: グループ内のピアには共通のアウトバウンドポリシーが適用されるが、個別に設定することはできない。

- インバウンドポリシーの柔軟性: インバウンドのポリシーについては、個別に設定することが可能である。

BGPピアグループの設定

1. ピアグループを作成する
~~~
(config)# router bgp {AS番号}
(config-router)# neighbor {グループ名} peer-group
~~~

2.ピアグループごとに共通になるパラメータを設定（オプション）
~~~
(config-router)# neighbor {グループ名} remote-as {AS番号}
(config-router)# neighbor {グループ名} update-source {インターフェース}
(config-router)# neighbor {グループ名} route-map {ルートマップ名} {in | out}
~~~

3.ピアグループにまとめるピアを指定

~~~
(config-router)# neighbor {ピアのIPアドレス} peer-group {グループ名}
~~~

##### BGPのリセット

- ハードリセット:`#clear ip bgp *`

全てのBGPセッションを再確立する。これにより、全ての経路情報が再広告されるため、ネットワークへの影響が大きい。


- ソフトリセット:`#clear ip bgp * soft`

BGPセッションを中断せずに、ピアとの経路情報を再評価する。影響を最小限に抑えながら設定変更を適用するために使用される。

##### 確認コマンド

- `#show ip bgp`

BGPテーブル内の経路情報を表示する。各経路のパス属性（AS Path、Next Hopなど）や状態を確認できる。

- `#show ip route bgp`

ルーティングテーブルにおいて、BGPによって学習された経路を表示する。BGPで得られた経路がルーティングテーブルにどのように適用されているかを確認するのに使用する。

- `#show ip bgp summary`

BGPネイバーの状態を要約して表示する。ピアの状態、受信および送信したプレフィックスの数、Uptimeなどの情報を確認できる

- `#show running-config`

現在の設定内容を表示する。BGPの設定が正しく反映されているかを確認するために使用する。

##### RIB-Failure
`#show ip bgp`にて、"RIB-Failure"と表示されている場合、BGPで受信した経路情報がルーティングテーブル（RIB: Routing Information Base）にインストールできない場合を指す。RIB-Failureは、次のような理由で発生することがある

- より優れたルートが存在する場合

他のプロトコル（例えば、OSPFやEIGRP）によって、より低い管理距離を持つ経路がすでにインストールされている場合、BGP経路はRIBにインストールされない。このため、BGP経路は「RIB-Failure」となる。

- ルーティングループの回避

ループを回避するために、特定の経路がRIBにインストールされないことがある。この場合も「RIB-Failure」として表示される。

- RIB内に既に同一プレフィックスが存在する場合

すでにRIBに同一のプレフィックスが存在しており、その経路が他のプロトコルで学習されたものの場合、BGPで学習した経路がRIBにインストールされない。

