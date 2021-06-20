---
title: "CIでArtifact Registryから古いイメージを削除する"
date: 2021-06-20T10:55:58+09:00
draft: false
tags: ["artifact-registry", "cloud-build", "container", "cicd", "devops", "issue"]
---

## CIをしていると古いアーティファクトが残る問題

pyspabotのリファクタリングが進んできたので、プロジェクトのコスト節約のために古いコンテナイメージは自動で削除されたほうがいいと思い調査を始めた。

[公式ドキュメント](https://cloud.google.com/artifact-registry/docs/docker/manage-images)を見てたら、Sethが書いた [GCR Cleaner](https://github.com/sethvargo/gcr-cleaner) を使うと楽だよと書いてあった。
しかしセットアップドキュメントをぱっと見たところ、なんかめんどくさそうなことを書いてるから、`gcloud`コマンドとCloud Buildで簡単にできないか考えた。

## フィルター条件を考える

厳密な管理ではなく、あくまで古いものを消したいというモチベーションなので、ビルドのたびに下記くらいの条件でイメージを削除することにする。

* 過去1ヶ月より昔のビルド
* 直近5件以外
* 保存用のタグがついていない

以下の比較用にフィルターなしの実行結果を載せておく。

```console
$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot            Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

IMAGE                                                       DIGEST                                                                   CREATE_TIME          UPDATE_TIME
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:3fd331727b1e75fe5f5e4117ccec75946adf99ebf80646cda72eecfaf3de15df  2021-06-20T10:58:05  2021-06-20T10:58:05
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:5ca9f6e9f91966e5f848676dc460232f04bbe00286aebf2b534d9cc5cf658663  2021-06-20T10:34:41  2021-06-20T10:34:41
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:7d2c7ab739d3142e765769ca2ae8e58d8cce4865e7ae3d07a01c6f5b77ebf391  2021-06-19T22:44:42  2021-06-19T22:44:42
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:8656f70913ebdf376902ed55d9a397e9ee6f7128106bdc53d154a6e8bad45f0a  2021-06-19T22:52:51  2021-06-19T22:52:51
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:b3ef690a4d0f4a6e911543356e7f2502de9a34fadd015c6b26375794467e98d3  2021-06-20T00:16:39  2021-06-20T00:16:39
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:c025f6cd575935c3a02fb1b85d06064ff7d2d74acf9aa0fe6febb08cdaaf6df6  2021-06-19T23:19:18  2021-06-19T23:19:18
```

### 時間によるフィルター

「過去1ヶ月より昔のビルド」は時間によるフィルターで対応できそう。これは2021年6月20日より前に作成されたコンテナイメージの一覧を持ってきた結果。上手く動いている。

```console
$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --filter="update_time < 2021-06-20"
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

IMAGE                                                       DIGEST                                                                   CREATE_TIME          UPDATE_TIME
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:7d2c7ab739d3142e765769ca2ae8e58d8cce4865e7ae3d07a01c6f5b77ebf391  2021-06-19T22:44:42  2021-06-19T22:44:42
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:8656f70913ebdf376902ed55d9a397e9ee6f7128106bdc53d154a6e8bad45f0a  2021-06-19T22:52:51  2021-06-19T22:52:51
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:c025f6cd575935c3a02fb1b85d06064ff7d2d74acf9aa0fe6febb08cdaaf6df6  2021-06-19T23:19:18  2021-06-19T23:19:18
```

### 直近N件でのフィルター

これは一発ではできないけど組み合わせでできそう。`--sort-by`と`--limit`があるので、`update_time`で降順にして「全件数-N」で表示された分を削除すれば行ける。

たとえば直近2件だけを残すために、それ以外を取得するために実行してみたコマンド。`$limit`は確認のために見ている。

```console
$ nums=$(gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --format="value(update_time)" --quiet | wc -l)
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

$ echo $nums
6

$ limit=$( expr $(($nums-2)) )

$ echo $limit
4

$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --sort-by=update_time --limit=$( expr $(( $nums - 2 )) )
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

IMAGE                                                       DIGEST                                                                   CREATE_TIME          UPDATE_TIME
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:7d2c7ab739d3142e765769ca2ae8e58d8cce4865e7ae3d07a01c6f5b77ebf391  2021-06-19T22:44:42  2021-06-19T22:44:42
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:8656f70913ebdf376902ed55d9a397e9ee6f7128106bdc53d154a6e8bad45f0a  2021-06-19T22:52:51  2021-06-19T22:52:51
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:c025f6cd575935c3a02fb1b85d06064ff7d2d74acf9aa0fe6febb08cdaaf6df6  2021-06-19T23:19:18  2021-06-19T23:19:18
us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot  sha256:b3ef690a4d0f4a6e911543356e7f2502de9a34fadd015c6b26375794467e98d3  2021-06-20T00:16:39  2021-06-20T00:16:39

$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --sort-by=update_time --limit=$( expr $(( $nums - 2 )) ) --format="value(version)"
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

sha256:7d2c7ab739d3142e765769ca2ae8e58d8cce4865e7ae3d07a01c6f5b77ebf391
sha256:8656f70913ebdf376902ed55d9a397e9ee6f7128106bdc53d154a6e8bad45f0a
sha256:c025f6cd575935c3a02fb1b85d06064ff7d2d74acf9aa0fe6febb08cdaaf6df6
sha256:b3ef690a4d0f4a6e911543356e7f2502de9a34fadd015c6b26375794467e98d3
```

この `version` の値で `gcloud artifacts docker images delete` すれば良さそう。試してみる。

```console
$ for version in $( gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --sort-by=update_time --limit=$( expr $(( $nums - 2 )) ) --format="value(version)" ); do
gcloud artifacts docker images delete us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot@$version --quiet
done
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

Digests:
- us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot@sha256:7d2c7ab739d3142e765769ca2ae8e58d8cce4865e7ae3d07a01c6f5b77ebf391
Delete request issued.
Waiting for operation [projects/pyspa-bot/locations/us-west1/operations/eec1318c-a4c1-4ea9-a829-cca64b7c15f0
] to complete...done.
Digests:
- us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot@sha256:8656f70913ebdf376902ed55d9a397e9ee6f7128106bdc53d154a6e8bad45f0a
Delete request issued.
Waiting for operation [projects/pyspa-bot/locations/us-west1/operations/f1436585-b823-452c-98e2-6e319a80a269
] to complete...done.
Digests:
- us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot@sha256:c025f6cd575935c3a02fb1b85d06064ff7d2d74acf9aa0fe6febb08cdaaf6df6
Delete request issued.
Waiting for operation [projects/pyspa-bot/locations/us-west1/operations/98ec0134-9045-47e2-b35a-109c599d1317
] to complete...done.
Digests:
- us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot@sha256:b3ef690a4d0f4a6e911543356e7f2502de9a34fadd015c6b26375794467e98d3
Delete request issued.
Waiting for operation [projects/pyspa-bot/locations/us-west1/operations/0e5cc877-b525-4608-8c00-7a2bdb0c27f3
] to complete...done.
```

できた。

### タグでフィルター

`tags` フィールドが一切値を返さず、フィルターも効いてないので、バグってんじゃないかと思う。（要確認）

```console
$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --format=yaml
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

---
createTime: '2021-06-20T01:58:05.111183Z'
package: us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot
tags: ''
updateTime: '2021-06-20T01:58:05.111183Z'
version: sha256:3fd331727b1e75fe5f5e4117ccec75946adf99ebf80646cda72eecfaf3de15df
---
createTime: '2021-06-20T01:34:41.933455Z'
package: us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot
tags: ''
updateTime: '2021-06-20T01:34:41.933455Z'
version: sha256:5ca9f6e9f91966e5f848676dc460232f04bbe00286aebf2b534d9cc5cf658663


$ gcloud artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --format=yaml --filter="tags:test"
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

$ gcloud alpha artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --format=yaml --filter="tags:test"
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.

$ gcloud beta artifacts docker images list us-west1-docker.pkg.dev/pyspa-bot/pyspabot-images/pyspabot --format=yaml --filter="tags:test"
Listing items under project pyspa-bot, location us-west1, repository pyspabot-images.
```
