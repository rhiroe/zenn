---
title: 【Rails6.1】ActiveRecord::Relationメソッドでの安全でない生SQLの利用（非推奨）を削除への対応
date: 2021-12-10
tags: ["Rails"]
excerpt: TracePoint便利
---

# ことの始まり

Rails6.1の変更点の一つに

> ActiveRecord::Relation メソッドでの安全でない生SQLの利用（非推奨）を削除

というものがある。わかりづらいが要は以下のようなものだ。

```ruby
Model.order('COALESCE(foo, 0)')
# ActiveRecord::UnknownAttributeReference: Query method called with non-attribute argument(s): "COALESCE(foo, 0)"
```

6.0まではこれが警告を出すだけだったが、6.1からは例外を発生するようになった。

# 本題

Rails6.1へのアップデートにおいてこれを解決する必要があった。
Rails.6.0の状態でも

```ruby
ActiveRecord::Base.allow_unsafe_raw_sql = nil # :deprecated 以外の値
```

とすることで、6.1の例外を発生させる挙動にすることができる。とりあえずこれでテストを流してみることにした。

# テストは落ちるが例外が発生しない

そう、テストは落ちているが、例外が発生していないように見えるのだ。テストが落ちているのでどこかで上記の影響を受けているはずなのだが、ログには例外が起きた形跡がない。

テストが落ちる原因を調査すると、確かに上記の例外が起きていたのだが、delayed_jobで実行していたために、ここでのエラーでテストが失敗していなかった。

これだと、テストが落ちた部分を直すだけでは見逃す可能性があるので怖い。

# TracePointでもみ消された例外も拾う

これが本当の本題。テスト結果からわからない例外の内、`ActiveRecord::UnknownAttributeReference`のものをファイルに出力する処理を追加した上でテストを流すことにした。

```ruby
RSpec.configure do |config|
  config.around do |e|
    trace = TracePoint.trace(:raise) do |tp|
      if ActiveRecord::UnknownAttributeReference === tp.raised_exception
        File.open('tmp/errors.txt', 'a') { |f| f.puts [tp.inspect, tp.raised_exception, nil] }
      end
    end

    trace.enable
    e.run
    trace.disable
  end
end
```

これで、「エラーが起きたがテストは継続され、結果がたまたま同じ」というケースにも気づくことができる。テストカバレッジ次第だが、これで修正が必要な部分をもれなく拾い上げることが可能になった。

見つけた場所はサニタイズの処理を追加して対応した。

```ruby
ApplicationRecord.sanitize_sql_array(...)
```

余談だが`ActiveRecord::Sanitization`に定義されているメソッドを見ると、いろんなサニタイズの手段が提供されていて面白い。

終わり。
