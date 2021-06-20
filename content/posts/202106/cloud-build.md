---
title: "Cloud Buildの設定で手間取ったところ"
date: 2021-06-19T23:01:33+09:00
draft: false
tags: ["cloud-build", "secret-manager", "docker", "cicd", "devops"]
---

## Secret Managerで取得した値をDockerプロセスに渡せない

[Cloud Buildのドキュメント](https://cloud.google.com/build/docs/securing-builds/use-secrets)にあるように

* `entrypoint` フィールドで `bash` を指定
* `args` で `$$` 接頭辞付きでシークレットの代入変数を指定

したにも関わらずデータが渡ってなかった。

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: bash
  args: ['-c', 'docker build -t foo:test --build-arg FOO=$_FOO_VALUE']
  secretEnv:
  - '_FOO_VALUE'

availableSecrets:
  secretManager:
  - versionName: projects/$PROJECT_iD/secrets/foo-value/versions/latest
    env: _FOO_VALUE
```

よくよく見たら `'bash'` じゃなくて `bash` になってたのを修正したら直った。

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'bash'
  args: ['-c', 'docker build -t foo:test --build-arg FOO=$_FOO_VALUE']
  secretEnv:
  - '_FOO_VALUE'

availableSecrets:
  secretManager:
  - versionName: projects/$PROJECT_iD/secrets/foo-value/versions/latest
    env: _FOO_VALUE
```

そもそも `entrypoint` を変更して通常と異なる方法でビルドしないといけないのはイケてない気がする。
