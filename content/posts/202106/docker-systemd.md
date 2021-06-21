---
title: "DockerでSystemdが起動しない"
date: 2021-06-20T00:42:48+09:00
draft: false
tags: ["docker", "systemd", "compute-engine", "devops"]
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

## Google Compute Engineでもコンテナではsystemdは動かせない

GCEで `create-with-container` であれば特権モードあるし動かせるかなと思ったけど同様のエラーでだめだった。

```json
{
  "insertId": "fv9v8ux3fcz3oct3k",
  "jsonPayload": {
    "message": "System has not been booted with systemd as init system (PID 1). Can't operate.\n",
    "cos.googleapis.com/stream": "stderr",
    "cos.googleapis.com/container_id": "5c6b9779143a41d205c90e9a1ffd46d80ff43d77923a86c7814afe413d1557a0"
  },
  "resource": {
    "type": "gce_instance",
    "labels": {
      "zone": "us-west1-b",
      "instance_id": "283259186628978601",
      "project_id": "pyspa-bot"
    }
  },
  "timestamp": "2021-06-20T15:54:21.192002751Z",
  "labels": {
    "compute.googleapis.com/resource_name": "6c8ca6aa99cd"
  },
  "logName": "projects/pyspa-bot/logs/cos_containers",
  "receiveTimestamp": "2021-06-20T15:54:26.940774263Z"
}
```


(調査中)
