---
title: "GKEからCloud SQLにアクセスする"
date: 2021-07-16T15:21:36+09:00
draft: false
tags: ["cloud-sql", "kubernetes", "google-kubernetes-engine", "iam"]
---

## Kubernetesクラスタを更新する

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

## Kubernetes Secretの作成

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

## Kubernetes Service Account (KSA) の作成

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

## Google Cloud Service Account (GSA) の作成

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

## GSAとKSAの紐付け

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

## KSAにアノテーションを追加する

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

## 参考

* https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
* https://cloud.google.com/sql/docs/sqlserver/connect-kubernetes-engine
* https://y-ohgi.blog/entry/2020/08/19/GKE%E3%81%AEWorkloadIdentity%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%A6%E3%83%A2%E3%83%80%E3%83%B3%E3%81%AA%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%88%B6%E5%BE%A1%E3%82%92%E5%AE%9F%E7%8F%BE%E3%81%99%E3%82%8B
* https://stackoverflow.com/questions/65948131/unable-to-bind-google-service-account-to-kubernetes-service-account
