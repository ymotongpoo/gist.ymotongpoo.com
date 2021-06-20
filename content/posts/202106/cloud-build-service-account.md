---
title: "Cloud Buildで独自のService Accountを使う"
date: 2021-06-17T14:58:07+09:00
draft: false
tags: ["google-cloud", "cloud-build", "iam", "cicd"]
---

## `serviceAccount` セクション

最近になって `cloudbuild.yaml` で `serviceAccount` セクションがbetaで追加されていた。

* [Build configuration overview | Cloud Build](https://cloud.google.com/build/docs/build-config)

とりあえず次のように設定して手動トリガーで試してみたら動いた。

```yaml
...

serviceAccount: "projects/${PROJECT_ID}/serviceAccounts/xxxxxxx@xxxx.iam.gserviceaccount.com"
...
```

## ビルドトリガーだと動かない

ところが、これでいいと思ったらGitHubのpushをトリガーにしたビルドだと次のようなエラーになって動かない。

{{< figure src="/image/202106/20210617-1.png" >}}

* [Configuring user-specified service accounts | Limitations](https://cloud.google.com/build/docs/securing-builds/configure-user-specified-service-accounts#limitations)

ここにはまだ使えないよと書いてある。

> User-specified service accounts only work with manual builds; they don't work with build triggers.

で、これはいつ直るのかなと思って見てみたら2021年2月から更新がない。

* [issue #175097630 | Specific service account per trigger](https://issuetracker.google.com/issues/175097630#comment5)

早く動いてほしいなー。