---
title: "PrometheusOperatorを使ってGKE上に監視基盤を作ってみた"
date: 2021-09-23T09:19:29-04:00
slug: ""
description: "Prometheus, Grafanaを使って監視基盤を構築した際の話です。"
keywords: ["Prometheus", "Grafana", "GCP", "kubernetes"]
draft: false
tags: ["observability"]
math: false
toc: true
---


# 本記事でやること
GKEクラスターにPromehteusをデプロイし、アプリのPodを監視する基盤を構築する。
今回は公開されているhelm chart ([prometheus-community/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack))を利用してPrometheus OperatorやAlertMangager等の周辺ツールの導入を行う。

# Prometheusとは
監視用のソフトウェアとして開発された。Pull型の時系列データベースで、Prometheus内部のServiceDiscovery機能を使い、監視対象のターゲットを自動で判別することができる。
また、PromQLというPrometheus専用のクエリ言語を使って必要なメトリクスやデータをターゲットから取得(Pull)することができる。  

図:Prometheusのアーキテクチャ
![archi](https://knowledge.sakura.ad.jp/images/2021/03/Prometheus-arch-scaled.jpg)

監視対象のターゲットはPrometheusServerから要求があった際に、Promthetheusの求める形式でメトリクスを返す必要がある。この要求に対してメトリクス情報を返すコンポーネントをExporterという。
アプリケーションやDBはこのExporterをサーバ内に導入することでPrometheusからのメトリクス収集の要求に対応することができる。
DBなどの標準的なな外部サービスは公式でExporterが提供されている。


# Prometheus Operatorとは
[https://github.com/prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)

Prometheus OperaotrはPrometheusの監視対象との接続や設定の管理を行う。
k8sを触っているとOperatorという言葉に慣れて容易に理解ができるが、そうでないと感覚的に理解するのが難しい。(慣れる)
Operatorを利用する際に得られるメリットは下記。

* **Kubernetes Custom Resources**: CRD(Custom Resource Definition)で定義することができるため、ymlでPrometheusやAlertMangager, その他の関連リソースを管理することができる
* **Simplified Deployment Configuration**: Prometheusのコンポーネントのデプロイをk8sのリソースで管理することができる(Prometheusのバージョンなど)
* **Prometheus Targget Configuration**: 監視対象との接続に必要な設定を自動生成することができる。複雑な設定はOperatorによって隠蔽されており、利用者はk8sリソースのlabelを指定するだけでよい。

## 全体構成

Prometheus Operatorを利用した場合の構成図
![S](https://www.scsk.jp/sp/sysdig/blog/20200112c.png)
https://www.scsk.jp/sp/sysdig/blog/prometheus/prometheuskubernetes-_prometheus_operator_3.html

ターゲットの監視を行う上で必要になるのがServiceMonitorやPodMonitorであり、実際に接続する際にはPrometheusとターゲットの間で双方を接続する役割を担う。

図:PrometheusOperatorを利用した場合のアーキテクチャ
![S](https://www.scsk.jp/sp/sysdig/blog/20200112c.png)

余談だが、ふとPodMonitorとServiceMonitorってどう使い分けるべき？違いは何？という疑問が生じた。

公式のIssueによると、下記の解釈らしい。
> I think ServiceMonitor should be the default choice unless you have some reason to use PodMonitor. For example, you may want to scrape a set of pods which all have a certain label which is not consistent between different services.

ServiceMonitorを使えばServiceにぶら下がったPod群すべてを監視対象のターゲットにすることができる。PodMonitorを使う場合というのは、異なるサービス間で特定のラベルがついたPodを監視したい(またはServiceを持たないPod)という場合。
そしてPodMonitorを使いたいモチベーションが特別ないならServiceMonitorを使っておけ。とのこと。


[https://github.com/prometheus-operator/prometheus-operator/issues/3119](https://github.com/prometheus-operator/prometheus-operator/issues/3119)




# 導入手順

## Helmインストール

```sh
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
  
導入できたことを確認。

```sh
$ helm version
version.BuildInfo{Version:"v3.6.2", GitCommit:"ee407bdf364942bcb8e8c665f82e15aa28009b71", GitTreeState:"clean", GoVersion:"go1.16.5"}
```

## Prometheus Operatorのインストール

### 対象のhelm chartを追加

```sh
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
```

### 監視用の名前空間を作成

```bash
$ kubectl create namespace monitoring
```
  
### Prometheus Operatorのインストール

```sh
$ helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

### (参考) Chartの構造

```yaml
<チャート名ディレクトリ>/
Chart.yaml          # チャートの概要が記述されたYAMLファイル(ファイル名は予約)
LICENSE             # オプション:このチャートのライセンス情報
README.md           # オプション: チャートの説明
requirements.yaml   # オプション: このチャートが利用する(依存する)他のチャートの一覧。(ファイル名は予約)
values.yaml         # このチャートのデフォルト設定値が定義されたYAMLファイル。ファイル名は予約)
charts/             # このチャートが依存するチャートをコピー配置するディレクトリ(ディレクトリ名は予約)
templates/          # チャートの本体ともいえる、KubernetesオブジェクトのリソースYAMLのテンプレート群を配置するディレクトリ(ディレクトリ名は予約)
templates/NOTES.txt # オプション:利用方法を生成するテキストファイル
tests/              # オプション:helm testで実行されるテスト用YAMLを配置するディレクトリ(ディレクトリ名は予約)
```
   
### ここまででインストールしたものの確認
 
```sh
$ kubectl get pods -n monitoring
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-stack-kube-prom-alertmanager-0   2/2     Running   0          5m12s
prometheus-prometheus-stack-kube-prom-prometheus-0       2/2     Running   1          5m11s
prometheus-stack-grafana-cb6b6d849-lndcc                 2/2     Running   0          5m21s
prometheus-stack-kube-prom-operator-cfbf6c47f-dwjnm      1/1     Running   0          5m21s
prometheus-stack-kube-state-metrics-7c5c84f64d-swq6w     1/1     Running   0          5m21s
prometheus-stack-prometheus-node-exporter-swp88          1/1     Running   0          5m21s
```

* prometheus-stack-kube-prom-operator
  * Prometheus Operator 本体
* prometheus-prometheus-stack-kube-prom-prometheus
  * Prometheus本体
* alertmanager-prometheus-stack-kube-prom-alertmanager
  * Alertmanager本体
* prometheus-stack-grafana
  * Prometheus のメトリクス可視化用の Grafana 本体
* prometheus-stack-kube-state-metrics
  * Kubernetesのオブジェクト毎のメトリクス出力用exporter
* prometheus-stack-prometheus-node-exporter
  * Node毎のメトリクス出力用exporter

利用したhelmチャートは``prometheus-community/kube-prometheus-stack``だが、デフォルトでAlertManagerやgrafana, node Exporter等も含まれていることがわかる。
このあたりはPrometheusとセットで構築することも多く、初期構築のコストも高いため、Helmでインストール、管理できるのはかなり助かる。
Helmで導入されたPrometheusにもそれなりの数のアラート設定があるため、改修したければカスタムリソースを別途定義して設定を上書きすることができれば、CDフローとしても取り込めるだろう。

## 監視対象のアプリケーションをデプロイ
アプリはデプロイされてる前提だが、もしデプロイしていなければデプロイしておく。

## ServiceMonitorの作成
アプリのServiceを監視するServiceMonitorをカスタムリソースを使って定義する。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-service-monitor
  namespace: monitoring  
  labels: 
    release: prometheus-stack # (1)Prometheusと紐づく
spec:
  selector:
    matchLabels:
      app: sample-app  # (2)アプリのServiceと紐づける
  endpoints: 
    port: "8000"
``` 

アプリ側で用意したServiceは下記。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  labels: 
    app: sample-app # (2)
spec:
  type: ClusterIP
  selector:
    app: sample-app
  ports:
    - name: sample-app
      protocol: TCP
      port: 5000
      targetPort: 5000
```

Prometheus⇔ServiceMonitorの接続は、Prometheusのymlを取得することで確認できる。
helmチャートでインストールしたPrometheusのymlを取得して確認する。

```yaml
    ... 
    spec:
      alerting:
        alertmanagers:
        - apiVersion: v2
          name: prometheus-stack-kube-prom-alertmanager
          namespace: monitoring
          pathPrefix: /
          port: web
      ...
      serviceAccountName: prometheus-stack-kube-prom-prometheus
      serviceMonitorNamespaceSelector: {}
      serviceMonitorSelector: #監視対象のServiceMonitorを選択
        matchLabels:
          release: prometheus-stack # (1)
      shards: 1
      version: v2.27.1

```
PrometheusはserviceMonitorSelectorの定義によって監視するServiceMonitorを判別している。
そのため、今回追加したServiceMonitorも同様のラベルを持つように設定している。


# 動作確認
GKE上で下記コマンドを実行し、Serviceへのポートをフォワードする。

```sh
$ gcloud container clusters get-credentials sample-app --zone asia-northeast1 --project <PROJECT_NAME> \
 && kubectl port-forward --namespace monitoring $(kubectl get pod --namespace monitoring --selector="app.kubernetes.io/name=prometheus,prometheus=prometheus-stack-kube-prom-prometheus" --output jsonpath='{.items[0].metadata.name}') 8080:9090
```

Prometheusの画面が立ち上がるので、Targetを選択して対象のコンポーネントがUP状態になっていることを確認する。

## さいごに


2021/9時点ではPrometheus Operatorはbetaリリースだが、遠くない未来にGA(General Availability) を迎えるはずなので注目。w