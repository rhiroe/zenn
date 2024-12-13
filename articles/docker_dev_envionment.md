---
title: Dockerで自分専用のコンテナ環境を作成する
date: 2024-2-8
tags: ["Docker"]
---

## 背景

最近だと開発環境でDockerを使う場面も増えてきていると思います。僕は普段Railsのアプリケーションを開発していますが、開発環境はDockerのコンテナで立ち上げています。この場合、開発スタイルは以下の2つに分かれると思います。

- `docker compose exec`を駆使してホストで開発する
- コンテナに入って開発する

僕は後者のスタイルが好きですが、前者のホストで開発するスタイルをよく見かけます。

前者の場合、コマンド実行する際に`docker compose exec`を駆使する必要があり、タイプ量も多く億劫に感じることも多いので、それの回避方法を幾つか紹介するとともに、こっそり自分だけ後者のスタイルで開発できるようにする方法も紹介します。

## 1. aliasを設定する

一番簡単な方法かつ有名なものかなと思います。`docker compose exec`のaliasを登録しておくことで、タイプ量を削減します。

```shell
alias dce="docker compose exec"
```

```shell
dce app bin/rails s
```

## 2. Makefileを作成する

アプローチはaliasと似ていますが、Makefileでコマンドを定義しておくことで、さまざまなコマンドを`make ○○`で実行できます。

## 3. docker-compose.yamlをoverrideする

これが本題で一番書きたかったことです。gitツリーを汚さずに自分の`docker-compose.yaml`だけを上書きします。

`docker-compose.yaml`の上書きといえば`docker-compose.override.yaml`なので、それを用意します。

```yaml
services:
  app:
    volumes:
      - .git:/usr/src/app/.git:delegated
      - /Users/username/.ssh:/root/.ssh:delegated
```

コンテナの中でgit操作を行いたいので`.git`、GitHubにssh接続したいので`~/.ssh`をマウントします。

このままだとコミットに含まれてしまいます。`.gitignore`を書いたとしても差分が発生してしまいます。

ここで編集するファイルは`.git/info/exclude`です。

```shell
vi .git/info/exclude
```

```text
docker-compose.override.yaml
```

役割としては`.gitignore`と同じようなもので、記述したファイルがコミット対象に含まれなくなります。

これで自分だけこっそりとコンテナに入って作業をすることができるようになります。
