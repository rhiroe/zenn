---
title: EncryptedCredentials をもっと使ってあげて！
date: 2019-6-20
tags: ["Rails"]
excerpt: 君たちRailsの標準機能をもっと使いなよ！
---

## 前置き
何だかTwitterやSlackやブログや何やらで  
「Railsで〇〇作ってみた」  
っていうのをよく見かけるのだが、その中では
> 環境変数を扱うgemのdotenvをインストール

...ってちょっと待った！  
確かに`dotenv`もよくできたgemで以前はよく使われていたけど  
今はRailsに`EncryptedCredentials`っていう  
すごい便利な機能があるじゃないか...

ということで`EncryptedCredentials`について書こうと思う。

## EncryptedCredentialsとは
まずは[このプルリク](https://github.com/rails/rails/pull/30067)を見てくれ。  
簡単にまとめると
- 公開したくない値が色々なところにあって紛らわしい
- `master key`で復号できる暗号化ファイルで一括管理しよう
- 開発テスト環境は`master key`がなくても動くようにするよ

だいたいこんな感じでしょうか。  
これの何がすごいかっていうと...

__秘匿情報を暗号化してGit管理できる__

ことなんですよ。  
つまりチーム開発でそれぞれが全ての秘匿情報を設定せずとも  
`master key`の値さえこっそり共有できればOKなのです。  
まぁ逆に言えば`master key`が割れると丸裸なんですが...  
そんなチョンボする人は流石にいないでしょう！

## 使い方
EncryptedCredentialsはRailsの標準機能なので  
`rails new`した段階で使えます。  
ただしバージョンは5.2以降でないとダメです。  
この機能自体がRails5.2以降のものですからね！

### 暗号化ファイルを編集する
暗号化されたファイルを復号して編集するんですが、  
そのためのエディタを設定しておく必要があります。  
まだ指定していない人は指定しておきましょう。  
以下はエディタに`vi`を指定する場合です。
```sh
# ~/.bash_profile
export EDITOR="vi"
```
編集する場合は以下のコマンドです。
```bash
rails credentials:edit
```
この時、`master key`が存在しない場合、自動で作られます。  
次回以降編集する場合や、アプリケーション内で復号する場合必要になってくるので、間違えて消したりしないようにしましょう。  
もし消してしまった場合、以下の作業のやり直しになります。  
編集画面は以下のように先ほど指定したエディタで開かれているかと思います。
```yaml
# aws:
#   access_key_id: 123
#   secret_access_key: 345

# Used as the base secret for all MessageVerifiers in Rails, including the one protecting cookies.
secret_key_base: 45c202c6f9...
```
サンプルが書いてあるので書き方はわかりやすいですね。

### 暗号化ファイルから値を読み取る
値を読み取るには先ほども言った通り`master key`が必要です。  
`master key`は`config/master.key`にあるので確認してみてください。  
今回はサンプルのコメントアウトを外して呼び出してみます。
```yaml
aws:
  access_key_id: 123
  secret_access_key: 345

# Used as the base secret for all MessageVerifiers in Rails, including the one protecting cookies.
secret_key_base: 45c202c6f9...
```
```ruby
rails console --sandbox
Rails.application.credentials.aws[:access_key_id]
#=> 123
Rails.application.credentials.aws[:secret_access_key]
#=> 345
Rails.application.credentials.secret_key_base
#=> "45c202c6f9..."
```
どうでしょうか、慣れれば非常に簡単だと思います。

`master.key`ファイルを共有できない場合  
環境変数`RAILS_MASTER_KEY`に`master key`の値を渡すことで複合が可能になります。
```bash
export RAILS_MASTER_KEY="..."
```

### 注意点

注意点というほどでもないですが、  
本番環境では以下の設定を有効化することが推奨されます。
```ruby
# config/environments/production.rb
config.require_master_key = true
```
これは`master key`の指定漏れを防ぐための設定で  
`master key`が指定されてない状態でサーバ起動を実行しようとするとエラーが発生します。

```bash
rails server
...
Missing encryption key to decrypt file with. Ask your team for your master key and write it to /xxx/my-app/config/master.key or put it in the ENV['RAILS_MASTER_KEY'].
Exiting
```

いかがでしたでしょうか。  
ググって見つけた記事を丸々参考にして、作ってみるのもいいですが  
Railsの標準機能にも目を向けてみるのも大切です。

意外と知られていない機能ですが、とても便利なのでぜひ使ってみてくださいね。
