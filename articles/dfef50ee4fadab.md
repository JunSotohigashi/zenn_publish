---
title: "開発環境としてのDocker運用ベストプラクティス"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "VS Code", "devcontainer"]
published: false
---

# 概要
ホスト側のワーキングディレクトリを，コンテナ内にバインドして開発できると便利です．ところが，コンテナのデフォルトユーザはrootなので，コンテナ内からファイルを作成すると，所有者がrootになってしまいます．この問題への対応方法はいくつかあるのですが，VSCodeと組み合わせると一筋縄ではいかなかったので，知見を共有します．


# 最小構成
## ファイル構成
```
project_hoge
├── .devcontainer.json
├── Dockerfile
├── compose.yaml
└── entrypoint.sh
```

```json:.devcontainer.json
{
    "dockerComposeFile": "compose.yaml",
    "service": "demo_app",
    "runServices": [
        "demo_app"
    ],
    "remoteUser": "user",
    "updateRemoteUserUID": false,
    "workspaceFolder": "/workspace",
    "shutdownAction": "stopCompose",
    "customizations": {
        "vscode": {
            "settings": {
                "terminal.integrated.defaultProfile.linux": "bash"
            },
            "extensions": []
        }
    }
}
```


```Dockerfile:Dockerfile
FROM debian:bookworm-slim

##############################
# Install dependencies at here
##############################

# Example1: Install git
# RUN apt-get update && \
#     apt-get install -y --no-install-recommends \
#     git && \
#     apt-get clean

# Example2: Install Python and pip
# RUN apt-get update && \
#     apt-get install -y --no-install-recommends \
#     python3 python3-pip && \
#     apt-get clean
# COPY requirements.txt /tmp/requirements.txt
# RUN pip3 install --no-cache-dir -r /tmp/requirements.txt


# Create non-root user
RUN groupadd -g 1000 user && \
    useradd -m -u 1000 -g user user

# Modify UID and GID with entrypoint script
COPY entrypoint.sh /
ENTRYPOINT [ "/entrypoint.sh" ]

CMD [ "bash" ]

```

```yaml:compose.yaml
services:
  demo_app:
    build:
      context: .
      dockerfile: Dockerfile
    image: dev_docker_demo_app
    tty: true
    volumes:
      - .:/workspace
    working_dir: /workspace
    environment:
      - TERM=xterm-256color
    ################################################
    # Uncomment below lines if you want to use GPU
    ################################################
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

```

```sh:entrypoint.sh
#!/bin/bash

# This script changes the UID and GID of the in-container user
# to match the current directory's UID and GID.
# This script must be run as root, then it will switch to the non-root user.

USER=user
HOME=/home/$USER

# Get UID and GID from the current directory
uid=$(stat -c "%u" .)
gid=$(stat -c "%g" .)

# Change in-container user's UID and GID to match the current directory
if [ "$uid" -ne 0 ] && [ "$uid" -ne "$(id -u $USER)" ]; then
    usermod -u $uid $USER
    groupmod -g $gid $USER
    chown -R $uid:$gid $HOME
fi

# Switch to the non-root user
exec setpriv --reuid=$USER --regid=$USER --init-groups "$@"

```

## コンテナ操作
```sh
# ビルド
docker compose build

# コマンドの実行
docker compose run --rm demo_app (command)

# ビルド+コマンドの実行
docker compose run --build --rm demo_app (command)
```


# 解説
## Dockerfile
ここで必要なパッケージのインストールなどを行います．
この時点ではrootですので，`/root`に作成したデータは一般ユーザとなった後で参照できないことに注意してください．最後に，イメージ内に一般ユーザを作成し，コンテナの立ち上げ時に動的にUIDを変更するため，`entrypoint.sh`を設定しておきます．

## compose.yaml
ここで，イメージ名の設定や，ワークスペースのバインド，GPU使用の設定を行います．
Docker composeを使用すると，立ち上げコマンドがシンプルになるほか，複数コンテナを運用するときも好都合です．

## entrypoint.sh
バインドされたワークスペースのownerのUIDを取得し，コンテナ内ユーザのUIDを変更します．
`/home/user`以下の全ファイルのownerを再帰的に変更する処理が行われますので，大量のファイルがあるとコンテナの起動処理が重くなります．このスクリプトはrootとして開始されますが，UIDの切り替えが完了するとuserに切り替わります．そのため，dockerの`-u`オプションで強制的にUIDを変更しないでください．

## .devcontainer.json
VSCodeをコンテナにアタッチするときの構成ファイルです．
コンテナ内ユーザをuserに設定します．この項目がないと，rootとしてアタッチされてしまいます．


# 補足
## devcontainerのupdateRemoteUserUID機能について
この機能を使用すると，新しいユーザを追加してイメージを再ビルドする処理が行わるため，コンテナ立ち上げが遅くなります．また，厳密には別のイメージを使用することになるため，再現性の観点であまり好ましくありません．

## SSH，GPGについて
プライベートリポジトリへのアクセス時など，本来であればSSH秘密鍵をコンテナ内に持ち込む必要があります．しかし，VSCodeをアタッチすることでこれらは自動的に行われますので，ssh-agentなどの設定は必要ありません．

## 複数のイメージを運用する場合
イメージ毎に`devcontainer.json`を用意してください．

`entrypoint.sh`は必ずコピーしてください．Dockerfileより上の階層に配置してしまうと，ビルド時に参照できません．

### ファイル構成
```
project_hoge
├── .devcontainer
│   ├── app1
│   │   └── devcontainer.json
│   └── app2
│       └── devcontainer.json
├── app1
│   ├── Dockerfile
│   └── entrypoint.sh
├── app2
│   ├── Dockerfile
│   └── entrypoint.sh
└── compose.yaml
```

```json:.devcontainer/app1/devcontainer.json
{
    "dockerComposeFile": "compose.yaml",
    "service": "demo_app1",
    "runServices": [
        "demo_app1"
    ],
    "remoteUser": "user",
    "updateRemoteUserUID": false,
    "workspaceFolder": "/workspace",
    "shutdownAction": "stopCompose",
    "customizations": {
        "vscode": {
            "settings": {
                "terminal.integrated.defaultProfile.linux": "bash"
            },
            "extensions": []
        }
    }
}
```

```json:.devcontainer/app2/devcontainer.json
{
    "dockerComposeFile": "compose.yaml",
    "service": "demo_app2",
    "runServices": [
        "demo_app2"
    ],
    "remoteUser": "user",
    "updateRemoteUserUID": false,
    "workspaceFolder": "/workspace",
    "shutdownAction": "stopCompose",
    "customizations": {
        "vscode": {
            "settings": {
                "terminal.integrated.defaultProfile.linux": "bash"
            },
            "extensions": []
        }
    }
}
```

```yaml:compose.yaml
services:
  demo_app1:
    build:
      context: ./app1
      dockerfile: Dockerfile
    image: dev_docker_demo_app1
    tty: true
    volumes:
      - .:/workspace
    working_dir: /workspace
    environment:
      - TERM=xterm-256color
    ################################################
    # Uncomment below lines if you want to use GPU
    ################################################
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]
  demo_app2:
    build:
      context: ./app2
      dockerfile: Dockerfile
    image: dev_docker_demo_app2
    tty: true
    volumes:
      - .:/workspace
    working_dir: /workspace
    environment:
      - TERM=xterm-256color
```



# まとめ
* compose.yamlでワーキングディレクトリをバインド
* Dockerfileで一般ユーザを作成
* entrypoint.shで，コンテナ立ち上げ時に，コンテナ内ユーザのUIDをホスト側に合わせる
* UIDはワーキングディレクトリの所有者から取得する
* devcontainer.jsonを作成することでVSCodeに対応





