---
title: "Terraform+AnsibleでGoogle Compute Engineインスタンスを作る"
date: 2021-07-12T11:08:33+09:00
draft: false
tags: ["terraform", "ansible", "secret-manager", "compute-engine", "cloud-build"]
---

Google Cloud Buildを使ってGoogle Compute Engine上に開発環境を作ろうと思ったんで諸々メモ。

## TerraformでGoogle Secret Managerを扱う

Sethが書いたブログ記事に簡潔にまとまっている。さすが元Hashicorp、さすがセキュリティ担当。

* https://www.sethvargo.com/accessing-google-secret-manager-secrets-from-terraform/

まずこういう具合にデータを用意する。

```hcl
data "google_secret_manager_secret_version" "terraform_ssh_private_key" {
    provider = google
    secret = "terraform-ssh-private-key"
    version = "1"
}

data "google_secret_manager_secret_version" "terraform_ssh_pub_key" {
    provider = google
    secret = "terraform-ssh-pub-key"
    version = "1"
}
```

そして、Google Compute Engineのインスタンス作成において次のように設定する。

```hcl
resource "google_compute_instance" "debian10" {
    name = "debian10"
    machine_type = "e2-standard-4"

    ...(略)...

    metadata = {
        block-project-ssh-keys = "true"
        ssh-keys = "${var.gce_ssh_user}:${data.google_secret_manager_secret_version.terraform_ssh_pub_key.secret_data}"
    }

    provisioner "remote-exec" {
        inline = ["echo hello"]
        connection {
            type = "ssh"
            host = self.network_interface[0].access_config[0].nat_ip
            user = var.gce_ssh_user
            private_key = data.google_secret_manager_secret_version.terraform_ssh_private_key.secret_data
        }
    }

    provisioner "local-exec" {
        working_dir = "./ansible/"
        command = <<EOL
        ANSIBLE_HOST_KEY_CHEKING=False ansible-playbook -i hosts/inventory.gcp.yaml debian.yaml
        EOL
    }
}
```

ポイントはシークレットの値にアクセスする際に `data.google_secret_manager_secret_version.<resource_name>.secret_data` とすること。最後の値へのアクセスが `secret_data` になっているのが注意。

## Cloud Buildで呼んだTerraform内でAnsibleを呼べるようにする

Cloud Buildの設定ではTerraformのbuilderに `hashicorp/terraform:<version>` を使っているが、これを使うと `local-exec` でAnsibleが呼び出せない。（イメージ内にAnsibleが入ってないので当たり前）

仕方ないので自分でCloud Builderを作成する。最後は `ENTRYPOINT` にしないと起動できないので注意。

```dockerfile
FROM python:3.9-slim-buster as fetcher
ENV TERRAFORM_VER="1.0.2"
ENV ANSIBLE_VER="2.11.0"
RUN apt-get update && apt-get install -y wget unzip
WORKDIR /download
RUN wget -q -O terraform.zip "https://releases.hashicorp.com/terraform/1.0.2/terraform_${TERRAFORM_VER}_linux_amd64.zip" \
    && unzip terraform.zip \
    && rm terraform.zip

RUN wget https://bootstrap.pypa.io/get-pip.py -O get-pip.py \
    && python3 get-pip.py && rm get-pip.py

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
RUN ansible-galaxy collection install google.cloud community.general
RUN ansible-galaxy install kewlfft.aur fubarhouse.golang

FROM python:3.9-slim-buster
COPY --from=fetcher /download/terraform /usr/local/bin/terraform
RUN chmod +x /usr/local/bin/terraform
COPY --from=fetcher /usr/local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
COPY --from=fetcher /usr/local/bin/ansible /usr/local/bin/ansible
COPY terraform.bash /usr/local/bin/terraform.bash
RUN chmod +x /usr/local/bin/terraform.bash
ENTRYPOINT ["/usr/local/bin/terraform.bash"]
```

また、`ENTRYPOINT` 用のコマンドがCloud Buildから引数を受け取れるように、このようにラッパーのシェルスクリプトを用意しておいて、そちらを指定するのが一般的。

```bash
echo "Running: terraform $@"
/usr/local/bin/terraform "$@"
```


## Ansibleのhosts内で使うサービスアカウント情報を渡す部分

```yaml
---
# This inventory is using gcp_compute plugin. Install the plugin in advance of the execution.
# $ ansible-galaxy collection install google.cloud
# https://docs.ansible.com/ansible/latest/collections/google/cloud/gcp_compute_inventory.html
plugin: gcp_compute
zones: # populate inventory with instances in these regions
  - asia-northeast1-b
  - asia-northeast1-a
  - asia-northeast2-b
  - asia-northeast2-a
  - asia-east1-b
projects: development-215403
service_account_file: /workspace/ansible-service-account.json
auth_kind: serviceaccount
scopes:
 - 'https://www.googleapis.com/auth/compute'
hostnames:
  # List host by name instead of the default public ip
  - 'name'
compose:
  # Set an inventory parameter to use the Public IP address to connect to the host
  # For Private ip use "networkInterfaces[0].networkIP"
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
```

`service_account_file` の情報を渡さないと `ansible-playbook` が実行が失敗するわけだけど、これは `google_compute_instance` の `local-exec` からだと変数として値を渡せるものではない。

仕方がないのでCloud Buildの特徴を使って `/workspace` 以下にサービスアカウントファイルを持ってくる手順を `cloudbuild.yaml` に記述する。

```yaml
- name: 'gcr.io/cloud-builders/gcloud-slim'
  entrypoint: 'bash'
  args: ['shellscript/fetch_service_account_file.bash']
```

この中で呼んでるシェルスクリプトはこんな感じ。

```bash
#!/bin/bash
set -ex

IAM_ACCOUNT="cloud-build-terraform-ansible@development-215403.iam.gserviceaccount.com"

keys=$( gcloud iam service-accounts keys list \
  --iam-account=${IAM_ACCOUNT} \
  --filter="key_type=USER_MANAGED" \
  --format="value(name)" )

if [ -n "$keys" ]; then
    for key in $keys; do
        gcloud iam service-accounts keys delete $key \
          --iam-account=${IAM_ACCOUNT}
    done
fi

gcloud iam service-accounts keys create ansible-service-account.json \
  --iam-account=${IAM_ACCOUNT}
```


## Ansibleが "Authentication failed" となってapt関連を動かせない

これでめちゃくちゃはまった。

```console
Error running command 'ANSIBLE_HOST_KEY_CHEKING=False ansible-playbook -i hosts/inventory.gcp.yaml debian.yaml': exit status 4. Output:
PLAY [debian]
******************************************************************

TASK [Gathering Facts]
*********************************************************
[WARNING]: Platform linux on host debian is using the discovered Python interpreter at /usr/bin/python3.7, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible/2.11/reference_appendices/interpreter_discovery.html for more information.
ok: [debian]

TASK [debian : Change default shell]
*******************************************
changed: [debian]

TASK [debian : Install preconditional apt packages]
****************************
failed: [debian] (item={'name': 'apt-transport-https', 'state': 'latest'}) => {"ansible_loop_var": "item", "item": {"name": "apt-transport-https", "state": "latest"}, "msg": "Failed to uthenticate: Authentication failed.", "unreachable": true}
```

実際はAnsibleは全然関係なくて、taskの順番で実際に変更後に使うshellをインストールする前にそのshellに変更してしまったから、shellに接続できなくて死んでたという話。エラーメッセージだけみてSSH周りがおかしいのだと思って混乱してた。再現する簡単な設定。

```yaml
- name: Change default shell
  ansible.builtin.user:
    name: demo
    shell: /usr/bin/zsh
  become: yes

- name: Update package
  ansible.builtin.apt:
    name: zsh
    state: latest
  become: yes
```
