# 1122

## Terraform

### 削除
`terraform destroy`

`terraform plan -destroy`を用いると削除の実行計画を確認することができる。

~~~
Plan: 0 to add, 0 to change, 2 to destroy.
~~~
先日デプロイしたラボを削除。ローカルに残ったファイルは手動で削除して良い。

### import

cml2_lifecycleというリソースを用いて、CMLからエクスポートしたyamlを読み込ませる。

単にyamlを手動でCMLにインポートする手法との比較が残課題...

ステージングの制御は考慮せずに実施

~~~main.tf
resource "cml2_lifecycle""test"{
      topology = templatefile("topology.yaml",{labname = "import test"})
      staging = {
        stages = [""]
      }
}
~~~
~~~ps1
PS C:\Users\hoge\Desktop\terraform\cml_import> terraform apply
（略）
cml2_lifecycle.test: Creation complete after 3m36s [id=ae114036-2b6f-40c3-a3c6-6c69e23817e2]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
~~~
全てのノードを同時に起動しようとするため、ある程度の制御はした方が好ましい。

[topology_import.tf](https://github.com/CiscoDevNet/terraform-provider-cml2/blob/main/examples/resources/cml2_lifecycle/topology_import.tf)

### ドリフト

Terraformのstateファイルと実際のインフラに不整合が発生することをドリフトという。

`terraform apply`した後のインフラにマネジメントコンソール等で変更を加えると不整合が生じる。
そのため、Terraformで管理するクラウドは一貫してTerraformでオペレーションすることとした方が好ましい。

しかし、CML2の検証というユースケースにおいては、ある程度共通するコンフィグ等の自動化さえ出来れば良く、技術の検証はCML2上で実施するため塩梅を検討する必要があるだろう。

### トポロジーのデプロイ

EIGRP検証のため、ルータを複数プロビショニングする。
 
 - 複数のルータを配置
 - `logging synchronous`等の必須コンフィグを盛り込む
 - 機微な情報を`terraform.tfvars`に退避。

![pic1](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1122/1122_1.png)

ひし形にそれぞれ接続するイメージで書いてみる

<details><summary>main.tf</summary>

    variable address{}
    variable username{}
    variable password {}

    terraform {
    required_providers {
        cml2 = {
        source  = "registry.terraform.io/ciscodevnet/cml2"
        version = "~> 0.7.0"
        }
    }
    }

    provider "cml2" {
    address     = var.address
    username    = var.username
    password    = var.password
    skip_verify = true
    }

    resource "cml2_lab" "lab1" {
    title = "EIGRP"
    }

    resource "cml2_node" "rt1" {
    lab_id         = cml2_lab.lab1.id
    nodedefinition = "iosv"
    configuration = file("config.txt")
    label          = "RT1"
    x              = -120
    y              = 0
    }

    resource "cml2_node" "rt2" {
    lab_id         = cml2_lab.lab1.id
    nodedefinition = "iosv"
    configuration = file("config.txt")
    label          = "RT2"
    x              = 0
    y              = -120
    }

    resource "cml2_node" "rt3" {
    lab_id         = cml2_lab.lab1.id
    nodedefinition = "iosv"
    configuration = file("config.txt")
    label          = "RT3"
    x              = 120
    y              = 0
    }

    resource "cml2_node" "rt4" {
    lab_id         = cml2_lab.lab1.id
    nodedefinition = "iosv"
    configuration = file("config.txt")
    label          = "RT4"
    x              = 0
    y              = 120
    }

    resource "cml2_link" "link1" {
    lab_id = cml2_lab.lab1.id
    node_a = cml2_node.rt1.id
    slot_a = 1
    node_b = cml2_node.rt2.id
    slot_b = 0
    }

    resource "cml2_link" "link2" {
    lab_id = cml2_lab.lab1.id
    node_a = cml2_node.rt2.id
    slot_a = 1
    node_b = cml2_node.rt3.id
    slot_b = 0
    }

    resource "cml2_link" "link3" {
    lab_id = cml2_lab.lab1.id
    node_a = cml2_node.rt3.id
    slot_a = 1
    node_b = cml2_node.rt4.id
    slot_b = 0
    }

    resource "cml2_link" "link4" {
    lab_id = cml2_lab.lab1.id
    node_a = cml2_node.rt4.id
    slot_a = 1
    node_b = cml2_node.rt1.id
    slot_b = 0
    }

</details>

<details><summary>config</summary>

    hostname unnamedRT
    !
    no ip domain lookup
    !
    line con 0
    exec-timeout 0 0
    logging synchronous
    line aux 0
    line vty 0 4
    exec-timeout 0 0
    logging synchronous
    !
    end
    

</details>
file()で読み込ませるファイルの改行コードはLFにすること。

IPアドレスのアサインやhostname等の機器により異なる部分の書き方が課題

![pic2](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1122/1122_2.png)


## EIGRP

### EIGRPの特徴

- 拡張ディスタンスベクタ型ルーティング（ハイブリッドルーティングとも呼ばれる）: ディスタンスベクタ型のルーティングプロトコルで、経路計算に複数のメトリックを使用する。

- DUAL（Diffusing Update Algorithm）による高速収束: ネットワークの状態変化に対して迅速に収束し、安定性を保つ。

- 低いCPU負荷: ルーターの負荷が少なく、効率的に経路選択が可能。

- 自動集約: デフォルトで自動的にネットワークを集約するが、手動で集約の調整も可能。

- 手動によるルート集約: インターフェース単位で手動でルート集約が設定できる。

- 不等コストロードバランシング: バリアンス値の設定により、コストの異なる経路でも負荷分散が可能。

- 3つのテーブルを保持: 「ネイバーテーブル」「トポロジテーブル」「ルーティングテーブル」の3つのテーブルを持つ。

### K値
EIGRPで用いられる復号メトリック。

- K1（帯域幅）: リンクの帯域幅。
- 
- K2（負荷）: リンクの現在の負荷。

- K3（遅延）: パケットがリンクを通過するのにかかる時間。

- K4（信頼性）: リンクの信頼性を示す値。

- K5（MTU）: 最大伝送単位。

デフォルトでは、帯域幅と遅延のみがメトリック計算に使われ、他のK値は0として扱われる。

### 経路集約

例えば、複数のサブネット（192.168.1.0/24、192.168.2.0/24、192.168.3.0/24）を持つ環境で、これらを1つの経路に集約することで、ルーティングテーブルのサイズを削減できる。
~~~
Router(config)# interface serial 0/0
Router(config-if)# ip summary-address eigrp 1 192.168.0.0 255.255.252.0
~~~
この設定により、192.168.1.0/24、192.168.2.0/24、192.168.3.0/24 の3つのサブネットを192.168.0.0/22に集約し、ルーティングテーブルの簡素化が可能になる。このように集約することで、他のルーターに送るルーティング情報が少なくなり、ネットワークの安定性や効率が向上する。

### FDとRD

- FD (Feasible Distance): 自ルータから宛先ネットワークまでの合計メトリック。

- RD (Reported Distance): ネイバルータから宛先ネットワークまでの合計メトリック（ネイバから報告されるメトリック）。RDはAD (Advertised Distance) とも呼ばれる。

### サクセサとフィージブルサクセサ

- サクセサ: EIGRPの最適経路のネクストホップで、FDが最も小さいネイバー。

- フィージブルサクセサ:サクセサのFDより小さいRDを持つネイバー。「FD > RD」を満たす経路はループが発生しないことが保証されており、フィージブルサクセサはバックアップルートや不等コストロードバランシングのネクストホップとして利用される。

### EIGRPにおける最適経路の選出

1. 宛先ネットワークまでの全ネイバ経由のFD(Feasible Distance) を算出

2. 最も小さいFDを持つネイバをサクセサとする

3. サクセサのFDと各ネイバの RD (Reported Distance) を比較し、FD > RD となるネイバをフィージブルサクセサとする

4. サクセサをネクストホップとする経路をルーティングテーブルに載せる

### 構築

![pic3](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1122/1122_3.png)
~~~
RT1(config)#router eigrp 10
RT1(config-router)#network 192.168.1.0 0.0.0.255
RT1(config-router)#network 192.168.2.0 0.0.0.255
~~~
同様の要領で全てのルータに設定する。

#### 確認

~~~
RT4#sh ip eigrp topology
EIGRP-IPv4 Topology Table for AS(10)/ID(192.168.4.1)

P 192.168.3.0/24, 1 successors, FD is 3072
        via 192.168.4.2 (3072/2816), GigabitEthernet0/0
P 192.168.2.0/24, 1 successors, FD is 2816
        via Connected, GigabitEthernet0/1
P 192.168.1.0/24, 1 successors, FD is 3072
        via 192.168.2.1 (3072/2816), GigabitEthernet0/1
P 192.168.4.0/24, 1 successors, FD is 2816
        via Connected, GigabitEthernet0/0
~~~

~~~
RT4#sh ip route eigrp

D     192.168.1.0/24 [90/3072] via 192.168.2.1, 00:00:49, GigabitEthernet0/1
D     192.168.3.0/24 [90/3072] via 192.168.4.2, 00:00:49, GigabitEthernet0/0
~~~
全てのネットワークを認識しており、最適な経路(この場合単に近道)を選定できている。