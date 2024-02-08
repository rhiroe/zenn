---
title: ActiveModel::Attributesを使ってみたら非常に便利だった。
date: 2019-6-19
tags: ["Rails"]
excerpt: ActiveModel::Attributes は Rails 5.2 から使えます。
---

## 前置き
皆さんはAttributesAPIをご存知だろうか。  
AttributesAPI自体はRailsの標準機能だが、意識して使っている人はそう多くないと思う。  
本題に入る前にAttributesAPIについて軽くお話ししておく。  

### AttributesAPIとは
> What is attribute API?  
Attribute API converts the attribute value to an appropriate Ruby type.

既存のアトリビュートの値を適切な型に変換したり、任意のアトリビュートを定義出来る機能です。  
この機能はRails5でActiveRecordに追加されました。  
言葉で説明してもわかりづらいと思うのでコードで説明しましょう。
```ruby
create_table "users", force: :cascade do |t|
  t.string   "name"
  t.string   "age"
end
```
```ruby
class User < ApplicationRecord; end
```
このようなテーブルが存在したとします。  
テーブルの`age`カラムはString型なので`Book`クラスの`age`アトリビュートも当然String型になりますね。
```ruby
User.new(name: '山田太郎', age: '18').name
#=> "18"
```
この`age`の値をInteger型で扱う必要が出てきました。  
ただ、テーブルのカラムの属性自体を変えるのは骨が折れますよね？  
カラムから取り出した値に`.to_i`をいちいち行うのもイケてませんし、  
そもそもIntegerとして扱いたいのにStringがそのまま保存されてしまうのも不安です。  
そこで、ですよ。`attribute`の出番です。  
```ruby
class User < ApplicationRecord
  attribute :age, :integer
end
```
```ruby
User.new(name: '山田太郎', age: '18').name
#=> 18
```
文字列が数字に変換されています。  
もちろんDBの型はStringのままなのでデータベースにはString型で保存されます。  
そして、String型のデータを呼び出した時、Integer型に変換されて出てきます。  
つまり今回の`age`カラムに"18"を保存して再度呼び出した時の流れは...
```ruby
# "18" から 18 に変換されてモデルインスタンスが生成される
user = User.new(name: '山田太郎', age: '18')

# 18 から "18" に変換されてString型のカラムに保存される
user.save

# String型のカラムから取り出した "１８" から 18 に変換されて呼び出される
User.find_by(name: '山田太郎').age
#=> 18
```
といった具合になります。  
他にもカスタム型を定義することができます。  
コードで例をあげると
```ruby
create_table "products", force: :cascade do |t|
  t.string    "name"
  t.integer   "price"
end
```
```ruby
class Product < ApplicationRecord
  attribute :price, :yen
end
```
```ruby
# config/initializers/types.rb
class YenType < ActiveRecord::Type::String
  def cast(value)
    super "#{value}円"
  end
end

ActiveRecord::Type.register(:yen, YenType)
```
```ruby
Product.new(price: 1980).price
#=> "1980円"
```
こんな感じですね。まぁ、こんな変換はするべきではないですが...。  
あと、この型変換、`where`に対しては何もしてくれないので  
うまくいかない場合は同じ要領で`serialize`メソッドを使ってカスタマイズしましょう。

とまぁ随分長々と前置きを語りましたが、これがAttributesAPIの力です。  
非常に強力で便利ですよね？

## ActiveRecordでしか使えなかったattributeがActiveModelで使えるようになった！
ここからが本題です。  
全く同じ性能...というわけではありませんが、  
似た性能のものがActiveModel::Attributesとして追加されました。  

### どうやって使うの？
```ruby
class Model
  include ActiveModel::Model
  include ActiveModel::Attributes
end
```
これだけです。  
`ActiveModel::Model`と`ActiveModel::Attributes`を`include`するだけです。  
あとは同じように書いて同じように使えます。
```ruby
class Model
  include ActiveModel::Model
  include ActiveModel::Attributes
  attribute :column, :string
  attribute :num, :integer
end
```
`ActiveModel::Validations`も追加で`include`すればバリデーションも設定できます。
```ruby
class Model
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Validations
  attribute :column, :string
  attribute :num, :integer
  validates :column, presence: true
end
```
加えてカスタム型定義もできます。

### ActiveModel::Attributesでできないこと
ただ、やはり完全に同じものというわけではないのでできないこともあります。  
ActiveRecordでは以下のようなことができていました。
```ruby
class Model
  attribute :float_range, :float, range: true
end
```
```ruby
Model.new(float_range: '[1, 2.5]').float_range
#=> 1.0..2.5
```
これをActiveModel::Attributesで行うことができません。  
具体的には`:range`や`:array`,`:json`などを指定できないのです。  
ただし、扱えないということはなく
```ruby
class Model
  include ActiveModel::Model
  include ActiveModel::Attributes
  attribute :column, hogehuga: :foobar
end
```
のように何の意味もないキーに何の意味もないシンボルをつけると  
Array, Rangeに限らず何でも入るようになります。
```ruby
Model.new(column: [1..2.3, true, "hoge"]).column
#=> [1..2.3, true, "hoge"]
```
ただ、代わりに型変換もされなくなりますし、  
どう見てもバグってるので使うのはお勧めしません。

あと、クラスを書いたファイルを置く場所はよく考えないといけませんね。  
何も考えず`app/models`直下に置くとチームで混乱を招く恐れもあります。  
テーブルを持たないクラスを`models`に置くべきではないとか...

### どういう時に使えるの？
- データベースにテーブルを持ってないけれど、フォームから送信される値のバリデーションをしたい
- フォームから送信される文字列のデータを`to_i`や`to_f`など使うことなく自動で変換して欲しい

こんな経験はないでしょうか？  
そう、まさにこんな時こそActiveModel::Attributesの出番です。

## 最後に
意外と使ってみないと便利さみたいなのが伝わらないかもなぁとも思いますが  
痒いところに手が届くような機能で値の正当性を担保したい場合に非常に便利だと思っています。  

是非是非皆さんも使ってみてください。
