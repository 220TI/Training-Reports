# 1121

## BGP構築

- RT2とRT3でeBGPを設定する。
  
![pic1](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1121/1121_1.png)

~~~
RT2(config)#router bgp 10

RT2(config-router)#neighbor 10.1.1.2 remote-as 20

RT2(config-router)#network 1.0.0.0 
~~~
~~~
RT3(config-router)#neighbor 10.1.1.1 remote-as 10

RT3(config-router)#
*Nov 21 03:38:39.095: %BGP-5-ADJCHANGE: neighbor 10.1.1.1 Up

RT3(config-router)#network 20.0.1.0 
~~~

↑サブネットマスクを指定していなかったためトラブルとなるが、下記で補足する...

`show ip bgp`、`show ip bgp summary` 確認

~~~
RT2#sh ip bgp
BGP table version is 2, local router ID is 10.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.0.0.0          0.0.0.0                  0         32768 i

 RT2#sh ip bgp summary
BGP router identifier 10.1.1.1, local AS number 10
BGP table version is 2, main routing table version 2
1 network entries using 144 bytes of memory
1 path entries using 84 bytes of memory
1/1 BGP path/bestpath attribute entries using 160 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 388 total bytes of memory
BGP activity 1/0 prefixes, 1/0 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.2        4           20     102     102        2    0    0 01:28:50        0
 ~~~

~~~
RT3#sh ip bgp
BGP table version is 2, local router ID is 20.0.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.0.0.0          10.1.1.1                 0             0 10 i

 RT3#sh ip bgp summary
BGP router identifier 20.0.1.1, local AS number 20
BGP table version is 2, main routing table version 2
1 network entries using 144 bytes of memory
1 path entries using 84 bytes of memory
1/1 BGP path/bestpath attribute entries using 160 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 412 total bytes of memory
BGP activity 1/0 prefixes, 1/0 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4           10     100     100        2    0    0 01:27:17        1
 ~~~

ネイバの確立までは確認できたが、互いにAS内のルートを広告できていない。

### 原因
ルーティングテーブルにないルートを配送することは出来ないため。

また、今回はクラスレスアドレス`20.0.1.0/24`を用いたため、配送の際は確実にサブネットマスクの指定をしなければならない。

~~~
RT2(config)#ip route 1.0.0.0 255.0.0.0 Gi 0/1
~~~

~~~
RT3(config)#ip route 20.0.1.0 255.255.255.0 gigabitEthernet 0/1

RT3(config)#router bgp 20
RT3(config-router)#network 20.0.1.0 mask 255.255.255.0
~~~

それぞれBGPでのルートを学習出来たことを確認する。

~~~
RT2#sh ip route bgp
(略)
B        20.0.1.0 [20/0] via 10.1.1.2, 00:15:27
~~~
~~~
RT3#sh ip route bgp
(略)
B     1.0.0.0/8 [20/0] via 10.1.1.1, 03:10:45
~~~

## 再配送
実際の環境では複数のルーティングプロトコルが混在している。例えば、AS内部でのルーティングはOSPF等のIGPが用いられ、AS間のルーティングではBGPが用いられる。この際、BGPピアにOSPFで学習したルートを再配送する場合がある。

![pic2](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1121/1121_2.png)

### 必須知識
- それぞれのルーティングプロトコル

  - RIP
  - OSPF
  - EIGRP
  - BGP
- ルーティングループの防止
- route-map
- 
