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

![ここにディレクトリ構造](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1122/1122_1.png)

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

![完成画像](https://raw.githubusercontent.com/220TI/Training-Reports/refs/heads/master/1122/1122_2.png)


## EIGRP

### シリアル接続
