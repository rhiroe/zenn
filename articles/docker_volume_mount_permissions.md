---
title: Dockerのvolume mountしたディレクトリの中の所有者をコンテナの実行ユーザーに変更する
date: 2024-10-8
tags: ["Docker"]
---

# 結論

Dockerfileでvolume mountするディレクトリをあらかじめ作成しておく

# 詳細

こんな感じのDockerfileがあり
```Dockerfile
FROM ruby:3.3.4-bullseye

ENV LANG=C.UTF-8
ENV TZ=Asia/Tokyo
ENV EDITOR=vim

EXPOSE 3000

RUN apt update -qq &&\
    apt upgrade -y &&\
    apt install -y --no-install-recommends \
    vim &&\
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
```

こんな感じのcompose.yamlがあり
```compose.yaml
services:
  app:
    build:
      context: .
    ports:
      - 3001:3000
    volumes:
      - ..:/usr/src/app
      - bundle:/usr/local/bundle:cached
```

こんな感じのdevcontainer.jsonがあり
```devcontainer.json
{
  "name": "Rails App",
  "service": "app",
  "overrideCommand": true,
  "containerUser": "vscode",
  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {
      "username": "vscode"
    },
  }
}
```

コンテナの実行ユーザーが一般ユーザーの場合に、コンテナ起動後に`bundle install`を行うと`/usr/local/bundle/cache`に書き込み権限がないとしてエラーになっていた。

`/usr/local/bundle/cache`の権限を見ると、所有者のみ書き込み可能で所有者はrootユーザーになっていた。

volume対象のディレクトリがない場合、Dockerがよしなにディレクトリを生成してくれるが、その場合の所有者はrootとなり、この所有者や権限を後から変えることはできない。

なので、Dockerfileであらかじめディレクトリを作成しておく。

```Dockerfile
RUN mkdir -p /usr/local/bundle
```

これで `/usr/local/bundle`自体の所有者はroot(Dockerfile内の実行USER)だが、その中に作成されるディレクトリの所有者はコンテナの実行ユーザーとなった。
