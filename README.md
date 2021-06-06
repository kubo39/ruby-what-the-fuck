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

# Float

## `Float#eql?` はあるが approx equal 相当のものがない

むしろ使用頻度の高いもののほうが用意されていない。

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

## `Array#uniq!` `Array#reject!` など異常処理でもないのに値を返したり返さなかったり(正確にはnilを返す)するメソッドが存在する

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_UNIQ

```ruby
arr = [1,1,2]
arr.uniq! #=> [1,2]
arr.uniq! #=> nil
```

破壊的メソッドの返り値を使うべきではないという指摘は正しいが、それであればそもそも値を返すような標準ライブラリのAPIデザインがおかしい。

## `Array#+` `String#+` といった数値演算では加算として扱っている演算子を含んだメソッドを連結に用いている

* ref: https://docs.ruby-lang.org/ja/latest/class/Array.html#I_--2B

* ref: Walter Bright氏のツイート https://twitter.com/WalterBright/status/1378561204050796551

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

# 名前空間

## オープンクラスによる変更は依存ライブラリ含めてすべてのコードが影響範囲になる

このようなコードを書くもしくは依存ライブラリに入っている場合、依存ライブラリ含めてあらゆるコードが影響を受ける。

```ruby
class Array
  def fetch(i)
    at(i)
  end
end
```

これによって本来予期しない場所にまで影響が及ぶことがあり、特に依存ライブラリが突然壊れるようになったときなど原因究明が困難となりうる。

# rspec

## letというメソッド名でcall-by-needになっている

意味不明。

## rspec-parameterizedでテストAPIを意図せず上書きしてしまったときのエラーが意味不明になる

以下の例は多少強引だが、whereでテストAPI(ここでは)を上書きできてしまった場合にどうなるかをみる。

```ruby
require 'rspec-parameterized'

describe 'Bad' do
  where(:case_name, :a, :b, :expect) do
    [
      ['add case1', 1, 1, 2],
      ['add case2', 2, 1, 3]
    ]
  end

  with_them do
    context "case #{params[:a]} + #{params[:b]}" do
      it 'should works' do
        expect { a + b }.to eq expect
      end
    end
  end
end
```

テストAPIは `instance_eval` によって上書きされてしまい、意味不明なエラー内容になる。
`params`や`all_params`を上書きする際には警告が表示されるがRSpecが持っているテストAPIに関しては警告は表示されない。

```console
$ bundle exec rspec bad.rb
FF

Failures:

  1) Bad add case1 case 1 + 1 should works
     Failure/Error: expect { a + b }.to eq expect

     NoMethodError:
       undefined method `to' for 2:Integer
     # ./bad.rb:14:in `block (4 levels) in <top (required)>'

  2) Bad add case2 case 2 + 1 should works
     Failure/Error: expect { a + b }.to eq expect

     NoMethodError:
       undefined method `to' for 3:Integer
     # ./bad.rb:14:in `block (4 levels) in <top (required)>'

Finished in 0.00338 seconds (files took 0.32412 seconds to load)
2 examples, 2 failures

Failed examples:

rspec './bad.rb[1:1:1:1]' # Bad add case1 case 1 + 1 should works
rspec './bad.rb[1:2:1:1]' # Bad add case2 case 2 + 1 should works
```
