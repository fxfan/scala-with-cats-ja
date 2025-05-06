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

Let's see an example. In implementing a programming language we need some representation of values within the language.
Suppose our language supports strings, integers, and doubles, which we will represent with the corresponding Scala types.
The code below shows how we can implement this.

```scala mdoc:silent
enum Value[A] {
  case VString(value: String) extends Value[String]
  case VInt(value: Int) extends Value[Int]
  case VDouble(value: Double) extends Value[Double]
}
```

This is indexed data, as it meets the criteria above: we have a type parameter `A` that is instantiated with a concrete type in the cases `VString`, `VInt`, and `VDouble`.
The natural next question is why is this useful?
It will take a more involved example to show why, so let us now dive into one that makes good use of indexed data.


### The Probability Monad 

Our case study will be creating a probability monad. This is a composable abstraction for defining probability distributions. The probability monad has a lot of uses. The most relevant to most developers is generating data for property-based tests, so we'll focus on this use case. However, it can also be used, for example, for statistical inference or for creating generative art. See the conclusions (Section [@sec:indexed-types:conclusions]) for some pointers to these uses.

Let's start with an example of generating random data. [Doodle][doodle] is a Scala library for graphics and visualization. A core part of the library is representing colors. Doodle has two different representations, RGB and OkLCH, with conversions between the two. Testing these conversions is an excellent use of property-based testing. If we can generate many, say, random RGB colors, we can test the conversion by checking the roundrip from RGB to OkLCH and back results in the original color[^numerics]. 

To create an RGB color we need three unsigned bytes, so our first task is to define how we generate a random byte. Doodle happens to have an implementation of the probability monad that we will use. Here is how we can do it.

```scala mdoc:silent
import cats.syntax.all.*
import doodle.core.Color
import doodle.core.UnsignedByte
import doodle.random.{*, given}

val randomByte: Random[UnsignedByte] = 
  Random.int(0, 255).map(UnsignedByte.clip)
```

Note that once again we see the interpreter strategy. A `Random[UnsignedByte]` is a value representing a program that will generate a random `UnsignedByte` when it runs.

With three random unsigned bytes we can create a random RGB color.

```scala mdoc:silent
val randomRGB: Random[Color] =
  (randomByte, randomByte, randomByte).mapN((r, g, b) => Color.rgb(r, g, b))
```

We might want to check our code by generating a few random values.

```scala mdoc
randomRGB.replicateA(3).run
```

It seems to be working.

What we have seen is an illustration of using the probability monad to generate random data. The probability monad works the same way as every other algebra: we have constructors (`Random.int`), combinators (`map`, and `mapN`), and interpreters (`run`). Being a monad means the algebra has some specific structure. For example, it tells us that we have `pure` and `flatMap` available, from which we can derive `mapN`.

Let's sketch an plausible interface for our probability monad.

```scala
trait Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B]
  
  def map[B](f: A => B): Random[B]
  
  def product[B](that: Random[B]): Random[(A, B)]
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

The interface has the minimum requirements to be a monad, and a few other combinators and constructors. We can make progress on the implementation by applying the reification strategy, introduced in Section [@sec:interpreters:reification].

```scala
enum Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)
  
  def map[B](f: A => B): Random[B] =
    RMap(this, f)
  
  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)
  
  case RFlatMap[A, B](source: Random[A], f: A => Random[B]) 
    extends Random[B]
  case RMap[A, B](source: Random[A], f: A => B) 
    extends Random[B]
  case RProduct[A, B](source: Random[A], that: Random[B])
    extends Random[(A, B)]
  case RPure[A](value: A)
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

[^numerics]: Due to numeric issues there may be small differences between the colors that we should ignore.

[doodle]: https://www.creativescala.org/doodle/


```scala mdoc:reset:silent
```
--->

## インデックス付きデータ

インデックス付きデータの中心となるアイデアは、データ内に型の等式をエンコードすることである。それらの等式はデータを検査する際（通常は構造的再帰を通じて）に発見され、それによって生成できる値が制限される。あらためて、余データとの双対性に注目しよう。インデックス付き余データは呼び出せるメソッドを制限し、インデックス付きデータは生成できる値を制限する。インデックス付きデータは一般化代数的データ型と呼ばれることが多いが、ここでは、インデックス付き余データとの関係を強調するために、より簡潔な「インデックス付きデータ」という用語を用いる。ついでに言えば、タイピングが簡単であることもこの用語を用いる理由である[^tn-indexed-types-data-01]。

[^tn-indexed-types-data-01] 【訳注】 英語で indexed data のほうが generalized algebraic data types よりも短くてタイピングが楽だと言っている。

具体的には、Scala におけるインデックス付きデータは、次のような場合に現れる。

1. 少なくともひとつの型パラメータをもつ直和型を定義し
2. その型のバリアントが型パラメータを具体的な型に固定して定義されている場合

例をひとつ見てみよう。プログラミング言語を実装する場合を考えたとき、その言語における値の表現がいくつか必要となる。たとえば、対象の言語が文字列、整数、浮動小数点数をサポートするものとしよう。これらは Scala の対応する型を用いて表現することができる。実装例を以下に示す。」

```scala mdoc:silent
enum Value[A] {
  case VString(value: String) extends Value[String]
  case VInt(value: Int) extends Value[Int]
  case VDouble(value: Double) extends Value[Double]
}
```

これは上述の条件を満たしているので、インデックス付きデータである。型パラメータ `A` をもち、それがバリアント `VString`、`VInt`、`VDouble` において具体的な型で固定されている。続いて生じる疑問は、これがなぜ有用なのかだが、それに答えるにはもうすこし手の込んだ例が必要になる。次に、インデックス付きデータの利点をうまく活かしている例を詳しく見ていこう。

### 確率モナド

このケーススタディでは確率モナドを構築する。これは確率分布を定義するための合成可能な抽象である。確率モナドには多くの用途があるが、多くの開発者にとってもっとも身近なのはプロパティベースのテストにおけるデータ生成である。ここではその用途に焦点をあてていく。確率モナドは統計的推論やジェネラティブアートの作成などにも利用できる。それらの使い方については本章のまとめである[@sec:indexed-types:conclusions]節を参照してほしい。

まずはランダムデータを生成する例から始めよう。[Doodle][doodle] は Scala のグラフィックおよび可視化ライブラリである。このライブラリの中核のひとつが色の表現である。Doodle では RGB と OkLCH のふたつの表現形式があり、両者間の変換が定義されている。この変換ロジックはプロパティベースのテストの題材として最適である。たくさんのランダムな RGB カラーを生成し、それを OkLCH に変換して再び RGB に戻した結果が元の色になるかどうかを確認することで、変換の正しさを検証することができる[^numerics]。

RGB カラーを生成するには三つの符号なしバイトが必要である。そこで、最初の課題としてランダムなバイトの精製方法を定義したい。Doodle には確率モナドの実装が用意されているので、それを利用していこう。

```scala mdoc:silent
import cats.syntax.all.*
import doodle.core.Color
import doodle.core.UnsignedByte
import doodle.random.{*, given}

val randomByte: Random[UnsignedByte] = 
  Random.int(0, 255).map(UnsignedByte.clip)
```

ここにもインタープリタ戦略が登場している点に注目してほしい。`Random[UnsignedByte]` は、実行するとランダムな `UnsignedByte` を生成するプログラムを表す値である。

ランダムな符号なしバイトが三つあれば、ランダムな RGB カラーを生成できる。

```scala mdoc:silent
val randomRGB: Random[Color] =
  (randomByte, randomByte, randomByte).mapN((r, g, b) => Color.rgb(r, g, b))
```

いくつかランダムな値を生成することで、コードが正しく動作することを確認しておこう。

```scala mdoc
randomRGB.replicateA(3).run
```

うまく動いているようである。

ここで見たのは確率モナドを使ってランダムデータを生成する例である。確率モナドは他のあらゆる代数と同じ仕組みで動作する。すなわち、コンストラクタ（`Random.int`）、コンビネータ（`map` および `mapN`）、インタープリタ（`run`）によって構成される。モナドであるということは、この代数が特定の構造をもっていることを意味する。たとえば、`pure` や `flatMap` が定義されており、それらから `mapN` を導出できる。

以下に、確率モナドのインターフェースをそれっぽくスケッチしてみよう。

```scala
trait Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B]
  
  def map[B](f: A => B): Random[B]
  
  def product[B](that: Random[B]): Random[(A, B)]
}
object Random {
  def pure[A](value: A): Random[A] = ???
  
  // 0以上1未満の、一様分布にしたがうランダムな Double を生成する
  val double: Random[Double] = ???
  
  // 一様分布にしたがうランダムな Int を生成する
  val int: Random[Int] = ???
}
```

このインターフェースは、モナドであるための最小限の要件と、いくつかのコンビネータやコンストラクタを備えている。実装を進めるには、[@sec:interpreters:reification]節で紹介したレイフィケーション戦略を適用すればよい。

```scala
enum Random[A] {
  def flatMap[B](f: A => Random[B]): Random[B] =
    RFlatMap(this, f)
  
  def map[B](f: A => B): Random[B] =
    RMap(this, f)
  
  def product[B](that: Random[B]): Random[(A, B)] =
    RProduct(this, that)
  
  case RFlatMap[A, B](source: Random[A], f: A => Random[B]) 
    extends Random[B]
  case RMap[A, B](source: Random[A], f: A => B) 
    extends Random[B]
  case RProduct[A, B](source: Random[A], that: Random[B])
    extends Random[(A, B)]
  case RPure[A](value: A)
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

[^numerics]: 数値的な問題により、色にわずかな差異が生じることがあるが、それは無視すべきである。

[doodle]: https://www.creativescala.org/doodle/
