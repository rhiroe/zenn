---
title: DevContainer Featuresを作成し、開発環境の構築を一部容易にする
date: 2024/12/13
tags: ["VS Code", "DevContainer"]
---

# 前置き

これは、[Timee Product Advent Calendar 2024](https://qiita.com/advent-calendar/2024/timee-product)の16日目の記事です。

### この記事の3行まとめ

開発環境を構築する手順のうち、特定のソフトウェアのインストールを含む手順をDevContainer Features化し、インストールの手順を自動化した。

DevContainer Features化することで、すべての開発者がインストールするソフトウェアのバージョンを統一でき、発生した問題に対してDevContainer Featuresを修正することで他の開発者が同じ問題に遭遇することがなくなった。

DevContainer Featuresの作成はとても簡単であり、開発環境構築のうちソフトウェアのインストールを含む手順をDevContainer Features化することをおすすめする。

# こんにちは

株式会社タイミーでバックエンドエンジニアをしている廣江([rhiroe](https://x.com/buta_botti))です。

# DevContainer Featuresを作った

以前、saml2awsというDevContainer Featuresを作成しました。
[https://github.com/rhiroe/features/tree/main/src/saml2aws](https://github.com/rhiroe/features/tree/main/src/saml2aws)

saml2awsというDevContainer Featuresは、その名の通りコンテナに[saml2aws](https://github.com/Versent/saml2aws)をインストールするためのものです。

これを利用することにより、DevContainer内で `saml2aws` コマンドが使用可能になります。また、インストールされるソフトウェアのバージョンが全ての開発者間で揃えられるため、何か問題が起きた場合であっても、このDevContainer Featuresを修正することで、同様の問題に別の開発者が遭遇するということがなくなります。

## 背景

タイミーでは開発環境を構築する際に必要な環境変数をAWSのパラメータストアから取得する手順があります。その際に[aws-cli](https://github.com/aws/aws-cli)をはじめとしたいくつかのソフトウェアをマシンにインストールする必要があります。

また、AWSへのログインにGoogleアカウントとsaml2awsを利用しているのですが、このsaml2awsがGoogleのログインフォームの変更に合わせてバージョンアップを行なっているため、インストールしているバージョンが古いとログインに失敗するなんてことがあります。

さらに、これらのソフトウェアのインストール手順が普段開発しているリポジトリとは別のリポジトリにドキュメントとして書かれていたり、その中には開発環境の構築には無関係なドキュメントも含まれていたりと、環境変数の取得周りのハードルがやや高い状態でした。

これを解消し、DevContainerを起動するだけで概ね開発に必要なソフトウェアが揃っており、環境構築がスムーズに完了する状態を目指すため、DevContainer Featuresの作成に取り掛かりました。

## DevContainer Featuresの作り方

作り方自体はとても簡単です。

公式にテンプレートが用意されているので、まずはそれをコピーします。

[https://github.com/devcontainers/template-starter](https://github.com/devcontainers/template-starter)

ミラープッシュを行うとフォークせずリポジトリのコピーができます。

```
$ git clone --bare <https://github.com/devcontainers/template-starter.git>
$ cd template-starter
$ git push --mirror <https://github.com/${username}/${reponame}.git>

```

すると、中にはすでにサンプルとして`color`と`hello`というFeatureが存在するので、それを書き換えたり参考にしつつ作り始めることができます。

### devcontainer-template.json

オプションは`src/${feature_name}/devcontainer-template.json`に書いていきます。既にサンプルがある上に使用感は概ねDevContainerのdevcontainer.jsonと同じなので、なんとなく使い方はわかるかと思います。ドキュメントは下記になります。

[https://containers.dev/implementors/features](https://containers.dev/implementors/features)

ここで、`entrypoint`に指定されたshellがDevContainer起動時に実行されて、目的のソフトウェアのインストールやら何やらを行うことで目的が達成されるというような形になります。

### 頭を悩ませたポイント

entrypointのスクリプトを書く中で頭を悩ませたのは、実行ユーザーの問題でした。実行ユーザーはさまざまな場面で指定されます。それはベースイメージの実行ユーザーであったり、devcontainer.jsonのオプションであったり、自動でよしなに設定される場合もあります。その上でDevContainer Featuresはrootユーザーであっても任意の一般ユーザーであっても動作するものでなくてはいけません。そのためにはファイルの所有者と権限を実行ユーザーに合わせて適切に変更する必要があります。

しばらくどうすれば良いかわからずに悩んでいたんですが、既に世の中にあるDevContainer Featuresも同様の問題を抱えており、もう解決されているはずだと思いそちらを参考にすることにしました。具体的にはdevcontainer公式の出しているDevContainer Featuresを参考にさせていただきました。

[https://github.com/devcontainers/features](https://github.com/devcontainers/features)

たとえばdocker outside of dockerを実現する[docker-outside-of-docker](https://github.com/devcontainers/features/tree/main/src/docker-outside-of-docker)のコードを見てみると

```
# Determine the appropriate non-root user
if [ "${USERNAME}" = "auto" ] || [ "${USERNAME}" = "automatic" ]; then
    USERNAME=""
    POSSIBLE_USERS=("vscode" "node" "codespace" "$(awk -v val=1000 -F ":" '$3==val{print $1}' /etc/passwd)")
    for CURRENT_USER in "${POSSIBLE_USERS[@]}"; do
        if id -u ${CURRENT_USER} > /dev/null 2>&1; then
            USERNAME=${CURRENT_USER}
            break
        fi
    done
    if [ "${USERNAME}" = "" ]; then
        USERNAME=root
    fi
elif [ "${USERNAME}" = "none" ] || ! id -u ${USERNAME} > /dev/null 2>&1; then
    USERNAME=root
fi

```

のように自動で設定される場合は`${USERNAME}`に`"auto"`や`"automatic"`というものが来るらしいことがわかります。これを参考に同じように`${USERNAME}`の設定を行い、その後に`${USERNAME}`に合わせた所有者の変更や権限の変更を行うようにしました。

[https://github.com/rhiroe/features/blob/main/src/saml2aws/install.sh](https://github.com/rhiroe/features/blob/main/src/saml2aws/install.sh)

### テスト

作ったDevContainer Featuresが期待通り動作するかテストが必要です。テストはtestディレクトリにサンプルがあるので、それを変更もしくは参考に書き始めることができます。

テストが書けたら`devcontainer features test`コマンドでテストを実行します。「`devcontainer`コマンドなんてない」？ご安心ください。コピーしたテンプレートにはちゃんと`.devcontainer/devcontainer.json`が含まれています。devcontainerを起動すれば、その中には必要なものが一通り揃っています。

```
$ devcontainer features test -f ${feature_name} -i ${base_image} .

```

### CI/CD

コピーしたテンプレートにはCI/CDまでカバーされており、他に何も手を加えずとも変更をPushすればGitHubActionsでテストが実行され、ドキュメントが更新されます。また任意で手動実行可能なリリース用のActionも用意されており、GutHub上からポチポチするだけでリリースまで完了します。それぞれファイルは`.github/workflows`にあります。

### 完成

リリースまで行えば完了です。

あとは自分たちの開発しているリポジトリの`.devcontainer/devcontainer.json`の`features`に追加して使ってあげましょう。

また、[このファイル](https://github.com/rhiroe/devcontainers.github.io/blob/gh-pages/_data/collection-index.yml)に対して、作ったDevContainer Featuresを追加するPRを出せば、下記ページで公開されます。

[https://containers.dev/features](https://containers.dev/features)

# 最後に

以前からDevContainerのFeaturesはとても便利な機能だなと思っていたが、それを簡単に作ることもできるということを今回知ることができた。アプリケーションを動かすためには直接関係ないが、開発環境を構築する際に必要となるソフトウェアをインストールする必要がある、みたいな場面はこれまでもそれなりに遭遇してきたことがあるが、そういったものがDevContainer Features化されると、新しくJoinしてくるメンバーに優しい環境が出来上がるのではないかと思っています。

タイミーでも実際にインターン生を受け入れる際に、環境構築が容易になって助かっているというフィードバックを受けています。

DevContainer自体もとても便利な機能ですが、もう一歩踏みこんでDevContainer Featuresも積極的に便利に使われている世界になると嬉しいなと思っています。

# 蛇足

タイミーでは一緒にはたらく仲間を積極的に募集しています。もし、タイミーという会社に興味を持っていただけましたら、下記リンクからぜひカジュアル面談でお話ししましょう。

[https://product-recruit.timee.co.jp/casual](https://product-recruit.timee.co.jp/casual)
