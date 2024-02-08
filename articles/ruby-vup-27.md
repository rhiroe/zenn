---
title: Rubyのインストールにはまった
date: 2022-1-18
tags: ["Ruby"]
excerpt: openssl関連でハマったら
---

OpenSSLのバージョンが新しすぎてRubyのインストールに失敗する。  
エラーログからはわかりにくいけど...

```text
Last 10 log lines:
        from ./tool/rbinstall.rb:846:in `block (2 levels) in install_default_gem'
        from ./tool/rbinstall.rb:279:in `open_for_install'
        from ./tool/rbinstall.rb:845:in `block in install_default_gem'
        from ./tool/rbinstall.rb:835:in `each'
        from ./tool/rbinstall.rb:835:in `install_default_gem'
        from ./tool/rbinstall.rb:799:in `block in <main>'
        from ./tool/rbinstall.rb:950:in `block in <main>'
        from ./tool/rbinstall.rb:947:in `each'
        from ./tool/rbinstall.rb:947:in `<main>'
make: *** [do-install-all] Error 1
```
こんな感じになってたら当てはまると思う。  
portable-opensslをインストールしてこれを使おう。

```shell
brew install homebrew/portable-ruby/portable-openssl
echo 'export PATH=/usr/local/opt/portable-openssl/bin:$PATH' >> ~/.zshrc
echo 'export RUBY_CONFIGURE_OPTS="--with-readline-dir=`brew --prefix readline` --with-openssl-dir=`brew --prefix portable-openssl`"' >> ~/.zshrc

source ~/.zshrc
rbenv install 2.7.5
```
