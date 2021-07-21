---
title: "GKEクラスターをVPCネイティブで作成する"
date: 2021-07-20T10:00:33+09:00
draft: false
tags: ["google-kubernetes-engine", "vpc", "kubernetes", "network"]
---

## 作業ログ

作成されたクラスターの確認。

```console
$ gcloud container clusters list
NAME          LOCATION    MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
trace-sample  us-west1-a  1.19.9-gke.1900  34.83.239.113  e2-medium     1.19.9-gke.1900  3          RUNNING
```

ノードプールの更新。

```console
$ gcloud container node-pools update default-pool --cluster=trace-sample --workload-metadata=GKE_METADATA --zone=us-west1-a
Updating node pool default-pool...done.
Updated [https://container.googleapis.com/v1/projects/trace-microservices-sample/zones/us-west1-a/clusters/trace-sample/nodePools/default-pool].
```

次はKubernetes Secretの作成。

```console
$ kubectl create secret generic cloud-sql-secret \
> --from-literal=username=postgres \
> --from-literal=password=xxxxxxxxxxxxxxxxx \
> --from-literal=database=shoe_database
error: failed to create secret Post "http://localhost:8080/api/v1/namespaces/default/secrets?fieldManager=kubectl-create": dial tcp [::1]:8080: connect: connection refused
```

前と同じ名前でクラスター作ったけど `kubectl` がアクセスできない。

```console
$ gcloud container clusters get-credentials trace-sample
Fetching cluster endpoint and auth data.
kubeconfig entry generated for trace-sample.
```

KSAを作成

```console
$ kubectl create serviceaccount cloud-sql-ksa --namespace default
serviceaccount/cloud-sql-ksa created

$ kubectl get serviceaccount --namespace default
NAME            SECRETS   AGE
cloud-sql-ksa   1         12s
default         1         46m
```

GSAは前に作ってあるので、KSAとGSAの紐付けを行う。

```console
$ gcloud iam service-accounts add-iam-policy-binding \
> --role roles/iam.workloadIdentityUser \
> --member "serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]" \
> cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com
Updated IAM policy for serviceAccount [cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwXHhKG7SRo=
version: 1
```

KSAにアノテーションを追加する

```console
$ kubectl annotate serviceaccount cloud-sql-ksa iam.gke.io/gcp-service-account=cloud-sql-ksa@trace-microservices-sample.iam.gserviceaccount.com
serviceaccount/cloud-sql-ksa annotated
```

これで実行してみる

```
$ skaffold dev
...(略)...
Waiting for deployments to stabilize...
 - deployment/appservice: 0/3 nodes are available: 3 Insufficient cpu.
    - pod/appservice-5bfd679945-lb924: 0/3 nodes are available: 3 Insufficient cpu.
 - deployment/clientservice: creating container clientservice
    - pod/clientservice-645fcb6957-4rqrk: creating container clientservice
 - deployment/appservice: 0/3 nodes are available: 3 Insufficient cpu.
    - pod/appservice-5bfd679945-lb924: 0/3 nodes are available: 3 Insufficient cpu.
 - deployment/appservice failed. Error: 0/3 nodes are available: 3 Insufficient cpu..
Cleaning up...
...(略)...
```


```console
$ kubectl apply -f test.yaml
pod/workload-identity-test created
~/corp/k8stest ☁️ trace-sample

$ kubectl exec -it workload-identity-test --namespace default -- /bin/bash
error: unable to upgrade connection: container not found ("workload-identity-test")

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
workload-identity-test   1/1     Running   0          41s

$ kubectl exec -it workload-identity-test --namespace default -- /bin/bash
root@workload-identity-test:/# gcloud auth list
                         Credentialed Accounts
ACTIVE  ACCOUNT
*       cloud-sql-ksa@trace-microservices-sample.iam.gserviceaccount.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```

Kubernetesクラスタに対してサービスアカウントのapplyをしてなかったことに気がつく。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-sql-ksa
```

```console
$ kubectl apply -f service-account.yaml
serviceaccount/cloud-sql-ksa created
```

あらためて実行。

```console
$ skaffold dev
...(略)...
 - deployment/appservice: container cloud-sql-proxy terminated with exit code 1
    - pod/appservice-69d6447d6c-dsmls: container cloud-sql-proxy terminated with exit code 1
      > [appservice-69d6447d6c-dsmls cloud-sql-proxy] 2021/07/16 07:48:01 current FDs rlimit set to 1048576, wanted limit is 8500. Nothing to do here.
      > [appservice-69d6447d6c-dsmls cloud-sql-proxy] 2021/07/16 07:48:02 errors parsing config:
      > [appservice-69d6447d6c-dsmls cloud-sql-proxy]   googleapi: Error 403: The client is not authorized to make this request., notAuthorized
 - deployment/appservice failed. Error: container cloud-sql-proxy terminated with exit code 1.
...(略)...
```

エラーが変わった！！！KSAとGSAの紐付けはちゃんと行ってるので、もう一度アノテーションをしてみる。

```console
$ gcloud iam service-accounts add-iam-policy-binding \
> --role roles/iam.workloadIdentityUser \
> --member "serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]" \
> cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com
Updated IAM policy for serviceAccount [cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:trace-microservices-sample.svc.id.goog[default/cloud-sql-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwXHiYx_cVc=
version: 1

$ kubectl annotate serviceaccount cloud-sql-ksa iam.gke.io/gcp-service-account=cloud-sql-gsa@trace-microservices-sample.iam.gserviceaccount.com
serviceaccount/cloud-sql-ksa annotated
```

再度実行

```console
$ skaffold dev
...(略)...
Waiting for deployments to stabilize...
 - deployment/appservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/clientservice: waiting for rollout to finish: 0 of 1 updated replicas are available...
 - deployment/appservice is ready. [1/2 deployment(s) still pending]
 - deployment/clientservice is ready.
Deployments stabilized in 18.977 seconds
Press Ctrl+C to exit
Watching for changes...
[clientservice]  * Serving Flask app 'main' (lazy loading)
[clientservice]  * Environment: production
[clientservice]    WARNING: This is a development server. Do not use it in a production deployment.
[clientservice]    Use a production WSGI server instead.
[clientservice]  * Debug mode: off
[clientservice]  * Running on all addresses.
[clientservice]    WARNING: This is a development server. Do not use it in a production deployment.
[clientservice]  * Running on http://10.0.0.8:8080/ (Press CTRL+C to quit)
[clientservice] 10.0.0.1 - - [20/Jul/2021 09:07:45] "GET /healthz HTTP/1.1" 200 -
[appservice] 10.0.2.1 - - [20/Jul/2021 09:07:46] "GET /healthz HTTP/1.1" 200 -
[appservice] 10.0.0.11 - - [20/Jul/2021 09:07:55] "GET / HTTP/1.1" 200 -
[clientservice] [{"shoe_id": 0, "name": "lemon-furud-1901", "last_arrival": "2011-08-02T01:58:21", "stock": 192}, {"shoe_id": 1, "name": "orange-deneb-2190", "last_arrival": "2020-04-02T10:54:26", "stock": 98}, {"shoe_id": 2, "name": "banana-gomeisa-2129", "last_arrival": "2001-01-10T07:39:43", "stock": 99}, {"shoe_id": 3, "name": "orange-enif-2083", "last_arrival": "1988-02-27T16:12:57", "stock": 134}, {"shoe_id": 4, "name": "orange-atlas-2091", "last_arrival": "1980-02-16T11:49:52", "stock": 111}, {"shoe_id": 5, "name": "orange-deneb-2118", "last_arrival": "2001-08-18T03:09:10", "stock": 1}, {"shoe_id": 6, "name": "banana-enif-1989", "last_arrival": "2007-07-03T07:12:42", "stock": 56}, {"shoe_id": 7, "name": "grape-hamal-2075", "last_arrival": "2015-06-20T00:57:25", "stock": 69}, {"shoe_id": 8, "name": "mango-atlas-2123", "last_arrival": "1985-04-08T16:55:27", "stock": 106}, {"shoe_id": 9, "name": "mango-caph-2135", "last_arrival": "1995-08-14T07:44:49", "stock": 135}]
...(略)...
```

無事に起動した！！

## まとめ

はじめからそうなんだけど、GKEからCloud SQLをCloud SQL ProxyとWorkload Identityを使ってつなぐ場合には、公式ドキュメントどおりに設定していけばいい。

* [Connecting from Google Kubernetes Engine](https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine)

結局気をつけておくことは

* GKEクラスターをVPCネイティブで作っておく（あとから変えられない）
* Kubernetes Service Accountのapplyを忘れない
  * `kubectl annotate serviceaccount` はエラー無しで動いてしまうので気をつける
* Cloud SQL Proxyは結構マシンパワーを使うので強めのインスタンスでオートスケール設定しといたほうが無難