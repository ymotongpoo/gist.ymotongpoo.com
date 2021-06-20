---
title: "DockerでSystemdが起動しない"
date: 2021-06-20T00:42:48+09:00
draft: false
tags: ["docker", "systemd", "devops"]
---

## 失敗する

Docker内でSystemdを起動しようとすると失敗する。

```console
root@3e903545869d:/app# systemctl start pyspabot
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

検索すると同様の事象の報告がたくさん出てくる。

* https://jun-app.com/articles/stuck-on-docker-centos8/
* https://stackoverflow.com/questions/67515345/failed-to-connect-to-bus-host-is-down-in-ubuntu

## PID 1 が `/sbin/bash` になっている

原因はPID 1が `/sbin/bash` になってることで、これを変更しないといけないわけだけど、調べてもかなり面倒なことしないといけないぽい。
SystemdじゃなくてSupervisordにするのがいいのかな？

(調査中)
