---
title: RubyistがJavascriptの基礎学習をしてみる
date: 2020-3-31
tags: ["Javascript"]
excerpt: Javascript何もわからん
---

Rubyしか書けないし、Rubyしか書いてこなかったので、
それ以外の言語も触れるようになりたいなと思って、
リファレンス等を見ながら何となくでやってたところをしっかり理解しようという意図。

学習しながら書き足しているのでメモみたいなもの。

あまり参考にしないほうがいいかもしれない。

というか今までJavascriptよくわかってないのによく業務こなせてたな...。

雰囲気でコード書いてたのでウンコード量産してそう、すまん。

# クラス

そもそもJavascriptのRubyでいうクラスメソッドみたいなのがよくわかっていなかった。

```javascript
Example.foo();
```
みたいなやつ。
Javascriptって関数以外にメソッドも定義できるの？くらい何もわかってない。

## クラス宣言とクラス式

クラスを用意するにはクラス宣言とクラス式の2つの定義方法があるみたい。

```javascript
// クラス宣言
class Rectangle {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
}

// クラス式
let Rectangle = class {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
};
```

Rubyでいうところの以下みたいなものだろうか。

```ruby
# クラス宣言にあたる記法
class Name; end

# クラス式にあたる記法
Name = Class.new
```

多分おそらくだけどクラス宣言で定義するのが一般的だと思うので、そちらに絞って学習を続ける。

ちなみにどちらもホイスティング問題というのがあるらしい。

> クラスにアクセスする前に、そのクラスを宣言する必要があります。そうしないと、ReferenceError がスローされます

## クラス本体の記述

クラス本体は[Strictモード](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Strict_mode)というもので実行されるらしい。
通常エラーにならないけど、バグや落とし穴になりそうなところでエラーが起きるようになるものみたい。

あとはちょっと高速だったり、定義予定の構文を禁止(将来の予約後となる名前の使用禁止ってこと？)したりするとか。

要は厳密なコードを書く必要がありますよ、といったところだろうか。

### constructor

> `class`によって生成されるオブジェクトの生成や初期化を行う特別なメソッドです。

Rubyでいうところの`initialize`メソッドのことかな？

各クラスに1つしか定義できず、2回以上定義されると`SyntaxError`を発生させる。

```javascript
class Name {
    constructor(){}
    constructor(){}
}
// Uncaught SyntaxError: Classes may not have a field named 'constructor'
```

```ruby
class Name
  def initialize; end
  def initialize; end
end
# SyntaxOK
```

Rubyは何度でも定義できるのでちょっと差異があるけど、同じようなものという認識でいていいかな。

```javascript
class Name {
    constructor(foo){console.log(foo)}
}
new Name('Hello Javascript')
// Hello Javascript
```

```ruby
class Name
  def initialize(foo)
    puts foo
  end
end
Name.new('Hello Ruby')
# Hello Ruby
```

戻り値の違いはあるものの同じ感じ。

### プロトタイプメソッド

サンプル
```javascript
class Rectangle {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
  // ゲッター
  get area() {
    return this.calcArea();
  }
  // メソッド
  calcArea() {
    return this.height * this.width;
  }
}

const square = new Rectangle(10, 10);

console.log(square.area); // 100
```

まぁ、サンプルパッと出されても何もわからないのでRubyに書き直して咀嚼する。

```ruby
class Rectangle
  def initialize(height, width)
    @height = height
    @width = width
  end
  
  attr_reader :height, :width
  
  def area
    calc_area
  end
   
  def calc_area
    height * width
  end
end

square = Rectangle.new(10, 10)

print square.area # 100
```

こんな感じかな。ゲッターとある部分を単なるメソッドにしているのが意味合いを変えてしまっている気がしないでもないが...。

でもRubyのゲッターも単なるメソッドではあるので間違いではないか。

`this`をどう解釈すればいいのかわからない、Rubyの`self`ではなさそう...。

### thisってなんだ

ちょっと脱線するけど、`this`がよくわからないので調べてみる。

`this`って使われるタイミングで中身が左右されるみたいなのをみたことがある。

1. メソッド呼び出しパターン
2. 関数呼び出しパターン
3. コンストラクタ呼び出しパターン
4. apply,call呼び出しパターン

#### メソッド呼び出しパターン

メソッドの中で使われる`this`のパターン。

```javascript
const myObject = {
    value: 10,
    output: function() {
        console.log(this.value);
    }
};

myObject.output(); // 10
```

このサンプルはさっきまでの学習で知ったメソッドの定義と少し違うな...。

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value);
    }
};

myObject.output(); // 10
```

これでも期待した値が出力できるのでSyntaxSugarなのかな。

後者が前者の簡略構文っぽい。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Functions/Method_definitions

結果から見るに、メソッド内の`this`はそのメソッドが定義されているオブジェクト自身を参照するみたい。

#### 関数呼び出しパターン

関数の中の`this`はグローバルオブジェクトを参照するみたい。

グローバルオブジェクトのプロパティーに値をセットするとグローバル変数になるとか。

Javascriptは変数のスコープとかアクセスの種類とか覚えるのが難しそうだ。

```javascript
function output(){
    console.log(this); // グローバルオブジェクト
    this.value = 1;
    console.log(this.value); // 1
}
output();

console.log(this.value); // 1
console.log(value); // 1
```

ここでJavascriptできない僕が気になったのが

```javascript
const myObject = {
    value: 10,
    function output() {
        console.log(this.value)
    }
    output();
};
```

オブジェクトの中に直接関数定義した場合の`this`。

Javascriptちょっとわかる人ならすぐにわかると思うけど、これはSyntaxErrorだった。

オブジェクトの中に関数は直接定義できないみたい。

ただし...

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value); // 10
        
        function output(){
            console.log(this.value); // undefined
            this.value = 1;
            console.log(this.value); // 1
        }
        output();
    }
};

myObject.output();
console.log(value); // 1
```

みたいなオブジェクトの中のメソッドの中には関数を定義できる。

関数の中の`this`はたとえそれがメソッドの中であってもグローバルオブジェクトを参照するみたい。

難しいなぁ...。

もし、メソッド内の関数でオブジェクト自体を参照したい場合は、オブジェクトを参照している時点の`this`を変数に入れて持っておくことが有効みたい。

その際によく使われる変数名が`self`,`that`,`_this`らしい。

`seif`はブラウザ上だと`window.self`を指すらしくて、それを上書きしてしまうのはどうなのかなという気もしないでもない。

とはいえ最も使われる変数名は`self`みたいなのでそれに合わせたほうが良さそう。

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value); // 10
        const self = this;
        function output(){
            console.log(self.value); // 10
        }
        output();
    }
};

myObject.output();
console.log(this.value); // undefined
```

ちなみに関数の定義は他にもアロー関数というものがあって、その中で使われる`this`は関数呼び出しのパターンに当てはまらないみたい。

アロー関数の中の`this`はその親の`this`と等価みたい。

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value); // 10
        output = () => {
            console.log(this.value); // 10
        };
        output();
    }
};

myObject.output();
console.log(this.value); // undefined
```

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value); // 10
        getValue = () => { return this.value };
        function output(value) {
            console.log(value);
        };
        output(getValue()); // 10
    }
};

myObject.output();
```

```javascript
const myObject = {
    value: 10,
    output() {
        console.log(this.value); // 10
        function output() {
            getValue = () => { return this.value };
            console.log(getValue()); // undefined
        };
        output();
    }
};

myObject.output();
```

アロー関数の`this`は定義された場所から見て親の`this`と同じものを参照するみたい。

実行された場所は関係ない。

アロー関数は構文も含めてちょっと難しいな...。

#### コンストラクタ呼び出しパターン

よくわからないのでまずはサンプルから

```javascript
function MyObject(value) {
  this.value = value;
  this.increment = function() {
    this.value++;
  };
}
let myObject = new MyObject(0);
console.log(myObject.value); // 0

myObject.increment();
console.log(myObject.value); // 1
```

...なんだこれは、Javascriptはクラス以外も`new`できるのか。

```javascript
class MyObject {
    constructor(value) {
        this.value = value;
        this.increment = function() {
          this.value++;
        };
    }
}
let myObject = new MyObject(0);
console.log(myObject.value); // 0

myObject.increment();
console.log(myObject.value); // 1
```

これと等価っぽい。わからんけど。等価だと言ってくれ。

Javascript...わけがわからないな...。

関数定義であってもインスタンス化すると関数定義内の`this`はインスタンス化された関数自身を指すようになるみたい。

#### apply,call呼び出しパターン

`this`はこれを参照してね、というのを第一引数で指定してメソッドを実行できるものみたい。

```javascript
let myObject = {
  value: 1,
  output: function() {
    console.log(this.value);
  }
};
let yourObject = {
  value: 3
};

myObject.output(); // 1

myObject.output.apply(yourObject); // 3
myObject.output.call(yourObject); // 3
```

関数に対しては使えないのかな...？

```javascript
const myObject = {
  value: 1
};

function output(){
    console.log(this.value);
}

output.call(myObject) // 1
```

使えた。ふーん便利。

`apply`と`call`の違いは第2引数以降の渡し方で、`apply`は配列で第2引数に丸ごと渡して、`call`は第2引数以降をそれぞれ渡すみたい。

### ゲッター

> `get`構文は、オブジェクトのプロパティを関数に結びつけ、プロパティが参照された時に関数が呼び出されるようにします。

まぁ当然(？)ながら「プロパティ」という概念すら雰囲気でやってきたのでよくわかっていないからそこから

プロパティ

> JavaScript のオブジェクトは、自身に関連付けられたプロパティを持ちます。オブジェクトのプロパティは、オブジェクトに関連付けられている変数と捉えることができます。オブジェクトのプロパティは、オブジェクトに属するものという点を除けば、基本的に通常の JavaScript 変数と同じようなものです。

Rubyでいうインスタンス変数みたいなものかな。

```javascript
class Sample {
    constructor(){
        this.foo = "Hello";
    }
}

let obj = new Sample;
console.log(obj.foo);
// Hello
```

```ruby
class Sample
  def initialize
    @foo = "Hello"
  end
  
  attr_reader :foo # Rubyだと呼び出しのためのメソッド定義が必要
end

print Sample.new.foo
# Hello
```

```javascript
let obj = {
    foo : "Hello"
};
console.log(obj.foo);
// Hello
```

```ruby
obj = Class.new
obj.instance_variable_set('@foo', "Hello")
print obj.instance_variable_get('@foo')
# Hello
```

で、ゲッターの話に戻るけど、例のごとくサンプルをRubyに書き換えて咀嚼。

```javascript
const obj = {
  log: ['a', 'b', 'c'],
  get latest() {
    if (this.log.length == 0) {
      return undefined;
    }
    return this.log[this.log.length - 1];
  }
};

console.log(obj.latest);
// expected output: "c"
```

```ruby
obj = Class.new do
  @log = ['a', 'b', 'c']

  class << self

    attr_reader :log

    def latest
      return if log.size == 0
      
      log[log.size - 1]
    end
  end
end

print obj.latest
# expected output: "c"
```

こんな感じかな。

ゲッターとメソッドとの違いがいまいちよくわからない。

頭に`get`をつけることで、呼び出し元で`()`を省略できますよってだけなんだろうか。

### セッター

全くわからないけど、流れ的にRubyの`attr_writer`的なものかなとみる。

例のごとくサンプルをRubyに書き換えて咀嚼。

```javascript
const language = {
  set current(name) {
    this.log.push(name);
  },
  log: []
};

language.current = 'EN';
language.current = 'FA';

console.log(language.log);
// expected output: Array ["EN", "FA"]
```

```ruby
language = Class.new do
  @log = []

  class << self
  
    attr_reader :log

    def current=(name)
      @log << name
    end
   end
end

language.current = 'EN'
language.current = 'FA'

print language.log
```

こんな感じか。Rubyで書くと無理やりクラスを登場させているのでなんとなくわかりづらい感じもするが...。

これは個人のメモみたいなものなので気にせずにいこう。

話は戻ってJavascriptのゲッターとメソッドの違いを理解しないといけない。

ゲッターはRubyのメソッド呼び出しっぽく`()`を省略して呼べる。

メソッドはRubyっぽく呼び出すと、関数がそのまま返ってくる。

ちゃんと後ろに`()`をつけないと処理が行われない。

って感じかなぁ...。単に呼び出し方がちょっと違うだけなのかもしれない。

```javascript
class Sample {
    method() {
        console.log("Hello Javascript")
    }
}

obj = new Sample;
obj.method;

// ƒ method() {
//     console.log("Hello Javascript")
// }

obj.method();
// Hello Javascript
```

現にゲッターのサンプルは`get`使わなくても

```javascript
const obj = {
  log: ['a', 'b', 'c'],
  latest() {
    if (this.log.length == 0) {
      return undefined;
    }
    return this.log[this.log.length - 1];
  }
};

console.log(obj.latest()); // c
```

で期待した値を出力できるしなぁ...。

### 静的メソッド

`static`キーワードでクラスに静的なメソッドを定義できるみたい。

> 静的メソッドは、クラスのインスタンス化なしで呼ばれ、インスタンス化されていると呼べません。

インスタンス化なしで呼ぶということは、これがRubyでいうクラスメソッドにあたるのかな。

いつもの

```javascript
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }

  static distance(a, b) {
    const dx = a.x - b.x;
    const dy = a.y - b.y;

    return Math.hypot(dx, dy);
  }
}

const p1 = new Point(5, 5);
const p2 = new Point(10, 10);
p1.distance; //未定義
p2.distance; //未定義

console.log(Point.distance(p1, p2)); // 7.0710678118654755
```

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end
  
  attr_reader :x, :y

  class << self
    def distance(a, b)
      dx = a.x - b.x
      dy = a.y - b.y

      Math.hypot(dx, dy)
     end
  end
end

p1 = Point.new(5, 5)
p2 = Point.new(10, 10)
# p1.distance # NoMethodError
# p2.distance # NoMethodError

print Point.distance(p1, p2) # 7.0710678118654755
```

`Math.hypot(dx, dy)`のところとか全く同じものが使えるの面白い。

最初に睨んだ通り、静的メソッドはRubyのクラスメソッドみたいなものだった。

### プロトタイプと静的メソッドによるボクシング

ボクシングっていうのがよくわからなかったので調べた。

Javascriptにはオブジェクト型とプリミティブ型の2種類があるらしい。

プリミティブ型は全6種類
- 文字列(`'文字列'`)
- 数値(`3.14`)
- 真偽地(`true`)
- シンボル(`Symbol()`)
- Null値(`null`)
- 未定義(`undefined`)

これらはメソッドやプロパティを持たない。

対してRubyは全てがオブジェクトであり、全てのオブジェクトはメソッドを持っているので馴染みづらいな。

で、ボクシングというのはこれらプリミティブ型をオブジェクト型に変換してくれることを指すみたい。

ボクシングすることでオブジェクト型のように扱うことができ、メソッドやプロパティーを持たせることができる。

で、話は戻ってプロトタイプと静的メソッドによるボクシングの話になるが、クラスを書くときのStrictモードでは、
このボクシングが自動で行われないみたい。

なので、`this`の値が`undefined`の場合、メソッド内でも`undefined`のままらしい。

クラスの外はStrictモードではないので、自動ボクシングが行われる。

自動ボクシングの際、最初の`this`が`undefined`の場合、`this`にはグローバルオブジェクトが入ります。

```javascript
function Animal() { }

Animal.prototype.speak = function() {
  return this;
};

Animal.eat = function() {
  return this;
};

let obj = new Animal();
let speak = obj.speak;
speak(); // グローバルオブジェクト

let eat = Animal.eat;
eat(); // グローバルオブジェクト
```

クラスの外でも明示的にStrictモードを使用している場合は、自動ボクシングが行われないので`undefined`のまま。

```javascript
'use strict';

function Animal() { }

Animal.prototype.speak = function() {
  return this;
};

Animal.eat = function() {
  return this;
};

let obj = new Animal();
let speak = obj.speak;
speak(); // undefined

let eat = Animal.eat;
eat(); // undefined
```

### インスタンスのプロパティ

> インスタンスのプロパティはクラスのメソッドの中で定義しなければなりません

```javascript
class Rectangle {
  constructor(height, width) {    
    this.height = height;
    this.width = width;
  }
}
```

ということは、後からプロパティーを追加できないってことだろうか？

```javascript
class Rectangle {}

let obj = new Rectangle;
obj.height = 10;
obj.width = 20;
console.log(obj); // Rectangle {height: 10, width: 20}
```

...できるじゃん。これはどういうことなんだ？

クラスのインスタンスのプロパティはメソッド内で定義しないとSyntaxErrorになっちゃいますよってことかな...。

`this`の話に付随するところがありそう。

> クラスに付随する静的なプロパティやプロトタイプのプロパティは、クラス本体の宣言の外で定義しなければなりません

```javascript
class Rectangle {}
Rectangle.staticWidth = 20;
Rectangle.prototype.prototypeWidth = 25;
```

まぁこれもそうだろうなって感じ。

ただプロトタイプのプロパティってなんだろう...。

> プロトタイプは JavaScript オブジェクトが機能を互いに継承するメカニズムです。

へぇ〜

```javascript
Object.prototype.width = 50;
class Rectangle {}
Rectangle.width; // 50
Rectangle.prototype.width; // 50
Rectangle.prototype.height = 100;
Object.height; // undefined
Object.prototype.height = 200;
Rectangle.height; //200
Rectangle.prototype.height; // 100
```

なんとなくわかったけど、継承元の静的なプロトタイププロパティが継承先の静的なプロパティやプロトタイププロパティとして使えるっぽい。

で、もちろん上書きができるけど、プロトタイププロパティはそのインスタンスのプロパティには影響しないのか。

```javascript
class Object{
    constructor() {    
        this.prototype.height = 10;
    }
}
class Rectangle {}
let obj = new Rectangle;
obj.width; // undefined
```

インスタンスプロパティには使えないっぽい。

一旦おしまい。続きはまた今度。
