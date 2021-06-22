---
title: "PackerのFile Provisionerでできないこと"
date: 2021-06-21T22:13:38+09:00
draft: false
tags: ["packer", "devops"]
---

## root権限が使えない

たとえばSystemdの `.service` ファイルを所定の場所に置くために次のように書いた。

```json
"provisioners": [
  {
    "destination": "/etc/systemd/system/foo.service",
    "source": "foo.service",
    "type": "file"
  }
]
```

しかしこれを実行するとエラーになる。（次のログはCloud BuildでPackerを実行したログ）

```console
Step #3 - "build_gce_image": ==> googlecompute: Uploading ./config/foo.service => /etc/systemd/system/foo.service
Step #3 - "build_gce_image": ==> googlecompute: Upload failed: scp: /etc/systemd/system/foo.service: Permission denied
```

コレに関しては7年前のIssueがいまだに出てくる。

* [file provisioner permission denied #1551](https://github.com/hashicorp/packer/issues/1551)

@mitchellh の回答がこれ。

> Yep, sorry, I believe the documentation states that you should upload to somewhere you have write access, then use a shell provisioner to move things over. I stand by this since supporting automatic copy is a little too magical for my taste.

いけてなさすぎだと思うけど、そういうもんだと諦めた。

## ディレクトリーのコピーができない

Packerでディレクトリーをまるごとコピーしたいと思ったのでカジュアルに次のような設定を書いた。

```json
"provisioners": [
  {
    "destination": ".",
    "source": ".",
    "type": "file"
  }
]
```

しかしこれは全然だめだった。

```console
Step #3 - "build_gce_image": Build 'googlecompute' errored: scp: error: unexpected filename: .
```

Cloud Buildでやるなら事前にコピーしたいディレクトリをtarballにしておいて、それをコピーした後に、Shell Provisionerとかで展開しないとだめっぽい。面倒すぎる。
