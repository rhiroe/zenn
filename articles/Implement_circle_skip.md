---
title: CircleCIで[circle skip]を可能にするconfigの書き方
date: 2022/9/30
tags: ["CircleCI"]
---

# 完成物

.circleci/config.yml
```yml
version: 2.1
setup: true
orbs:
  continuation: circleci/continuation@0.3.1
jobs:
  setup:
    executor: continuation/default
    steps:
      - checkout
      - run: |
          MESSAGE_SUBJECT=$(git log -1 HEAD --pretty=format:%s)
          MESSAGE_BODY=$(git log -1 HEAD --pretty=format:%b)
          CIRCLESKIP="\[circle skip\]"
          if [[ ${MESSAGE_SUBJECT} =~ ${CIRCLESKIP} || ${MESSAGE_BODY} =~ ${CIRCLESKIP} ]]; then
            circleci-agent step halt
          fi
      - continuation/continue:
          configuration_path: .circleci/main.yml
workflows:
  setup:
    jobs:
      - setup
```
.circleci/main.yml
```yml
# 今あなたのリポジトリの .circleci/config.yml に書いてある内容
```

# やっていること

CircleCIの[ダイナミックコンフィグ](https://circleci.com/docs/ja/dynamic-config)と[circleci-agent step halt](https://circleci.com/docs/ja/configuration-reference#ending-a-job-from-within-a-step)の合わせ技で、本来のCircleCIのワークフローをスキップします。

CircleCIをスキップする方法には`[ci skip]`というものが用意されていますが、実際のプロダクトではいくつかのCIツールを併用していることが多いと思います。僕の関わっているプロダクトでは頻度が高く重いCIをCircleCIで、頻度が低く軽いCIをGitHubWorkflowsで実行しています。そうなると、CircleCIだけスキップしてGitHubWorkflowsだけ実行したいみたいなケースが度々発生します。そこで`circleci-agent step halt`を利用してCircleCIのJobをスキップするという方法は以前から存在していました。

元々、`circleci-agent step halt`には、ワークフローをスキップできないという課題があり、以前は全てのジョブで`[circle skip]`の判定を行いスキップしていました。これがダイナミックコンフィグのおかげでワークフローを分割でき、特定のstepをスキップするだけで後続のワークフロー自体をスキップするという芸当が可能になりました。

# 定期実行との組み合わせは？

`main.yml`とは別に定期実行用のワークフローが定義されたYAMLを用意します(ここでは`nightly.yml`とする)。あとはおそらく現在もしているであろう`[ scheduled_pipeline, << pipeline.trigger_source >> ]`での条件分岐をsetupジョブで行います。条件分岐をするタイミング次第で定期実行のみ`[circle skip]`を無視するということも可能です。

```yml
version: 2.1
setup: true
orbs:
  continuation: circleci/continuation@0.3.1
jobs:
  setup:
    executor: continuation/default
    steps:
      - checkout
      - when:
          condition:
            equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
          steps:
            - continuation/continue:
                configuration_path: .circleci/nightly.yml
      - unless:
          condition:
            equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
          steps:
            - run: |
                MESSAGE_SUBJECT=$(git log -1 HEAD --pretty=format:%s)
                MESSAGE_BODY=$(git log -1 HEAD --pretty=format:%b)
                CIRCLESKIP="\[circle skip\]"
                if [[ ${MESSAGE_SUBJECT} =~ ${CIRCLESKIP} || ${MESSAGE_BODY} =~ ${CIRCLESKIP} ]]; then
                  circleci-agent step halt
                fi
            - continuation/continue:
                configuration_path: .circleci/main.yml
workflows:
  setup:
    jobs:
      - setup
```

mainとnightlyのワークフローで使用するジョブやコマンドの定義が共通であれば、その共通定義を書いたYAML(ここでは`common.yml`とする)を用意し、`yq`コマンドでマージして使用しても良いかと思います。

```yml
version: 2.1
setup: true
orbs:
  continuation: circleci/continuation@0.3.1
jobs:
  setup:
    executor: continuation/default
    steps:
      - checkout
      - when:
          condition:
            equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
          steps:
            - run: yq eval-all '. as $item ireduce ({}; . * $item )' .circleci/nightly.yml .circleci/common.yml > .circleci/merged.yml
      - unless:
          condition:
            equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
          steps:
            - run: |
                MESSAGE_SUBJECT=$(git log -1 HEAD --pretty=format:%s)
                MESSAGE_BODY=$(git log -1 HEAD --pretty=format:%b)
                CIRCLESKIP="\[circle skip\]"
                if [[ ${MESSAGE_SUBJECT} =~ ${CIRCLESKIP} || ${MESSAGE_BODY} =~ ${CIRCLESKIP} ]]; then
                  circleci-agent step halt
                fi
            - run: yq eval-all '. as $item ireduce ({}; . * $item )' .circleci/main.yml .circleci/common.yml > .circleci/merged.yml
      - continuation/continue:
          configuration_path: .circleci/merged.yml
workflows:
  setup:
    jobs:
      - setup
```

# 〆

本来のワークフローではジョブが大量にあると思いますので、それぞれのビルドを全てスキップできるようになればコストカットに貢献できるかと思います。この情報が役に立てば幸いです。では良きCIライフを。