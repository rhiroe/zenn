---
title: RubyでJavaのインターフェースっぽいことをする
date: 2022-3-7
tags: ["Ruby", "joke"]
---

# RubyでJavaのインターフェースっぽいことをする

Rubyにはインターフェースという機能はないのですが、Rubyでインターフェースを実装するならどうするのがいいのか考えてみました。

## インターフェース相当のクラスを用意して、それを継承する

```ruby
class Fee
  def yen
    raise NotImplementedError
  end

  def label
    raise NotImplementedError
  end
end

class AdultFee < Fee
  def yen
    100
  end

  def label
    '大人'
  end
end

class ChildFee < Fee
  def yen
    50
  end

  def label
    '子供'
  end
end
```

真っ先に思いついた実装です。継承元にNotImplementedErrorの例外を起こすメソッドを定義し、継承先でメソッドの実装がないと実行時に例外が起きます。ただし、複数のインターフェースを定義しようとすると、多重継承できない関係でやや使い勝手が悪いです。あとは何でもかんでも継承で解決しようとするのは悪という風潮もありますね。

## インターフェース相当のモジュールを用意して、それぞれのクラスでincludeする

次はクラスではなく、モジュールでインターフェースを表現する方法です。これであればインターフェースが複数あっても複数のモジュールをincludeすればいいので使い勝手は良いはずです。ただし、やはり実行しないと例外が起きないのはインターフェースとして少し違う気もします。

```ruby
module Fee
  def yen
    raise NotImplementedError
  end

  def label
    raise NotImplementedError
  end
end

class AdultFee
  include Fee

  def yen
    100
  end

  def label
    '大人'
  end
end

class ChildFee
  include Fee

  def yen
    50
  end

  def label
    '子供'
  end
end
```

## クラス定義をする段階でメソッドの実装がなければ例外を起こす

クラス定義がされた段階でメソッドがなければ例外が起きて欲しいため、こちらで考えてみます。最終的には以下のような記述で、インターフェースで定義したメソッドが実装されてない場合に例外を起こすような形を目指します。

```ruby
module Fee
  interface :yen, :label
end

class AdultFee
  implements Fee
end

class ChildFee
  implements Fee
end
```

まずは、クラス定義が終了するタイミングで例外を起こす部分を作っていきます。クラスの最後にフックするためにTracePointを使います。

```ruby
class AdultFee
  def self.implements(*modules)
    @implements = modules.flat_map(&:implements)
  end

  implements Fee

  trace = TracePoint.new(:end) do |tp|
    self.instance_variable_get(:@implements).each do |implement|
      next if tp.self.instance_methods.include?(implement)

      tp.disable
      raise NotImplementedError, "Method \"#{implement}\" is not implemented in class \"#{self.name}\"."
    end
    tp.disable
  end

  trace.enable
end
```

Feeモジュールには`implements`というモジュールメソッドがあり、定義すべきメソッド名が返るという想定です。

ちなみに、わざわざTracePointを使うのは目指す形にある`implements Fee`という宣言をクラス定義の始めの方で行いたいからです。

次はこの処理を`Implements`モジュールに切り出してみます。

```ruby
module Implements
  def self.included(base)
    base.extend(ClassMethods)

    trace = TracePoint.new(:end) do |tp|
      base.instance_variable_get(:@implements).each do |implement|
        result = tp.self.instance_methods.include?(implement)
        next if result

        tp.disable
        raise NotImplementedError, "Method \"#{implement}\" is not implemented in class \"#{base.name}\"."
      end
      tp.disable
    end
    trace.enable
  end

  module ClassMethods
    def implements(*modules)
      @implements = modules.flat_map(&:implements)
    end
  end
end

class AdultFee
  include Implements
  implements Fee
end
```

次に、インターフェース部分であるFeeモジュールを作っていきます。

```ruby
module Fee
  def self.interface(*properties)
    define_singleton_method(:implements) { properties }
  end

  interface :yen, :label
end
```

他にインターフェースを作るとして共通化できる部分を`Interface`モジュールに切り出してみます。

```ruby
module Interface
  def interface(*properties)
    define_singleton_method(:implements) { properties }
  end
end

module Fee
  extend Interface
  interface :yen, :label
end
```

では、これまでに作った`Implements`モジュールと`Fee`モジュールを使ってクラスを定義してみます。

```ruby
class AdultFee
  include Implements
  implements Fee
end
# Method "yen" is not implemented in class "AdultFee". (NotImplementedError)

class ChildFee
  include Implements
  implements Fee
end
# Method "yen" is not implemented in class "ChildFee". (NotImplementedError)
```

どうでしょうか。クラス定義の時点でメソッド定義がないとエラーが起きており、記述も最初の目指す形に近付きました。

ただ、これでも問題はあります。クラスの再定義や後からメソッドを削除するのは検知できません。RubyでJavaが提供するインターフェース相当の機能を完璧に表現するのは難しそうでした。

Rubyでインターフェースを実現するためのライブラリがあるのかもしれませんが、パッと調べた限りでは見つけることはできませんでした...。

ちなみに、Ruby3から導入された[RBS](https://github.com/ruby/rbs)でインターフェースの定義ができるようです。Ruby3を利用している場合は、型推論と合わせてこちらを利用するのが良さそうです。ただし、RBSを書いたからと言ってクラス定義でメソッドの実装がなければ例外が起きるとかはなく、[Steep](https://github.com/soutaro/steep)を使って型検査した時に、初めて分かるというものなので、そういう点ではJavaのインターフェースとは違い、強制力は低いので注意です。
