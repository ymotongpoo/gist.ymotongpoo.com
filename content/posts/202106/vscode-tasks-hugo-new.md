---
title: "Vscode TasksでHugoの新規エントリーを書きやすくする"
date: 2021-06-24T09:15:15+09:00
draft: true
tags: ["vscode", "hugo", "tips"]
---

このブログはHugoで書いているわけだけど、さっとメモを取りたいときにいちいちターミナルでコマンドを叩くのは面倒。なのでVSCodeの`tasks.json`を上手く使えないかなと思ったらできたのでメモ。

## Hugoでの構成

いまはすべてのブログ記事は `content/posts/yyyymm` のディレクトリ以下に入れている。

```console
$ tree -L 3
.
├── archetypes
│   └── default.md
├── config.toml
├── content
│   ├── _index.md
│   └── posts
│       ├── 202106
│       └── _index.md
├── data
├── layouts
├── resources
│   └── ...
├── static
│   └── image
│       └── 202106
└── themes
    └── ...
```

これはコマンドで実行すると `hugo new posts/yyyymm/post-title.md` という具合になる。コレが面倒なのでVSCode Tasksで少しショートカットを作った。

## `tasks.json`

まず `tasks.json` で `post` というタスクを次のように設定する。このとき `title` という変数を作ってそれをプロンプトで入力させられるようにしている。

```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "post",
            "type": "shell",
            "command": "hugo new posts/$(date +%Y%m)/${input:title}.md",
            "problemMatcher": []
        }
    ],
    "inputs": [
        {
            "id": "title",
            "description": "entry title",
            "default": "",
            "type": "promptString"
        }
    ]
}
```

ショートカットも設定したいけどとりあえずコレで。
