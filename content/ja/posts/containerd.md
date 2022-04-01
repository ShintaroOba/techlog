---
type: Archive
title: Dockerじゃないコンテナの話
date: 2021-08-21
description: Containerdというとあるコンテナランタイムの話
titleWrap: wrap
tags: 
- クラウドネイティブ

image: images/summary/containerd.png
---


# なんでDockerじゃないコンテナの話をするの？
2020年12月、Kubernetesのマイナーリリース(v1.20)にてこんなアップデートがありました。

> Docker support in the kubelet is now deprecated and will be removed in a future release. The kubelet uses a module called "dockershim" which implements CRI support for Docker and it has seen maintenance issues in the Kubernetes community. We encourage you to evaluate moving to a container runtime that is a full-fledged implementation of CRI (v1alpha1 or v1 compliant) as they become available. 

これはKubernetesでは内部のコンテナランタイムにDockerを利用していたが、v1.20のアップデートでDocker(あくまでコンテナランタイムとしての利用)が非推奨になったことを意味していて、2021年後半リリース予定のKubernetes 1.22で削除される予定みたいです。これを受けて、Docker以外のコンテナランタイムについて触れてみることで、Kubernetesを舞台裏で支えているコンテナランタイムとは何ぞや？というところにスポットライトを当ててみたいと思います。

## こうなった(せざるを得なかった)理由
Kubernetesはコンテナオーケストレーションのプラットフォームなのでもちろん内部にコンテナランタイムを必要とします。しかしKubernetesとしては、ランタイムはDockerのみに依存するのではなく、選択肢の一つとしてあるべき。と考えられていたのです。
もう一つの理由ですが、こっちはKubernetes内部の話です。
Kubernetesのノード内でPodを操作する際にはkubeletと呼ばれるエージェントを使ってコンテナと通信を行います。ここではCRI(Container Runtime Interface)と呼ばれる規定に則ってコンテナとの通信が行われます。

![](https://f.v1.n0.cdn.getcloudapp.com/items/0I3X2U0S0W3r1D1z2O0Q/Image%202016-12-19%20at%2017.13.16.png)
  

Introducing Container Runtime Interface (CRI) in Kubernetes  
{{< blogcard url="https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/" >}}


実はDockerはこのCRIに則っておらずKubernetesの開発者がdockershimというブリッジを用意し、このdockershimによってkubeletとコンテナの通信を可能にしていました。

それはそれで問題ないのですが、dockershimのメンテに高いコストがかかっていたようです。多様なコンテナランタイムの選択肢を提供したいKubernetesにとっては、dockerにしか利用されないdockershimをメンテし続けるのも骨が折れる作業だろうなぁ。。

## Dockerに代わるコンテナランタイム
コンテナランタイムについて話してきましたが、実はコンテナランタイムには高レベルランタイムと低レベルランタイムという言葉があります。ついでにこれらについても軽く触れます。

### 高レベルランタイム
DockerやKubernetesからの命令を受けて、低レベルコンテナランタイムに渡す役割を担っており、本記事で扱う、Dockerの代替となるコンテナランタイムもこの高レベルランタイムに属します。
高レベルランタイムはホストOS上にdaemonプロセスとして常駐し、CRIによりKubernetesやDockerと対話をします。ローカルでのイメージの管理などもこの高レベルランタイムが行ってます。
先ほど述べた通り、高レベルランタイムにおいてはコンテナを作成する機能や実行する機能は持たず、クライアントからの「docker run」などの命令を低レベルランタイムに引き渡す役割を担います。

### 低レベルランタイム
低レベルランタイムの実体はdaemonではなくバイナリで、高レベルランタイムによって呼び出され実行されます。
コンテナ作成時には「OCI(Open Container Initiative)」と呼ばれる、いわゆるコンテナの標準仕様に基づいてコンテナが作成されます。もう少し厳密にいうと、低レベルランタイムがこの仕様を把握できるようにconfig.jsonというjsonファイルに標準仕様が記載されているため、低レベルランタイムはこれを読み取ってその通りにコンテナの作成を行います。

### 代表的な高レベルランタイム
#### Containerd

![](https://pocketstudio.net/wp-content/uploads/2017/10/containerd-horizontal-color.png)

元々はDocker社が開発していましたが、2017年にCloud Native Foundation(CNCF)に寄贈され、その後2019年2月末にCNCFを卒業しています。  

{{< blogcard url="https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/" >}}

実は、containerdはずっと前からDocker内部で使われていました。
「containerd v1.0」ではCRIに対応ができておらず、Docker内部に隠れたままでしたが、「containerd v1.1」ではついにCRIに対応され、晴れてKubernetesから直接containerdを操作できるようになりました。
なのでKubernetesがコンテナの操作をする際はdockerもdockershimを経由する必要もなく、直接containerdを使ってコンテナを操作することができるようになりました。
「Docker非推奨？！え、、マイナーバージョンアップにしてはえぐいことするな。。」という風にとらえられがちですが、内部的にはDockerからcontainerdを取り出しただけ。そんなに大したことをしていないのです。


#### CRI-O

![](https://vmblog.com/images/CRI-O-Logo.png)

CRI-Oとは、Kubernetes Incubator Projectとして開発され、OCI準拠の軽量コンテナランタイムです。2017年10月にv1.0が正式にリリースされています。
本記事では特にこれ以上触れませんが、containerd同様CRIに準拠していることからkubeletと直接連携することができます。

## containerdを触ってみる
containerdを導入する手順とctrコマンドを使った簡単なイメージ・コンテナ操作について触れる。

### ctrコマンド
コンテナランタイムであるcontainerdと対話するためのCLIコマンド。
主に実行中のコンテナの作成および管理、コンテナイメージの管理(レジストリへのPush、Pull)などが主な役割。
containerdとの通信はgRPCによって行われる。

### containerdインストール

* https://github.com/containerd/containerd/releases
から任意のバージョンのバイナリファイルをインストール

* インストール後、ctrコマンドを実行し、下記のような出力を確認。

```bash
$ ctr

NAME:
   ctr - 
        __
  _____/ /______
 / ___/ __/ ___/
/ /__/ /_/ /
\___/\__/_/

containerd CLI

USAGE:
   ctr [global options] command [command options] [arguments...]

VERSION:
   1.4.3

DESCRIPTION:
ctr is an unsupported debug and administrative client for interacting
with the containerd daemon. Because it is unsupported, the commands,
options, and operations are not guaranteed to be backward compatible or
stable from release to release of the containerd project.

COMMANDS:
   plugins, plugin            provides information about containerd plugins
   version                    print the client and server versions
   containers, c, container   manage containers
   content                    manage content
   events, event              display containerd events
   images, image, i           manage images
   leases                     manage leases
   namespaces, namespace, ns  manage namespaces
   pprof                      provide golang pprof outputs for containerd
   run                        run a container
   snapshots, snapshot        manage snapshots
   tasks, t, task             manage tasks
   install                    install a new package
   oci                        OCI tools
   shim                       interact with a shim directly
   help, h                    Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug                      enable debug output in logs
   --address value, -a value    address for containerd's GRPC server (default: "/run/containerd/containerd.sock") [$CONTAINERD_ADDRESS]
   --timeout value              total timeout for ctr commands (default: 0s)
   --connect-timeout value      timeout for connecting to containerd (default: 0s)
   --namespace value, -n value  namespace to use with commands (default: "default") [$CONTAINERD_NAMESPACE]
   --help, -h                   show help
   --version, -v                print the version

```

containerdのdaemonのオプションを/etc/containerd/config.tomlで設定する。
今回は、デフォルト設定を反映させるため、下記のコマンドを実行する。

```bash
$ containerd config default > /etc/containerd/config.toml
```

### daemonの設定ファイルの編集

containerdのdaemonのオプションを``/etc/containerd/config.toml``で設定する。
今回は、デフォルト設定を反映させるため、下記のコマンドを実行する。

```bash
$ containerd config default > /etc/containerd/config.toml
```

下記のようにconfig.tomlが変更されていることを確認する。

```bash
$ cat /etc/containerd/config.toml 

version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
plugin_dir = ""
disabled_plugins = []
required_plugins = []
oom_score = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  tcp_address = ""
...
```
## ctrコマンドを使ったイメージ/コンテナ操作

### dockerhubからイメージのpull

```bash
$ sudo ctr image pull docker.io/library/hello-world:latest

docker.io/library/hello-world:latest:                                             resolved       |++++++++++++++++++++++++++++++++++++++|                                                               index-sha256:7e02330c713f93b1d3e4c5003350d0dbe215ca269dd1d84a4abc577908344b30:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042: done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:0e03bdcc26d7a9a57ef3b6f1bf1a210cff6239bff7c8cac72435984032851689:    done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b:   done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 5.2 s                                                                    total:  6.5 Ki (1.3 KiB/s)                                       
unpacking linux/amd64 sha256:7e02330c713f93b1d3e4c5003350d0dbe215ca269dd1d84a4abc577908344b30...
done
```

### image一覧の取得

```bash
$ sudo ctr image ls

REF                                  TYPE                                                      DIGEST                                                                  SIZE     PLATFORMS                                                                                                             LABELS                                                                               -      
docker.io/library/hello-world:latest application/vnd.docker.distribution.manifest.list.v2+json sha256:7e02330c713f93b1d3e4c5003350d0dbe215ca269dd1d84a4abc577908344b30 6.5 KiB  linux/386,linux/amd64,linux/arm/v5,linux/arm/v7,linux/arm64/v8,linux/mips64le,linux/ppc64le,linux/s390x,windows/amd64 -    
```

### コンテナの作成
ここではコンテナIDが'demo'であるコンテナを作成する。

```bash
$ sudo ctr container create docker.io/library/hello-world:latest demo
```

### コンテナ一覧の取得

```bash
$ sudo ctr container list

CONTAINER    IMAGE                                   RUNTIME                  
demo         docker.io/library/hello-world:latest    io.containerd.runc.v2    

```

### イメージの削除

```bash
$ sudo ctr image remove docker.io/library/hello-world:latest
```

### コンテナの削除

```bash
$ sudo ctr container remove demo
```

### コンテナの実行
```
$ sudo ctr run docker.io/library/hello-world:latest demo
```

### dockerでビルドしたDockerImageの実行

#### DockerImageのエクスポート(.tar)
-oの後は任意のディレクトリ、<IMAGE_NAME>は対象のDockerImageを指定。
```
$ docker save -o ~/images/<IMAGE_NAME>.tar <IMAGE_NAME>
```

#### containerdでのインポート
```
$ sudo ctr image import <IMAGE_NAME>.tar

unpacking <IMAGE_NAME>:latest (sha256:ef4acfd85c856ea020328959ff3cac23f37fa639b7edfb1691619d9bfe1e06c7)...done
```

#### run
```
$ sudo ctr run docker.io/library/hello-java-app:latest v1
```

dockerと同じ感覚で使えますね。
