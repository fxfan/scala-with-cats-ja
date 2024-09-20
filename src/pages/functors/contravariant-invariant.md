## 反変ファンクターと非変ファンクター Contravariant and Invariant Functors {#sec:functors:contravariant-invariant}

As we have seen, we can think of `Functor's` `map` method as
"appending" a transformation to a chain.
We're now going to look at two other type classes,
one representing *prepending* operations to a chain,
and one representing building a *bidirectional*
chain of operations. These are called *contravariant*
and *invariant functors* respectively.

これまで見てきたように、`Functor` の `map` メソッドが行うのは、変換をチェーンに追加することだと考えることができる。ここでは、他の二つの型クラスを見ていく。ひとつは操作をチェーンに「先行して追加する」もの、もう一つは*双方向の*操作チェーンを構築する。これらはそれぞれ、*反変ファンクター（contravariant functor）*と*非変ファンクター（invariant functor）*と呼ばれている。

<div class="callout callout-info">
*This Section is Optional!*

You don't need to know about contravariant and invariant functors to understand monads,
which are the most important type class in this book and the focus of the next chapter.
However, contravariant and invariant do come in handy in
our discussion of `Semigroupal` and `Applicative` in Chapter [@sec:applicatives].

モナドを理解するのに、反変ファンクターや不変ファンクターについて知っておく必要はない。モナドは本書でもっとも重要な型クラスであり、次章の中心的テーマである。しかし、反変ファンクターや不変ファンクターは、[@sec:applicatives]の章で `Semigroupal` や `Applicative` を論じる際に役立つ。

If you want to move on to monads now,
feel free to skip straight to Chapter [@sec:monads].
Come back here before you read Chapter [@sec:applicatives].

今すぐモナドに進みたい場合は、[@sec:monads]章に飛んでかまわない。[@sec:applicatives]章を読む前に、この節に戻ってくるとよい。
</div>

### 反変ファンクターと `contramap` メソッド Contravariant Functors and the *contramap* Method {#sec:functors:contravariant}

The first of our type classes, the *contravariant functor*,
provides an operation called `contramap`
that represents "prepending" an operation to a chain.
The general type signature is shown in Figure [@fig:functors:contramap-type-chart].

最初に紹介するのは反変ファンクター（contravariant functor）である。この型クラスは `contramap` という操作を提供する。これは、操作をチェーンに「先行して追加する」ことを表す。一般的な型シグネチャは、図 [@fig:functors:contramap-type-chart]


![Type chart: the contramap method](src/pages/functors/generic-contramap.pdf+svg){#fig:functors:contramap-type-chart}

The `contramap` method only makes sense for data types that represent *transformations*.
For example, we can't define `contramap` for an `Option`
because there is no way of feeding a value in an
`Option[B]` backwards through a function `A => B`.
However, we can define `contramap` for the `Display` type class
we discussed in Section [@sec:type-classes:display]:

`contramap` メソッドは、変換を表すデータ型に対してのみ意味をもつ。たとえば、`Option` に対して `contramap` を定義することはできない。`Option[B]` に含まれる値を `A => B` という関数を通して逆方向にフィードバックする方法がないためである。しかし、[@sec:type-classes:display]節で説明した `Display` 型クラスに対しては `contramap` を定義できる。

```scala mdoc:silent
trait Display[A] {
  def display(value: A): String
}
```

A `Display[A]` represents a transformation from `A` to `String`.
Its `contramap` method accepts a function `func` of type `B => A`
and creates a new `Display[B]`:

`Display[A]` は `A` から `String` への変換を表している。`contramap` メソッドは、`B => A` 型の関数 `func` を受け取り、新しい `Display[B]` を生成する。

```scala mdoc:silent:reset-object
trait Display[A] {
  def display(value: A): String

  def contramap[B](func: B => A): Display[B] =
    ???
}

def display[A](value: A)(using p: Display[A]): String =
  p.display(value)
```

#### 演習: Showing off with Contramap

Implement the `contramap` method for `Display` above.
Start with the following code template
and replace the `???` with a working method body:

上記の `Display` に `contramap` メソッドを実装せよ。次のコードテンプレートから始め、`???` を動作する実装に置き換えるとよい。

```scala
trait Display[A] {
  def display(value: A): String

  def contramap[B](func: B => A): Display[B] =
    new Display[B] {
      def display(value: B): String =
        ???
    }
}
```

If you get stuck, think about the types.
You need to turn `value`, which is of type `B`, into a `String`.
What functions and methods do you have available
and in what order do they need to be combined?

行き詰まった場合は、型について考えるとよい。`B` という型の値 `value` を `String` に変換する必要がある。どのような関数やメソッドが利用可能で、それらをどういう順番で組み合わせるべきか考えてみよう。

<div class="solution">
Here's a working implementation.
We call `func` to turn the `B` into an `A`
and then use our original `Display`
to turn the `A` into a `String`.
In a small show of sleight of hand
we use a `self` alias to distinguish
the outer and inner `Displays`:

動作する実装を以下に示す。`func` を使って `B` を `A` に変換し、その後、元の `Display` を使って `A` を `String` に変換する。すこし巧妙なテクニックとして、`self` エイリアスを使い、外側と内側の `Display` を区別している。

```scala mdoc:silent:reset-object
trait Display[A] { self =>

  def display(value: A): String

  def contramap[B](func: B => A): Display[B] =
    new Display[B] {
      def display(value: B): String =
        self.display(func(value))
    }
}

def display[A](value: A)(using p: Display[A]): String =
  p.display(value)
```
</div>

For testing purposes,
let's define some instances of `Display`
for `String` and `Boolean`:

テスト用に、`String` と `Boolean` に対する `Display` インスタンスを定義しよう。

```scala mdoc:silent
given stringDisplay: Display[String] with {
  def display(value: String): String =
    s"'${value}'"
}

given booleanDisplay: Display[Boolean] with {
  def display(value: Boolean): String =
    if value then "yes" else "no"
}
```

```scala mdoc
display("hello")
display(true)
```

Now define an instance of `Display` for
the following `Box` case class.
This is an example of type class composotion
as described in Section [@sec:type-classes:composition]:

次に、下記のような `Box` というcaseクラスに対して `Display` インスタンスを定義せよ。これは[@sec:type-classes:composition]節で言及した型クラスの合成の例である。

```scala mdoc:silent
final case class Box[A](value: A)
```

Rather than writing out
the complete definition from scratch
(`new Display[Box]` etc...),
create your instance from an
existing instance using `contramap`.

`new Display[Box]` などとフルスクラッチで定義を書き出すのではなく、`contramap` を使って、既存のインスタンスから求めているインスタンスを作成すること。

```scala mdoc:invisible
given boxDisplay[A](using p: Display[A]): Display[Box[A]] =
  p.contramap[Box[A]](_.value)
```

Your instance should work as follows:

このインスタンスは次のように利用できる。

```scala mdoc
display(Box("hello world"))
display(Box(true))
```

If we don't have a `Display` for the type inside the `Box`,
calls to `display` should fail to compile:

`Box` の中身の型に対して　`Display` インスタンスが用意されていない場合、`display` 呼び出しはコンパイルに失敗する。

```scala mdoc:fail
display(Box(123))
```

<div class="solution">
To make the instance generic across all types of `Box`,
we base it on the `Display` for the type inside the `Box`.
We can either write out the complete definition by hand:

インスタンスがあらゆる型の `Box` に対して汎用的になるよう、`Box` の中身の型に対応する `Display` インスタンスをベースにする。

以下のように完全な定義を手作業で書き出してもよいし、

```scala mdoc:invisible:reset-object
trait Display[A] {
  self =>

  def display(value: A): String

  def contramap[B](func: B => A): Display[B] =
    new Display[B] {
      def display(value: B): String =
        self.display(func(value))
    }
}
final case class Box[A](value: A)
```
```scala mdoc:silent
given boxDisplay[A](
    using p: Display[A]
): Display[Box[A]] with {
  def display(box: Box[A]): String =
    p.display(box.value)
}
```

or use `contramap` to base the new instance
on the using clause:

もしくは、using句によって解決された `Display` インスタンスをベースに `contramap` を使って新しいインスタンスを定義することもできる。

```scala mdoc:invisible:reset-object
trait Display[A] {
  self =>

  def display(value: A): String

  def contramap[B](func: B => A): Display[B] =
    new Display[B] {
      def display(value: B): String =
        self.display(func(value))
    }
}

def display[A](value: A)(implicit p: Display[A]): String =
  p.display(value)

given stringDisplay: Display[String] =
  new Display[String] {
    def display(value: String): String =
      s"'${value}'"
  }

given booleanDisplay: Display[Boolean] =
  new Display[Boolean] {
    def display(value: Boolean): String =
      if(value) "yes" else "no"
  }
final case class Box[A](value: A)
```
```scala mdoc:silent
given boxDisplay[A](using p: Display[A]): Display[Box[A]] =
  p.contramap[Box[A]](_.value)
```


Using `contramap` is much simpler,
and conveys the functional programming approach
of building solutions by combining simple building blocks
using pure functional combinators.

`contramap` を使う方がはるかにシンプルである。また、　純粋関数型のコンビネータを用いてシンプルな部品を組み合わせることで解決策を構築するという、関数型プログラミングのアプローチをよく表現している。
</div>


### 非変ファンクターと `imap` メソッド Invariant functors and the *imap* method {#sec:functors:invariant}

*Invariant functors* implement a method called `imap`
that is informally equivalent to a
combination of `map` and `contramap`.
If `map` generates new type class instances by
appending a function to a chain,
and `contramap` generates them by
prepending an operation to a chain,
`imap` generates them via
a pair of bidirectional transformations.

*非変ファンクター*は、`imap` というメソッドを実装している。これは大雑把に言えば `map` と `contramap` を組み合わせたようなものである。`map` が関数をチェーンに追加することで、また `contramap` が操作をチェーンの先頭に挿入することで新しい型クラスインスタンスを生成するのに対して、`imap` は双方向の変換ペアを使ってインスタンスを生成する。

The most intuitive examples of this are a type class
that represents encoding and decoding as some data type,
such as Circe's [`Codec`][link-circe-codec]
and Play JSON's [`Format`][link-play-json-format].
We can build our own `Codec` by enhancing `Display`
to support encoding and decoding to/from a `String`:

もっとも直感的な例は、Circeの [`Codec`][link-circe-codec] やPlay JSONの [`Format`][link-play-json-format] のような、あるデータ型についてのエンコードとデコードを表現する型クラスである。`Display` に機能を追加して、`String` との間のエンコードとデコードをサポートすれば、独自の `Codec` を構築できる。

```scala mdoc:silent
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): A
  def imap[B](dec: A => B, enc: B => A): Codec[B] = ???
}
```

```scala mdoc:invisible:reset-object
trait Codec[A] {
  self =>

  def encode(value: A): String
  def decode(value: String): A

  def imap[B](dec: A => B, enc: B => A): Codec[B] =
    new Codec[B] {
      def encode(value: B): String =
        self.encode(enc(value))

      def decode(value: String): B =
        dec(self.decode(value))
    }
}
```

```scala mdoc:silent
def encode[A](value: A)(using c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(using c: Codec[A]): A =
  c.decode(value)
```

The type chart for `imap` is shown in
Figure [@fig:functors:imap-type-chart].
If we have a `Codec[A]`
and a pair of functions `A => B` and `B => A`,
the `imap` method creates a `Codec[B]`:

`imap` の型チャートを図 [@fig:functors:imap-type-chart] に示す。`Codec[A]` と、`A => B` および `B => A` の関数のペアがあれば、`imap` メソッドは `Codec[B]` を生成する。

![Type chart: the imap method](src/pages/functors/generic-imap.pdf+svg){#fig:functors:imap-type-chart}

As an example use case, imagine we have a basic `Codec[String]`,
whose `encode` and `decode` methods both 
simply return the value they are passed:

`Codec` インスタンスの定義例として、最低限の `Codec[String]` を想像してみよう。その `encode` と `decode` メソッドは、渡された値をそのまま返すだけのものである。

具体的な使用例として、`encode` と `decode` が渡された値を単にそのまま返すだけの、基本的な `Codec[String]` を想像してみよう。

```scala mdoc:silent
given stringCodec: Codec[String] with {
  def encode(value: String): String = value
  def decode(value: String): String = value
}
```

We can construct many useful `Codecs` for other types
by building off of `stringCodec` using `imap`:

`imap` を使って、この `stringCodec` を基に、他の型に対する多くの有用な `Codec` を構築することができる。

```scala mdoc:silent
given intCodec: Codec[Int] =
  stringCodec.imap(_.toInt, _.toString)

given booleanCodec: Codec[Boolean] =
  stringCodec.imap(_.toBoolean, _.toString)
```

<div class="callout callout-info">
*Coping with Failure*
*失敗への対処*

Note that the `decode` method of our `Codec` type class
doesn't account for failures.
If we want to model more sophisticated relationships
we can move beyond functors
to look at *lenses* and *optics*.

ここで紹介した `Codec` 型クラスの `decode` メソッドは失敗を考慮していない。データ間のより洗練された関係をモデル化したい場合は、ファンクターの枠を超えて*レンズ*や*オプティクス*を検討することができる。

Optics are beyond the scope of this book.
However, Julien Truffaut's library
[Monocle][link-monocle] provides a great
starting point for further investigation.

オプティクスはこの本の範囲外である。深く知りたい場合、Julien Truffautのライブラリ [Monocle][link-monocle] がすばらしい出発点を提供してくれる。
</div>


#### `imap` を使った変換的思考 Transformative Thinking with *imap*

Implement the `imap` method for `Codec` above.

上述の `Codec` に `imap` を実装せよ。

<div class="solution">
Here's a working implementation:

正しく動作する実装を以下に示す。

```scala mdoc:silent:reset-object
trait Codec[A] { self =>
  def encode(value: A): String
  def decode(value: String): A

  def imap[B](dec: A => B, enc: B => A): Codec[B] = {
    new Codec[B] {
      def encode(value: B): String =
        self.encode(enc(value))

      def decode(value: String): B =
        dec(self.decode(value))
    }
  }
}
```

```scala mdoc:invisible
given stringCodec: Codec[String] =
  new Codec[String] {
    def encode(value: String): String = value
    def decode(value: String): String = value
  }

given intCodec: Codec[Int] =
  stringCodec.imap[Int](_.toInt, _.toString)

given booleanCodec: Codec[Boolean] =
  stringCodec.imap[Boolean](_.toBoolean, _.toString)

def encode[A](value: A)(using c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(using c: Codec[A]): A =
  c.decode(value)
```
</div>

Demonstrate your `imap` method works by
creating a `Codec` for `Double`.

作成した `imap` メソッドが正しく動くことを、`Double` 型に対する `Codec` を定義することで示せ。

<div class="solution">
We can implement this using
the `imap` method of `stringCodec`:

`stringCodec` の `imap` メソッドを使ってこれを実装できる。

```scala mdoc:silent
given doubleCodec: Codec[Double] =
  stringCodec.imap[Double](_.toDouble, _.toString)
```
</div>

Finally, implement a `Codec` for the following `Box` type:

最後に、次の `Box` 型との相互変換を行う `Codec` を実装せよ。

```scala mdoc:silent
final case class Box[A](value: A)
```

<div class="solution">
We need a generic `Codec` for `Box[A]` for any given `A`.
We create this by calling `imap` on a `Codec[A]`,
which we bring into scope using an implicit parameter:

任意の `A` について、`Box[A]` との相互変換を行う汎用的な `Codec` が必要である。暗黙パラメータとしてスコープに導入される `Codec[A]` インスタンスの `imap` を用いることでこれを作成する。

```scala mdoc:silent
given boxCodec[A](using c: Codec[A]): Codec[Box[A]] =
  c.imap[Box[A]](Box(_), _.value)
```
</div>

Your instances should work as follows:

作成したインスタンスは次のように利用できるはずである。

```scala mdoc
encode(123.4)
decode[Double]("123.4")

encode(Box(123.4))
decode[Box[Double]]("123.4")
```

<div class="callout callout-warning">
*What's With the Names?*
*名前に込められた意味*

What's the relationship between the terms
"contravariance", "invariance", and "covariance"
and these different kinds of functor?

「反変」「不変」「共変」という名称とこれらの異なる種類のファンクターにはどのような関係があるのだろうか？


If you recall from Section [@sec:variance],
variance affects subtyping,
which is essentially our ability to use a value of one type
in place of a value of another type
without breaking the code.

[@sec:variance]節を思い出してほしい。変位は部分型関係に影響を与える。部分型関係とは本質的には、ある型の値を、別の型の値が期待されている文脈でコードを破壊することなく使用可能か、という関係性のことを指す。

Subtyping can be viewed as a conversion.
If `B` is a subtype of `A`,
we can always convert a `B` to an `A`.

部分型関係は、変換可能性とみなすことができる。もし `B` が `A` の部分型であれば、常に `B` を `A` に変換できる。

Equivalently we could say that `B` is a subtype of `A`
if there exists a function `B => A`.
A standard covariant functor captures exactly this.
If `F` is a covariant functor,
wherever we have an `F[B]` and a conversion `B => A`
we can always convert to an `F[A]`.

これは、`B => A` 型の関数が存在するとき `B` は `A` の部分型である、と言い換えることができる。共変ファンクターはこの関係性を正確に捉えている。`F` が共変ファンクターであれば、`B => A` という変換があるときは常に `F[B]` を `F[A]` に変換できる。

TODO: ファンクターに付けられた「反変」「不変」「共変」について。`B` を `A` に変換できるということは、`B` は `A` に必要とされるすべての情報をもっており（もしくは生み出せる）、`A` を代替できる。この関係を仮に、`B` は `A` の「変換的部分型」と呼ぶとしたら、`A` と `B` の間の変換的部分型関係とファンクター `F[A]` と `F[B]` の間の変換的部分型関係は、`A` と `B` の部分型関係と高カインド型 `F[A]` と `F[B]` の部分型関係のアナロジーで考えることができる。この発想により、変換関係に変位の概念を持ち込むことができる。と言いたいのだと思う。それとも、部分型関係を変換可能性に基づいて定義する一般的な考え方があるのだろうか。直訳＋訳註にするか、ひととおり訳してから改めて考える。

A contravariant functor captures the opposite case.
If `F` is a contravariant functor,
whenever we have a `F[A]` and a conversion `B => A`
we can convert to an `F[B]`.

Finally, invariant functors capture the case where
we can convert from `F[A]` to `F[B]`
via a function `A => B`
and vice versa via a function `B => A`.
</div>
