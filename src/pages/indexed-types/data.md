<!--

## Indexed Data

The key idea of indexed data is to encode type equalities in data.
When we come to inspect the data (usually, via structural recursion) we discover these equalities, which in turn limit what values we can produce. 
Notice, again, the duality with codata. 
Indexed codata limits methods we can call. 
Indexed data limits values we can produce.
Also, remember that indexed data is often known as generalized algebraic data types.
We are using the simpler term indexed data to emphasise the relationship to indexed codata,
and also because it's much easier to type!

Concretely, indexed data in Scala occurs when:

1. we define a sum type with at least one type parameter; and
2. cases within the sum instantiate that type parameter with a concrete type.

Let's see an example. Imagine we are implementing a programming language. We need some representation of values within the language.
Suppose our language supports strings, integers, and doubles, which we will represent with the corresponding Scala types.
The code below shows how we can implement this as a standard algebraic data type.

```scala mdoc:silent
enum Value {
  case VString(value: String)
  case VInt(value: Int)
  case VDouble(value: Double)
}

```

Using indexed data we can use the alternate implementation below.

```scala mdoc:reset:silent
enum Value[A] {
  case VString(value: String) extends Value[String]
  case VInt(value: Int) extends Value[Int]
  case VDouble(value: Double) extends Value[Double]
}
```

This is indexed data, as it meets the criteria above: we have a type parameter `A` that is instantiated with a concrete type in the cases `VString`, `VInt`, and `VDouble`.
It's quite easy to use indexed data in Scala, and people often do so not knowing that it is anything special.
The natural next question is why is this useful?
It will take a more involved example to show why, so let us now dive into one that makes good use of indexed data.


### The Probability Monad 

For our case study of indexed data we will create a probability monad. This is a composable abstraction for defining probability distributions. The probability monad has a lot of uses. The most relevant to most developers is generating data for property-based tests, so we'll focus on this use case. However, it can also be used, for example, for statistical inference or for creating generative art. See the conclusions (Section [@sec:indexed-types:conclusions]) for some pointers to these uses.

Let's start with an example of generating random data. [Doodle][doodle] is a Scala library for graphics and visualization. A core part of the library is representing colors. Doodle has two different representations of colors, RGB and OkLCH, with conversions between the two. These conversions involve some somewhat tricky mathematics. Testing these conversions is an excellent use of property-based testing. If we can generate many, say, random RGB colors, we can test the conversion by checking the roundrip from RGB to OkLCH and back results in the original color[^numerics]. 

To create an RGB color we need three unsigned bytes, so our first task is to define how we generate a random byte. Doodle happens to have an implementation of the probability monad that we will use. Here is how we can do it.

```scala mdoc:silent
import cats.syntax.all.*
import doodle.core.Color
import doodle.core.UnsignedByte
import doodle.random.{*, given}

val randomByte: Random[UnsignedByte] = 
  Random.int(0, 255).map(UnsignedByte.clip)
```

Note that once again we see the interpreter strategy. A `Random[A]` is a value representing a program that will generate a random value of type `A` when it runs.

With three random unsigned bytes we can create a random RGB color.

```scala mdoc:silent
val randomRGB: Random[Color] =
  (randomByte, randomByte, randomByte)
    .mapN((r, g, b) => Color.rgb(r, g, b))
```

We might want to check our code by generating a few random values.

```scala mdoc
randomRGB.replicateA(2).run
```

It seems to be working.

Once we have a source of random data we can write tests using it. We can easily generate more data than is feasible for a programmer to write by hand, and therefore have a higher degree of certainty that our code is correct than we would get with manual testing. The details of writing the tests are not important to us here, so let's move on.

We have seen is an illustration of using the probability monad to generate random data. The probability monad works the same way as every other algebra: we have constructors (`Random.int`), combinators (`map`, and `mapN`), and interpreters (`run`). Being a monad means the algebra has some specific structure. For example, it tells us that we have `pure` and `flatMap` available, from which we can derive `mapN`.

Let's sketch an plausible interface for our probability monad.

```scala mdoc:reset:silent
trait Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B]
}
object Random {
  def pure[A](value: A): Random[A] = ???
  
  // Generate a uniformly distributed random  Double greater
  // than or equal to zero and less than one.
  val double: Random[Double] = ???
  
  // Generate a uniformly distributed random  Int
  val int: Random[Int] = ???
}
```

The interface has the minimum requirements to be a monad, and a few other constructors. We can make progress on the implementation by applying the reification strategy, introduced in Section [@sec:interpreters:reification].

```scala mdoc:reset:silent
enum Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)

  case RFlatMap[A, B](source: Random[A], f: A => Random[B])
      extends Random[B]
  case RPure(value: A)
  case RDouble extends Random[Double]
  case RInt extends Random[Int]
}
object Random {
  import Random.{RPure, RDouble, RInt}

  def pure[A](value: A): Random[A] = RPure(value)

  // Generate a uniformly distributed random  Double greater
  // than or equal to zero and less than one.
  val double: Random[Double] = RDouble

  // Generate a uniformly distributed random  Int
  val int: Random[Int] = RInt
}
```

The next step is to implement an interpreter, which is a standard structural recursion. The interpreter has a parameter that provides a source of random numbers.

```scala
def run(rng: scala.util.Random = scala.util.Random): A =
  this match {
    case RFlatMap(source, f) => f(source.run(rng)).run(rng)
    case RPure(value)        => value
    case RDouble             => rng.nextDouble()
    case RInt                => rng.nextInt()
  }
```

This is an example of indexed data, as the cases `RDouble` and `RInt` provide a concrete type for the type parameter `A`. This means that these cases in the interpreter can produce values of that concrete type. If we did not use indexed data we could only generate values of type `A`, which the programmer would have to supply to use like in the `RPure` case.

To finish this implementation we should implement the `Monad` type class, which would give us `mapN` and other methods for free. However, this is outside the scope of this case study, which is focused on indexed data. I encourage you to do this yourself if you feel you would benefit from the practice.

Note that indexed data can mix concrete and generic types. Let's say we add a `product` method to `Random`.

```scala mdoc:reset:silent
enum Random[A] {
  // ...

  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)

  case RProduct[A, B](left: Random[A], right: Random[B]) extends Random[(A, B)]
  // .. other cases here
}
```

The right-hand side of the `RProduct` case instantiates the type parameter to `(A, B)`, which mixes the concrete tuple type with the generic types `A` and `B`

There are a few tricks to using indexed data that are essential in Scala 2, and can sometimes be useful in Scala 3. Take the following translation of the probability monad into Scala 2. (I've placed a `using` directive in this code, so if you paste it into a file and run it with the Scala CLI it will use the latest version of Scala 2.13.)

```scala mdoc:reset:silent
//> using scala 2.13

sealed trait Random[A] {
  import Random._

  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)

  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)

  def run(rng: scala.util.Random = scala.util.Random): A =
    this match {
      case RFlatMap(source, f) => f(source.run(rng)).run(rng)
      case RProduct(l, r)      => (l.run(rng), r.run(rng))
      case RPure(value)        => value
      case RDouble             => rng.nextDouble()
      case RInt                => rng.nextInt()
    }

}
object Random {
  final case class RFlatMap[A, B](source: Random[A], f: A => Random[B])
      extends Random[B]
  final case class RProduct[A, B](left: Random[A], right: Random[B])
      extends Random[(A, B)]
  final case class RPure[A](value: A) extends Random[A]
  case object RDouble extends Random[Double]
  case object RInt extends Random[Int]

  def pure[A](value: A): Random[A] = RPure(value)

  // Generate a uniformly distributed random  Double greater
  // than or equal to zero and less than one.
  val double: Random[Double] = RDouble

  // Generate a uniformly distributed random  Int
  val int: Random[Int] = RInt
}
```

In Scala 2 this generates a lot of type errors like

```
[error] constructor cannot be instantiated to expected type;
[error]  found   : Random.RProduct[A(in class RProduct),B]
[error]  required: Random[A(in trait Random)]
[error]       case RProduct(l, r)      => (l.run(rng), r.run(rng))
[error]            ^^^^^^^^
```

To solve this we need to create a nested method with a fresh type parameter in the interpreter, as shown below. With this change Scala 2's type inference works and it can successfully compile the code.

```scala
def run(rng: scala.util.Random = scala.util.Random): A = {
  def loop[A](random: Random[A]): A =
    random match {
      case RFlatMap(source, f)   => loop(f(loop(source)))
      case RProduct(left, right) => (loop(left), loop(right))
      case RPure(value)          => value
      case RDouble               => rng.nextDouble()
      case RInt                  => rng.nextInt()
    }

  loop(this)
}
```

The other trick is for when we want to use pattern matches that match type tags. This means the form like

```scala
case r: RPure[A] => ???
```

rather than

```scala
case RPure(value) => ???
```

For cases like `RProduct` it is not clear how to write these pattern matches, as the type parameters `A` and `B` for `RProduct` don't correspond to the type parameter `A` on `Random`. The solution is use lower case names from the type parameters. Concretely, this means we can write

```scala
case r: RProduct[a, b] => ???
```

The type parameters `a` and `b` are existential types; we know they exist but we don't know what concrete type they correspond to. I've found this is occasionally necessary in Scala 2, but very rare in Scala 3.

[^numerics]: Due to numeric issues there may be small differences between the colors that we should ignore.

[doodle]: https://www.creativescala.org/doodle/


```scala mdoc:reset:silent
```
--->

## インデックス付きデータ

インデックス付きデータの中心となるアイデアは、データ内に型の等式をエンコードすることである。それらの等式はデータを検査する際（通常は構造的再帰を通じて）に発見され、それによって生成できる値が制限される。あらためて、余データとの双対性に注目しよう。インデックス付き余データは呼び出せるメソッドを制限し、インデックス付きデータは生成できる値を制限する。インデックス付きデータは一般化代数的データ型と呼ばれることが多いが、ここでは、インデックス付き余データとの関係を強調するために、より簡潔な「インデックス付きデータ」という用語を用いる。ついでに言えば、タイピングが簡単であることもこの用語を用いる理由である[^tn-indexed-types-data-01]。

[^tn-indexed-types-data-01]: 【訳注】 英語で indexed data のほうが generalized algebraic data types よりも短くてタイピングが楽だと言っている。

具体的には、Scala におけるインデックス付きデータは、次のような場合に現れる。

1. 少なくともひとつの型パラメータをもつ直和型を定義し
2. その型のバリアントが型パラメータを具体的な型に固定して定義されている場合

例をひとつ見てみよう。プログラミング言語を実装していると想像してほしい。その言語における値の表現がいくつか必要である。たとえば、対象の言語が文字列、整数、浮動小数点数をサポートするものとしよう。これらは Scala の対応する型を用いて表現することができる。これを標準的な代数的データ型として実装すると以下のようなコードになる。

```scala mdoc:silent
enum Value {
  case VString(value: String)
  case VInt(value: Int)
  case VDouble(value: Double)
}
```

インデックス付きデータを用いれば、これを次のように実装することができる。

```scala mdoc:reset:silent
enum Value[A] {
  case VString(value: String) extends Value[String]
  case VInt(value: Int) extends Value[Int]
  case VDouble(value: Double) extends Value[Double]
}
```

これは上述の条件を満たしているので、インデックス付きデータである。型パラメータ `A` をもち、それがバリアント `VString`、`VInt`、`VDouble` において具体的な型で固定されている。Scala でインデックス付きデータを用いるのはとても簡単で、それを特別なことだとは認識せずにこのようなコードを書くことも多い。これがどのように使えるのかという疑問が続いて生まれるが、それに答えるにはもうすこし手の込んだ例が必要になる。次に、インデックス付きデータをうまく利用している例を詳しく見ていこう。

### 確率モナド

このケーススタディでは確率モナドを構築する。これは確率分布を定義するための合成可能な抽象である。確率モナドには多くの用途があるが、多くの開発者にとってもっとも身近なのはプロパティベースのテストにおけるデータ生成である。ここではその用途に焦点をあてていく。確率モナドは統計的推論やジェネラティブアートの作成などにも利用できる。それらの使い方については本章のまとめである[@sec:indexed-types:conclusions]節を参照してほしい。

まずはランダムデータを生成する例から始めよう。[Doodle][doodle] は Scala のグラフィックおよび可視化ライブラリである。このライブラリの中核のひとつが色の表現である。Doodle には RGB と OkLCH というふたつの色表現があり、それらのあいだに相互変換が定義されている。この変換にはやや込み入った数学が関係しており、その検証にはプロパティベースのテストが非常に有効である。たくさんのランダムな RGB カラーを生成できれば、それを OkLCH に変換して再び RGB に戻した結果が元の色になるかどうかを検証することで、変換の正しさを確かめることができる[^numerics]。

RGB カラーを生成するには三つの符号なしバイトが必要である。そこで、最初の課題としてランダムなバイトの精製方法を定義したい。Doodle には確率モナドの実装が用意されているので、それを利用していこう。

```scala mdoc:silent
import cats.syntax.all.*
import doodle.core.Color
import doodle.core.UnsignedByte
import doodle.random.{*, given}

val randomByte: Random[UnsignedByte] = 
  Random.int(0, 255).map(UnsignedByte.clip)
```

ここにもインタープリタ戦略が登場している点に注目してほしい。`Random[A]` インスタンスは、実行するとランダムな `A` 型の値を生成するプログラムを表す値である。

ランダムな符号なしバイトが三つあれば、ランダムな RGB カラーを生成できる。

```scala mdoc:silent
val randomRGB: Random[Color] =
  (randomByte, randomByte, randomByte)
    .mapN((r, g, b) => Color.rgb(r, g, b))
```

いくつかランダムな値を生成することで、コードが正しく動作することを確認しておこう。

```scala mdoc
randomRGB.replicateA(2).run
```

うまく動いているようである。

ランダムデータの生成源が用意できたら、それを使ってテストを書く。プログラマが手作業で用意するには非現実的な量のデータを簡単に生成できるため、コードの正しさについてより高い確信を得ることができる。テストの具体的な書き方はここでは重要ではない。次へ進むことにしよう。

ここで見たのは確率モナドを使ってランダムデータを生成する例である。確率モナドは他のあらゆる代数と同じ仕組みで動作する。すなわち、コンストラクタ（`Random.int`）、コンビネータ（`map` および `mapN`）、インタープリタ（`run`）によって構成される。モナドであるということは、この代数が特定の構造をもっていることを意味する。たとえば、`pure` や `flatMap` が定義されており、それらから `mapN` を導出できる。

以下に、確率モナドのインターフェースをそれっぽくスケッチしてみよう。

```scala mdoc:reset:silent
trait Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B]
}
object Random {
  def pure[A](value: A): Random[A] = ???
  
  // 0以上1未満の、一様分布にしたがうランダムな Double を生成する
  val double: Random[Double] = ???
  
  // 一様分布にしたがうランダムな Int を生成する
  val int: Random[Int] = ???
}
```

このインターフェースは、モナドであるための最小限の要件と、いくつかのコンストラクタを備えている。実装を進めるには、[@sec:interpreters:reification]節で紹介したレイフィケーション戦略を適用すればよい。

```scala mdoc:reset:silent
enum Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)

  case RFlatMap[A, B](source: Random[A], f: A => Random[B])
      extends Random[B]
  case RPure(value: A)
  case RDouble extends Random[Double]
  case RInt extends Random[Int]
}
object Random {
  import Random.{RPure, RDouble, RInt}

  def pure[A](value: A): Random[A] = RPure(value)

  // 0以上1未満の、一様分布にしたがうランダムな Double を生成する
  val double: Random[Double] = RDouble

  // 一様分布にしたがうランダムな Int を生成する
  val int: Random[Int] = RInt
}
```

次のステップでは標準的な構造的再帰を用いてインタープリタを実装する。このインタープリタにはパラメータがひとつあり、ソースとなる乱数生成器を受け取る。

```scala
def run(rng: scala.util.Random = scala.util.Random): A =
  this match {
    case RFlatMap(source, f) => f(source.run(rng)).run(rng)
    case RPure(value)        => value
    case RDouble             => rng.nextDouble()
    case RInt                => rng.nextInt()
  }
```

`RDouble` や `RInt` のケースが型パラメータ `A` に対して具体的な型を提供していることから、このコードはインデックス付きデータの一例である。したがって、これらのケースではインタープリタがその具体的な型の値を生成できる。インデックス付きデータを使わなければ、生成できるのは常に型 `A` の値に限られる。しかも、その値はプログラマが事前に用意しなければならない。たとえば `RPure` のケースがこれにあたる。

この実装を仕上げるには `Monad` 型クラスを実装するとよい。そうすれば `mapN` やその他のメソッドを自動的に得ることができる。しかし、本節はインデックス付きデータにフォーカスしており、`Monad` の実装はこのケーススタディの範囲外である。練習が必要だと感じるなら自分で `Monad` を実装してみるとよいだろう。

インデックス付きデータでは、具体的な型とジェネリックな型を混在させることもできる。たとえば `Random` に `product` メソッドを追加することを考えてみよう。

```scala mdoc:reset:silent
enum Random[A] {
  // ...

  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)

  case RProduct[A, B](left: Random[A], right: Random[B]) extends Random[(A, B)]
  // .. other cases here
}
```

`RProduct` ケースでは、型パラメータが `(A, B)` に固定されており、具体的なタプル型とジェネリックな型 `A` および `B` とが混在している。

Scala2 ではインデックス付きデータを扱うためにいくつかの工夫が必要であり、Scala3 においても同様の工夫が役立つことがある。次に示すのは、確率モナドを Scala2 に移植した例である。このコードには `using` ディレクティブを含めてあるので、ファイルに貼り付けて Scala CLI で実行すれば Scala 2.13 の最新版が用いられる。

```scala mdoc:reset:silent
//> using scala 2.13

sealed trait Random[A] {
  import Random._

  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)

  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)

  def run(rng: scala.util.Random = scala.util.Random): A =
    this match {
      case RFlatMap(source, f) => f(source.run(rng)).run(rng)
      case RProduct(l, r)      => (l.run(rng), r.run(rng))
      case RPure(value)        => value
      case RDouble             => rng.nextDouble()
      case RInt                => rng.nextInt()
    }

}
object Random {
  final case class RFlatMap[A, B](source: Random[A], f: A => Random[B])
      extends Random[B]
  final case class RProduct[A, B](left: Random[A], right: Random[B])
      extends Random[(A, B)]
  final case class RPure[A](value: A) extends Random[A]
  case object RDouble extends Random[Double]
  case object RInt extends Random[Int]

  def pure[A](value: A): Random[A] = RPure(value)

  // 0以上1未満の、一様分布にしたがうランダムな Double を生成する
  val double: Random[Double] = RDouble

  // 一様分布にしたがうランダムな Int を生成する
  val int: Random[Int] = RInt
}
```

Scala2 では以下のようなエラーが多数発生する。

```
[error] constructor cannot be instantiated to expected type;
[error]  found   : Random.RProduct[A(in class RProduct),B]
[error]  required: Random[A(in trait Random)]
[error]       case RProduct(l, r)      => (l.run(rng), r.run(rng))
[error]            ^^^^^^^^
```

この問題を解決するには、インタープリタの中で新たな型パラメータをもつネストされたメソッドを定義する必要がある。これを以下に示す。この変更を加えることで Scala2 の型推論が機能し、コードを正しくコンパイルできるようになる。

```scala
def run(rng: scala.util.Random = scala.util.Random): A = {
  def loop[A](random: Random[A]): A =
    random match {
      case RFlatMap(source, f)   => loop(f(loop(source)))
      case RProduct(left, right) => (loop(left), loop(right))
      case RPure(value)          => value
      case RDouble               => rng.nextDouble()
      case RInt                  => rng.nextInt()
    }

  loop(this)
}
```

もうひとつのテクニックは、型タグにマッチさせるパターンマッチを使いたい場合に必要となるものである。これは次のようなマッチング形式を使うケースを指している。

```scala
case r: RPure[A] => ???
```

以下のようなケースの話ではない。

```scala
case RPure(value) => ???
```

For cases like `RProduct` it is not clear how to write these pattern matches, as the type parameters `A` and `B` for `RProduct` don't correspond to the type parameter `A` on `Random`. The solution is use lower case names from the type parameters. Concretely, this means we can write

`RProduct` のようなケースでは、こういったパターンマッチをどのように書けばよいのかが明確でない。`RProduct` に現れる型パラメータ `A` および `B` は、`Random` に現れる型パラメータ `A` には対応していないからである。解決策は、型パラメータとして小文字の名前を使うことである。具体的には、次のように書くことができる。

```scala
case r: RProduct[a, b] => ???
```

ここで使われている型パラメータ `a` と `b` は存在型（existential type）である。何らかの型が存在することは分かっているが、どういう具体型なのかはわからない。このテクニックは Scala 2 ではときどき必要になるが、Scala 3 ではその必要はきわめてまれである。


[^numerics]: 数値的な問題により、色にわずかな差異が生じることがあるが、それは無視すべきである。

[doodle]: https://www.creativescala.org/doodle/
