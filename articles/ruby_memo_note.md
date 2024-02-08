---
title: Rubyで遊ぶ
date: 2020-11-25
tags: ["Ruby"]
excerpt: 雑に書きなぐっているだけ
---

記事内のコードの実行にはRuby2.7以上が必要。

```ruby
p `ruby -v`
```

### filter_map

- 結果となる値が falsey な値の場合は条件式の結果にかかわらず除外される。

```ruby
p [*1..10].filter_map { _1 if _1.even? }
#=> [2, 4, 6, 8, 10]

p [*1..10].filter_map { nil if _1.even? }
#=> []

p [*1..10].filter_map { false if _1.even? }
#=> []
```

### case in

- パターンマッチで使う変数をそれ以前に使っていた場合は上書きされる
- パターンマッチ文の外に出てもパターンマッチで代入された値は維持される

```ruby
a = 'Hello'

case [1, 2, 3]
in a
  p a # [1, 2, 3]
end


case [1,2,3]
in a, *b
  p a # 1
end

p a # 1
```

### numbered parameter

- ネストして使えない

```ruby
[1,2,3].tap { p _1.tap { p _1 } }
# SyntaxError ((irb):1: numbered parameter is already used in)
# (irb):1: outer block here

# ちなみにブロックパラメータを取ればネスト可能
[1,2,3].tap { |x| p x.tap { |x| p x } }
# [1, 2, 3]
# [1, 2, 3]
  
# 1回までならネスト中に numbered parameter の使用が可能
[1,2,3].tap { |x| p x.tap { p _1 } }
# [1, 2, 3]
# [1, 2, 3]
```

### instance_eval

- インスタンス変数やプライベートメソッドを無理やり呼び出せる

```ruby
class Sample
  def initialize
    @hello = 'Hello'
  end
  private def ruby
    'exciting'
  end
end
  
Sample.new.ruby
# NoMethodError (private method `ruby' called for #<Foo:0x00007f881e973aa8 @hello="Hello">)

p Sample.new.instance_eval('ruby')
#=> "exciting"
p Sample.new.instance_eval('@hello')
#=> "Hello"

# インスタンス変数の参照に関しては instance_variable_get の方がオススメ。デバッグ目的ならどっちでもよさそう。
p Sample.new.instance_variable_get(:@hello)
```

### protected と private

- 差異わかりますか？

```ruby
class Sample
  def call_protected_method(other)
    other.protected_method
  end
  def call_private_method(other)
    other.private_method
  end
  protected def protected_method; end
  private def private_method; end
end

p Sample.new.call_protected_method Sample.new
#=> nil
p Sample.new.call_private_method Sample.new
# NoMethodError (private method `private_method' called for #<Sample:0x00007fab892b3ab8>)
```
