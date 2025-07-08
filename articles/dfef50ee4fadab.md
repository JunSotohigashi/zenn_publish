---
title: "開発環境としてのDocker運用ベストプラクティス"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "VS Code", "devcontainer"]
published: false
---

# 概要
ホスト側のワーキングディレクトリを，コンテナ内にバインドして開発できると便利です．
ところが，コンテナのデフォルトユーザはrootなので，コンテナ内からファイルを作成すると，所有者がrootになってしまいます．
この問題への対応方法はいくつかあるのですが，VSCodeと組み合わせると一筋縄ではいかなかったので，知見を共有します．




# 最小構成
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
FROM debian:slim

##############################
# Install dependencies at here
##############################

# Example1: Install graphviz
# RUN apt-get update && \
#     apt-get install -y --no-install-recommends \
#     graphviz && \
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

# まとめ
* compose.yamlでワーキングディレクトリをバインド
* Dockerfileで一般ユーザを作成
* entrypoint.shで，コンテナ立ち上げ時に，コンテナ内ユーザのUIDをホスト側に合わせる
* UIDはワーキングディレクトリの所有者から取得する
* devcontainer.jsonを作成することでVSCodeに対応





