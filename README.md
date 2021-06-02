# Ruby WTF!!!!!!

# nil

## NilClass#to_iが0を返す

* ref: https://docs.ruby-lang.org/ja/latest/class/NilClass.html#I_TO_I

```ruby
nil.to_i #=> 0
```

nilはfalsyであり0がtruthyなのでセマンティクスが変わっている。そのため単純なNoMethodErrorよりも意図しないバグを引き起こしやすい上発見が遅れやすい。

これを防ぐには `Kernel#Integer` を常に使うべきであるが、なぜだか知らないがRubyコミュニティは `to_i` を好むようだ。
例外がきてほしくない場面(例:配列の処理で後にcompactするケース)などの場合、Ruby 3.0からは第三引数に例外を送出するかnilを返すか選択可能なオプションがつくので状況が変わるかもしれない。

* ref: https://docs.ruby-lang.org/ja/latest/method/Kernel/m/Integer.html

# 配列

## Array#[] の範囲外アクセスは例外ではなくnil

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_--5B--5D

```ruby
arr = []
arr[100] #=> nil
```

例外にしたい場合は `Array#fetch` を使う。

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_FETCH

```ruby
arr = []
arr.fetch(100) #=> IndexError
```

これまでのところ「`Array#[]` や `Array#at` より `Array#fetch` を使え」という文献をみたことがない。
JavaScriptもだいぶアレな言語仕様だが、~ Good Partsなどアンチパターンに関する(`===`を使えなど)共有は行われている。
これはRubyコミュニティの特性ということなんだろうか。

## Array#[] = で範囲外のインデックスへのアサインを行うと配列を拡張し、間がある場合nilで埋める(fill相当の処理が勝手に行われる)

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_--5B--5D--3D

```ruby
arr = []
arr[4] = 100
arr #=> [nil,nil,nil,nil,100]
```

`Object#freeze` を使うことで防ぐことができるが汎用的な方法ではない。

* ref: https://docs.ruby-lang.org/ja/latest/class/Object.html#I_FREEZE

```ruby
arr = [].freeze
arr[4] = 100 #=> FrozenError
```

## `Array#pop` が値を返す

```ruby
arr = [1,2]
arr.pop #=> 2
arr #=> [1]
```

pop操作中に値を返すような実装では必ず矛盾した構造となるポイントが存在する(整合性が保たれていない)ため例外安全性における基本保証について守ることができない。

* ref: C++ STL設計における例外安全 https://boostjp.github.io/archive/boost_docs/document/generic_exception_safety.html

## `Array#sort` が値を返す

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_SORT

だいたい上に同じ。ただしsortの場合はメソッドチェーンに組み込んで使われるためより悪質。

## `Array#uniq!` `Array#reject!` など異常処理でもないのに値を返したり返さなかったり(正確にはnilを返す)するメソッドが存在する

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_UNIQ

```ruby
arr = [1,1,2]
arr.uniq! #=> [1,2]
arr.uniq! #=> nil
```

破壊的メソッドの返り値を使うべきではないという指摘は正しいが、それであればそもそも値を返すような標準ライブラリのAPIデザインがおかしい。

# ハッシュ

## 連想配列(ハッシュマップ)の名前がハッシュ

Perlに由来しているためにこうなっているが、この命名は奇妙。
作者も失敗だったとどこかで言っていた。

余談だが[Crystalは同じ過ちをおかした](https://crystal-lang.org/reference/syntax_and_semantics/literals/hash.html)、人類は繰り返す。

## `Hash#map` がHashではなくArrayを返す

* ref: https://docs.ruby-lang.org/ja/latest/method/Enumerable/i/collect.html

```ruby
h = { a: 1, b: 2 }
h.map { |_, v| v += 1 } #=> [2, 3]
# h.map { |_, v| v += 1 }.to_h とする必要がある
```

# rspec

## letというメソッド名でcall-by-needになっている

意味不明。
