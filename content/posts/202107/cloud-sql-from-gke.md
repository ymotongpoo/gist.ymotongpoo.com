---
title: "GKEからCloud SQLにアクセスする"
date: 2021-07-16T15:21:36+09:00
draft: false
tags: ["cloud-sql", "kubernetes", "google-kubernetes-engine", "iam"]
---

## KubernetesとGoogle Cloudの設定を行う
### Kubernetesクラスタを更新する

Workload Identityを有効にしていなかったらまずその設定を行う。

```console
gcloud container clusters update <cluster-name> \
--workload-pool=<project_id>.svc.id.goog
```

自分の場合の例

```console
gcloud container clusters update trace-sample --workload-pool=trace-microservices-sample.svc.id.goog
Updating trace-sample...done.
Updated [https://container.googleapis.com/v1/projects/trace-microservices-sample/zones/us-west1-b/clusters/trace-sample].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-west1-b/trace-sample?project=trace-microservices-sample
```

### Kubernetesのノードプールを変更する

新しいノードプールを作るか既存のノードプールを `GKE_METADATA` を使うように変更する。自分は既存のノードプールを変更した。

```console
gcloud container node-pools update <nodepool_name> \
  --cluster=<cluster_name> \
  --workload-metadata=GKE_METADATA
```

自分の場合はこういう感じ

```console
$ gcloud container node-pools update default-pool \
--cluster=trace-sample \
--workload-metadata=GKE_METADATA
Updating node pool default-pool... Updating default-pool, done with 0 out of 2 nodes (0.0%): 1 being process
ed...⠹
Updating node pool default-pool... Updating default-pool, done with 1 out of 2 nodes (50.0%): 1 being proces
sed, 1 succeeded...done.
Updated [https://container.googleapis.com/v1/projects/trace-microservices-sample/zones/us-west1-b/clusters/trace-sample/nodePools/default-pool].
```

### Kubernetes Secretの作成

```console
$ kubectl create secret generic cloud-sql-secret \
--from-literal=username=<Cloud SQL username> \
--from-literal=password=<Cloud SQL password> \
--from-literal=database=<Cloud SQL database>
```

自分の場合の例

```console
$ kubectl create secret generic cloud-sql-secret \
--from-literal=username=postgres \
--from-literal=password=XXXXXXXXXXXXXXXX \
--from-literal=database=xxxxxxxxxxxxxxxx
secret/cloud-sql-secret created
```

これを作るとGKEのConfigurationsでSecretが作成されたことが確認できる。

{{< figure src="/image/202107/20210716-1.png" >}}

### Kubernetes Service Account (KSA) の作成

```console
$ kubectl create namespace <namespace_name>
$ kubectl create serviceaccount <ksa_name> --namespace <namespace_name>
```

自分の場合は横着してdefaultネームスペースを使ってしまった

```console
$ kubectl create serviceaccount cloud-sql-ksa --namespace default
serviceaccount/cloud-sql-ksa created

$ kubectl get serviceaccount --namespace default
NAME            SECRETS   AGE
cloud-sql-ksa   1         21s
default         1         10d
```

### Google Cloud Service Account (GSA) の作成

まず新規Google Cloud Service Accountを作成する。

```console
$ gcloud iam service-accounts create <gsa_name>
```

つづいて作ったGSAに対してCloud SQLのアクセス権限を付与する

```console
$ gcloud projects add-iam-policy-binding <project_id> \
--member serviceAccount:<gsa_name>@<project_id>.iam.gserviceaccount.com \
--role roles/cloudsql.client
```

自分の例

```console
$ gcloud iam service-accounts create cloud-sql-gsa
Created service account [cloud-sql-gsa].

$ gcloud projects add-iam-policy-binding trace-microservices-sample \
--member serviceAccount:cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com \
--role roles/cloudsql.client
Updated IAM policy for project [trace-microservices-sample].
bindings:
- members:
  ...(略)...
etag: BwXHNz0zmqE=
version: 1
```

### GSAとKSAの紐付け

```console
gcloud iam service-accounts add-iam-policy-binding \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:<project_id>.svc.id.goog[<kubernetes_namespace>/<ksa_name>]" \
<gsa_name>@<project_id>.iam.gserviceaccount.com
```

自分の場合の例

```console
$ gcloud iam service-accounts add-iam-policy-binding \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]" \
cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com
Updated IAM policy for serviceAccount [cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwXHN6fhkC0=
version: 1
```

### KSAにアノテーションを追加する

```console
kubectl annotate serviceaccount \
   <ksa_name> \
   iam.gke.io/gcp-service-account=<ksa_name>@<project_id>.iam.gserviceaccount.com
```

自分の場合

```console
kubectl annotate serviceaccount \
   cloud-sql-ksa \
   iam.gke.io/gcp-service-account=cloud-sql-ksa@trace-microservices-sample.iam.gserviceaccount.com

serviceaccount/cloud-sql-ksa annotated
```

## トラブル

### Cloud SQL Proxy が 403で起動しない

KSAとGSAの紐付けを行っていて、GSAにちゃんとCloud SQLのクライアントのロールを与えているのに接続できなかった。

```console
...(略)...
Waiting for deployments to stabilize...
 - deployment/appservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/clientservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/appservice: container cloud-sql-proxy terminated with exit code 1
    - pod/appservice-64ddff97b9-bdq7w: container cloud-sql-proxy terminated with exit code 1
      > [appservice-64ddff97b9-bdq7w cloud-sql-proxy] 2021/07/16 07:41:10 current FDs rlimit set to 1048576, wanted limit is 8500. Nothing to do here.
      > [appservice-64ddff97b9-bdq7w cloud-sql-proxy] 2021/07/16 07:41:13 errors parsing config:
      > [appservice-64ddff97b9-bdq7w cloud-sql-proxy]   googleapi: Error 403: The client is not authorized to make this request., notAuthorized
 - deployment/appservice failed. Error: container cloud-sql-proxy terminated with exit code 1.
...(略)...
```

よく確認するとKubernetesのマニフェストでサービスアカウント名をちゃんと指定していなかった。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appservice
spec:
  selector:
    matchLabels:
      app: appservice
  template:
    metadata:
      labels:
        app: appservice
    spec:
      serviceAccountName: cloud-sql-ksa # <-- ここにちゃんとKSAを指定
      terminationGracePeriodSeconds: 5
      containers:
      - name: appservice
        image: appservice
        ports:
        - containerPort: 5000
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: cloud-sql-secret
              key: username
              ...(略)...
```

### GCEメタデータが取得できない

上の設定を行ったら今度はまた別の問題が発生。

```console
Waiting for deployments to stabilize...
 - deployment/appservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/clientservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/appservice: container cloud-sql-proxy terminated with exit code 1
    - pod/appservice-77748675fc-dwhvk: container cloud-sql-proxy terminated with exit code 1
      > [appservice-77748675fc-dwhvk cloud-sql-proxy] 2021/07/16 10:38:38 current FDs rlimit set to 1048576, wanted limit is 8500. Nothing to do here.
      > [appservice-77748675fc-dwhvk cloud-sql-proxy] 2021/07/16 10:38:42 errors parsing config:
      > [appservice-77748675fc-dwhvk cloud-sql-proxy]   Get "https://sqladmin.googleapis.com/sql/v1beta4/projects/trace-microservices-sample/instances/us-west1~shoe-hat-inventory?alt=json&prettyPrint=false": metadata: GCE metadata "instance/service-accounts/default/token?scopes=https%!A(MISSING)%!F(MISSING)%!F(MISSING)www.googleapis.com%!F(MISSING)auth%!F(MISSING)sqlservice.admin" not defined
 - deployment/appservice failed. Error: container cloud-sql-proxy terminated with exit code 1.
```

これはGKEのアドオンとしてConfig Connectorというものを追加してあげないといけないらしい。

* https://cloud.google.com/config-connector/docs/how-to/install-upgrade-uninstall

```console
gcloud container clusters update trace-sample --update-addons ConfigConnector=ENABLED
Updating trace-sample...done.
Updated [https://container.googleapis.com/v1/projects/trace-microservices-sample/zones/us-west1-b/clusters/trace-sample].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-west1-b/trace-sample?project=trace-microservices-sample
```

それでも同様のエラーが発生するのでこれを試してみた。

* https://cloud.google.com/sql/docs/postgres/admin-api?hl=ja

```console
$ gcloud services enable sqladmin.googleapis.com
```

しかしこれでもだめ、よくよくドキュメントを見てみるとGKEからCloud SQLに接続する場合にはVPCネイティブじゃないとだめらしい。しかしこのクラスタはなっていない。

{{< figure src="/image/202107/20210716-3.png" >}}

これは[あとから変更できない](https://cloud.google.com/kubernetes-engine/docs/how-to/routes-based-cluster?hl=en#restrictions)のでクラスターを再構築するしか無い。
### リソースが足りない

問題がConfig Connectorだと思ってデプロイしてみたらいろんなサービスが追加で入れられてリソースが足りなくなった。

{{< figure src="/image/202107/20210716-2.png" >}}


最初クラスターを作り直そうかと思ったけど、せっかくなのでオートスケーリングでやってみる。

```
gcloud container clusters update trace-sample --enable-autoscaling --max-nodes 10
```

長くなったのでクラスターを再構築するところから始めます。
## 参考

* https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
* https://cloud.google.com/sql/docs/sqlserver/connect-kubernetes-engine
* https://y-ohgi.blog/entry/2020/08/19/GKE%E3%81%AEWorkloadIdentity%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%A6%E3%83%A2%E3%83%80%E3%83%B3%E3%81%AA%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%88%B6%E5%BE%A1%E3%82%92%E5%AE%9F%E7%8F%BE%E3%81%99%E3%82%8B
* https://stackoverflow.com/questions/65948131/unable-to-bind-google-service-account-to-kubernetes-service-account
* https://engineer.recruit-lifestyle.co.jp/techblog/2019-12-23-workload-identity/
* https://cloud.google.com/config-connector/docs/troubleshooting#metadata_not_defined
* https://cloud.google.com/config-connector/docs/how-to/install-upgrade-uninstall