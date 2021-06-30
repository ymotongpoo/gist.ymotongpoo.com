---
title: "Python用のVS Codeの設定"
date: 2021-06-25T14:32:03+09:00
draft: false
tags: ["python", "vscode", "flake8", "mypy", "isort", "black", "pyproject.toml", "tox.ini"]
---

## TL;DR

```json
{
    "[python]": {
        "gitlens.codeLens.symbolScopes": [
            "!Module"
        ],
        "editor.wordBasedSuggestions": false,
        "editor.formatOnSave": true,
        "editor.formatOnPaste": false,
        "editor.insertSpaces": true,
        "editor.tabSize": 4,
        "editor.codeActionsOnSave": {
            "source.organizeImports": true
        }
    },
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python3",
    "python.pythonPath": "${workspaceFolder}/.venv/bin/python",
    "files.watcherExclude": {
        "**/.venv/**": true,
        "**/.git/objects/**": true,
        "**/.git/subtree-cache/**": true
    },
    "python.linting.pylintEnabled": false,
    "python.linting.flake8Enabled": true,
    "python.linting.lintOnSave": true,
    "python.linting.flake8Args": [
        "--config", "${workspaceFolder}/tox.ini"
    ],
    "python.formatting.provider": "black",
    "python.formatting.blackPath": "${workspaceFolder}/.venv/bin/black",
    "python.sortImports.path": "${workspaceFolder}/.venv/bin/isort",
    "python.linting.mypyEnabled": true,
    "python.linting.mypyPath": "${workspaceFolder}/.venv/bin/mypy",
    "python.linting.mypyArgs": [
        "--follow-imports=silent",
        "--show-column-numbers",
        "--config-file", "pyproject.toml"
    ],
    "autoDocstring.docstringFormat": "google"
}
```

## 使うツール

* プロジェクト管理: [poetry](https://pypi.org/project/poetry)
* インポート管理: [isort](https://pypi.org/project/isort)
* Lint: [flake8](https://pypi.org/project/flake8)
* Formatter: [black](https://pypi.org/project/black)
* Type annotation: [mypy](https;//pypi.org/project/mypy)
* テストランナー: [nox](https://pypi.org/project/nox)

## 設定

なるべく `pyproject.toml` ファイルに寄せる運用にしたいが、flake8だけまだ対応できてないのは不満。（詳細は[ここ](https://github.com/PyCQA/flake8/issues/234)にある）

flake8は `setup.cfg` には対応してるのでとりあえずそちらで対応することにする。

## `setup.cfg`

```ini
[flake8]
max-line-length = 88
ignore = E203,W503,W504

[mypy]
ignore_missing_imports = true

[isort]
profile = black
```

（WIP）
## 参考

* https://mypy.readthedocs.io/en/stable/config_file.html
* https://tox.readthedocs.io/en/latest/example/basic.html#pyproject-toml-tox-legacy-ini
* https://tox.readthedocs.io/en/latest/config.html

