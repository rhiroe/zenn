---
title: 特定のファイルのみの変更でCircleCIを省略する
date: 2022/10/6
tags: ["CircleCI"]
---

前回`[circle skip]`を使えるようにしたのだが、特定のファイルのみの変更ではメッセージをつけなくとも自動的にCircleCIをスキップしたいという要望があったため対応した。週末にヒント、というかほぼ答えみたいなものを見つけていて、これをそのまま使えば良さそうだなという感じだったので10分ほどで対応した。

https://twitter.com/searls/status/1278165055654301697

.ciignoreファイルを作成

```.ciignore
README.md
doc/*
```

shellのファイルの中身はほぼ流用で

```shell
#!/bin/bash

if [[ -z $GIT_BASE_REF ]]; then
  echo "--> GIT_BASE_REF had no value"
  exit 0
fi
if [[ -z $GIT_REF ]]; then
  echo "--> GIT_REF had no value"
  exit 0
fi

# 日本語ファイルの文字化け(エスケープ)回避
git config --local core.quotepath false

non_ignored_paths=$(
  {
    git ls-files | awk '{print "./" $0}'
    find . -type f -name .ciignore
    {
      find . -type f -name .ciignore | xargs -n1 dirname |
      while read dir
      do
        sed 's|^|'"$dir/"'|' "$dir"/.ciignore
      done
    } | xargs -n1 find . -type d -name .git -prune -o -type f -path
  } | sort | uniq -u | cut -c3-
)
result="$(git diff "$GIT_BASE_REF" "$GIT_REF" $non_ignored_paths 2>&1)"
if [ -z "$result" ]; then
  echo "--> Skipping build"
  circleci-agent step halt
else
  echo "--> Continuing with build"
fi
```

前回の`[circle skip]`の判定の直前に

```yml
- run:
    name: .ciignoreに指定されているパスのファイルのみが変更されている場合はCIをスキップする
    command: |
      GIT_BASE_REF=<<pipeline.git.base_revision>> \
      GIT_REF=<<pipeline.git.revision>> \
      ./.circleci/script/skip_build_if_ignored_paths
```
こんな感じで挿入した。元Tweetで紹介されているものとほとんど全く同じ感じで流用させていただいたのだが、日本固有の問題もあり、それがファイル名に日本語が使われている場合にエラーが起きるというものだった。

```shell
git config --local core.quotepath false
```

と書いていることから察しの通り、この設定をしていないと日本語が文字化け(エスケープ)する。具体的には`git ls-files`で取得するファイル名が文字化けしている。するとどうなるかというと`git diff`に渡した時にそんなファイルはないと怒られてしまう。

逆にいうとハマったのはそのくらいで、それさえ直してしまえばすんなりと動いてくれた。

ちなみに今回の一番の発見は`uniq -u`が重複するものをリストから除外するものだということを知れたことでした。
