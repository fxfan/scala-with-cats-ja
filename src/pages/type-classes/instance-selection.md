## 型クラスと変位 Type Classes and Variance

In this section we'll discuss how variance interacts
with type class instance selection.
Variance is one of the darker corners of Scala's type system,
so we start by reviewing it
before moving on to its interaction with type classes.

この節では、変位（variance）が型クラスのインスタンス選択にどのように影響するかを説明する。変位は、Scalaの型システムの中でも理解が難しい部分のひとつなので、まずは変位についておさらいし、それから型クラスとの相互作用について見ていく。

### 変位 Variance {#sec:variance}

Variance concerns the relationship between 
an instance defined on a type and its subtypes.
For example, if we define a `JsonWriter[Option[Int]]`,
will the expression `Json.toJson(Some(1))` select this instance?
(Remember that `Some` is a subtype of `Option`).

変位は、ある型およびその部分型（subtype）に対して定義された型クラスインスタンス同士の関係性に関わる概念である。たとえば、`Some` は `Option` の部分型だが、`JsonWriter[Option[Int]]` 型のインスタンスが定義されているときに、`Json.toJson(Some(1))` という式のコンテキストパラメータとしてそのインスタンスが選択されるかどうか、が変位によって決まる。

<!--
TODO: Variance concerns the relationship between an instance defined on a type and its subtypes が何を言っているのかわからない
-->

We need two concepts to explain variance: 
type constructors, and subtyping.

変位を説明するにあたっては、型コンストラクタとサブタイピングという二つの概念が必要となる。

Variance applies to any **type constructor**,
which is the `F` in a type `F[A]`.
So, for example, `List`, `Option`, and `JsonWriter` are all type constructors.
A type constructor must have at least one type parameter,
and may have more.
So `Either`, with two type parameters, is also a type constructor.

変位はすべての**型コンストラクタ**に適用される。型コンストラクタとは `F[A]` という型における `F` の部分を指す。たとえば、`List` や `Option` や `JsonWriter` はすべて型コンストラクタである。型コンストラクタは最低でもひとつの型パラメータをもっていなければならない。つまり、型パラメータを二つもっている `Either` も型コンストラクタである。

Subtyping is a relationship between types.
We say that `B` is a subtype of `A`
if we can use a value of type `B`
anywhere we expect a value of type `A`.
We may sometimes use the shorthand `B <: A`
to indicate that `B` is a subtype of `A`.

サブタイピングとは型同士の関係である。`A` 型の値を必要としている場所に `B` 型の値を使うことができるのであれば、`B` は `A` の部分型であると言える。この関係を表すのに `B <: A` という省略記法を用いることがある。

Variance concerns the subtyping relationship between types `F[A]` and `F[B]`,
given a subtyping relationship between `A` and `B`.
If `B` is a subtype of `A` then

1. if `F[B] <: F[A]` we say `F` is **covariant** in `A`; else
2. if `F[B] >: F[A]` we say `F` is **contravariant** in `A`; else
3. if there is no subtyping relationship between `F[B]` and `F[A]` we say `F` is **invariant** in `A`.

変位は、`A` と `B` の間に部分型関係がある場合の、`F[A]` と `F[B]` との間の部分型関係がどのようなものであるかを表す。`B` が `A` の部分型である場合、各変位は以下のように説明できる。

1. `F[B] <: F[A]` であれば、`F` は `A` に対して **共変（covariant）** であるという
2. `F[B] >: F[A]` であれば、`F` は `A` に対して **反変（contravariant）** であるという
3. `F[B]` と `F[A]` の間に部分型関係がなければ、`F` は `A` に対して **非変（invariant）** であるという

When we define a type constructor
we can also add variance annotations to its type parameters.
For example, we denote covariance with a `+` symbol:

型コンストラクタを定義する際には、その型パラメータに変位アノテーションをつけることもできる。たとえば、共変性は `+` 記号で表す。

```scala
trait F[+A] // "+" は共変を表す
```

If we don't add a variance annotation, the type parameter is invariant.
Let's now look at covariance, contravariance, and invariance in detail.

変位アノテーションをつけなければ、その型パラメータは非変となる。次に、共変・反変・非変について詳しく見ていこう。

### 共変 Covariance

Covariance means that the type `F[B]`
is a subtype of the type `F[A]` if `B` is a subtype of `A`.
This is useful for modelling many types,
including collections like `List` and `Option`:

共変とは、`B` が `A` の部分型であるときに、`F[B]` が `F[A]` の部分型となる関係をいう。共変は、`List` や `Option` などのコレクションをはじめとする多くの型をモデリングするのに使える。

```scala
trait List[+A]
trait Option[+A]
```

The covariance of Scala collections allows
us to substitute collections of one type with a collection of a subtype in our code.
For example, we can use a `List[Circle]`
anywhere we expect a `List[Shape]` because
`Circle` is a subtype of `Shape`:

Scalaのコレクションが共変であることにより、ある型のコレクションをその部分型のコレクションで置き換えることが可能である。たとえば、`Circle` が `Shape` の部分型であるならば `List[Shape]` が期待されるすべての場所で `List[Circle]` を使うことができる。

```scala mdoc:silent
sealed trait Shape
final case class Circle(radius: Double) extends Shape
```

```scala
val circles: List[Circle] = ???
val shapes: List[Shape] = circles
```

```scala mdoc:invisible
val circles: List[Circle] = null
val shapes: List[Shape] = circles
```

Generally speaking, covariance is used for outputs:
data that we can later get out of a container type such as `List`,
or otherwise returned by some method.

一般的に言えば、共変は、`List` のようなコンテナ型から取り出すことのできるデータや、メソッドの戻り値など出力となるデータの型に対して用いられる。

### 反変 Contravariance

What about contravariance?
We write contravariant type constructors
with a `-` symbol like this:

反変は次のとおり `-` 記号で表記される。

```scala
trait F[-A]
```

Perhaps confusingly, contravariance means that the type `F[B]`
is a subtype of `F[A]` if `A` is a subtype of `B`.
This is useful for modelling types that represent inputs,
like our `JsonWriter` type class above:

ややこしいかもしれないが、反変とは、`A` が `B` の部分型であるときに、`F[B]` が `F[A]` の部分型となる関係をいう。既出の `JsonWriter` 型クラスのように、入力される型をモデリングするときに用いられる。

```scala mdoc:invisible
trait Json
```

```scala mdoc
trait JsonWriter[-A] {
  def write(value: A): Json
}
```

Let's unpack this a bit further.
Remember that variance is all about
the ability to substitute one value for another.
Consider a scenario where we have two values,
one of type `Shape` and one of type `Circle`,
and two `JsonWriters`, one for `Shape` and one for `Circle`:

もう少し掘り下げてみたい。変位とは、ある型の値を別の型の値に置き換えられるかどうか、を扱う概念である。`Shape` 型の値と `Circle` 型の値、そして `Shape` と `Circle` それぞれの `JsonWriter` がある状況を考えてみよう。

```scala
val shape: Shape = ???
val circle: Circle = ???

val shapeWriter: JsonWriter[Shape] = ???
val circleWriter: JsonWriter[Circle] = ???
```

```scala mdoc:invisible
val shape: Shape = null
val circle: Circle = null

val shapeWriter: JsonWriter[Shape] = null
val circleWriter: JsonWriter[Circle] = null
```

```scala mdoc:silent
def format[A](value: A, writer: JsonWriter[A]): Json =
  writer.write(value)
```

Now ask yourself the question:
"Which combinations of value and writer can I pass to `format`?"
We can `write` a `Circle` with either writer
because all `Circles` are `Shapes`.
Conversely, we can't write a `Shape` with `circleWriter`
because not all `Shapes` are `Circles`.

`format` に渡せる値とライターの組み合わせはどれか、考えてみてほしい。`Circle` はすべて `Shape` でもあるため、どちらのライターを使っても `Circle` を `write` できる。逆に、`Shape` はすべてが `Circle` というわけではないので、`circleWriter` で `Shape` を書き出すことはできない。

This relationship is what we formally model using contravariance.
`JsonWriter[Shape]` is a subtype of `JsonWriter[Circle]`
because `Circle` is a subtype of `Shape`.
This means we can use `shapeWriter`
anywhere we expect to see a `JsonWriter[Circle]`.

定義に基づくと、この関係は反変性を使ってモデル化される。`Circle` が `Shape` の部分型であるため、`JsonWriter[Shape]` は `JsonWriter[Circle]` の部分型となる。つまり、`JsonWriter[Circle]` が期待される場所であればどこでも、`shapeWriter` を使用することができる。

### 非変 Invariance

Invariance is the easiest situation to describe.
It's what we get when we don't write a `+` or `-`
in a type constructor:

非変性はもっともシンプルである。`+` と `-` いずれも指定しなかった型パラメータは非変となる。

```scala
trait F[A]
```

This means the types `F[A]` and `F[B]`
are never subtypes of one another,
no matter what the relationship between `A` and `B`.
This is the default semantics for Scala type constructors.

これは、 `A` と `B` との関係がいかなるものであっても、`F[A]` と `F[B]` とが互いに部分型関係をもたないことを意味する。これがScalaの型コンストラクタにおけるデフォルトの変位である。

### 変位とインスタンス選択 Variance and Instance Selection

When the compiler searches for a given instnace
it looks for one matching the type *or subtype*.
Thus we can use variance annotations
to control type class instance selection to some extent.

コンパイラは、求めている型か*もしくはその部分型*にマッチするgivenインスタンスを探す。したがって、変位指定を行うことで、ある程度は型クラスのインスタンス選択を制御することができる。

There are two issues that tend to arise.
Let's imagine we have an algebraic data type like:

よく発生する問題が二つある。以下のような代数的データ型があると想像してほしい。

```scala mdoc:silent
enum A {
  case B
  case C
}
```

The issues are:

 1. Will an instance defined on a supertype be selected
    if one is available?
    For example, can we define an instance for `A`
    and have it work for values of type `B` and `C`?

 2. Will an instance for a subtype be selected
    in preference to that of a supertype.
    For instance, if we define an instance for `A` and `B`,
    and we have a value of type `B`,
    will the instance for `B` be selected in preference to `A`?

検討すべき点は次のとおりである。

 1. スーパータイプに対して定義されたインスタンスがあれば、それが選択されるか。たとえば、`A` 用に定義されたインスタンスは `B` 型や `C` 型の値に対しても機能するのか。

 2. 部分型に対して定義されたインスタンスはスーパータイプに対するものよりも優先的に選択されるか。たとえば、`A` と `B` それぞれ用のインスタンスが定義されている状態で、`B` 型の値に対するインスタンスを必要とした場合、`B` 用のインスタンスが `A` 用のものよりも優先的に選択されるのか。

It turns out we can't have both at once.
The three choices give us behaviour as follows:

両方を同時に実現することはできない。選択肢によって、次のような動作になります：

---------------------------------------------------------------------------
型クラスの変位指定                     非変         共変        反変
----------------------------------- ----------- ----------- ---------------
スーパータイプ用のインスタンスが使われるか  いいえ       いいえ       はい
より特化した型のインスタンスが使われるか   いいえ       はい         いいえ
---------------------------------------------------------------------------

Let's see some examples, using the following types
to show the subtyping relationship.

以下の型を用いて、部分型関係を示す例を見てみよう。

```scala mdoc:reset:silent
trait Animal
trait Cat extends Animal
trait DomesticShorthair extends Cat
```

Now we'll define three different type classes for the three types of variance, 
and define an instance of each for the `Cat` type.

今、三種類の変位に対応する三つの異なる型クラスを定義し、それぞれに対して `Cat` 型のインスタンスを定義する。

```scala mdoc:silent
trait Inv[A] {
  def result: String
}
object Inv {
  given Inv[Cat] with
    def result = "Invariant"
    
  def apply[A](using instance: Inv[A]): String =
    instance.result
}

trait Co[+A] {
  def result: String
}
object Co {
  given Co[Cat] with
    def result = "Covariant"

  def apply[A](using instance: Co[A]): String =
    instance.result
}

trait Contra[-A] {
  def result: String
}
object Contra {
  given Contra[Cat] with
    def result = "Contravariant"

  def apply[A](using instance: Contra[A]): String =
    instance.result
}
```

Now the cases that work, all of which select the `Cat` instance.
For the invariant case we must ask for exactly the `Cat` type.
For the covariant case we can ask for a supertype of `Cat`.
For contravariance we can ask for a subtype of `Cat`.

まず正常に動作するケースを考えよう。選択されるのは常に `Cat` 用のインスタンスである。非変の場合、厳密に `Cat` 型を指定する必要がある。共変の場合は、`Cat` のスーパータイプ、反変の場合は、`Cat` の部分型を指定することができる。

```scala mdoc
Inv[Cat]
Co[Animal]
Co[Cat]
Contra[DomesticShorthair]
Contra[Cat]
```

Now cases that fail.
With invariance any type that is not `Cat` will fail.
So the supertype fails

次は動作しないケースである。非変の場合、`Cat` 以外の型に対しては該当するインスタンスは見つからない。以下のようなスーパータイプ用の型インスタンス検索は失敗するし、

```scala mdoc:fail
Inv[Animal]
```

部分型用も同様である。

```scala mdoc:fail
Inv[DomesticShorthair]
```

Covariance fails for any subtype of the type for which the instance is declared.

共変の場合、インスタンスが定義されている型の部分型に対するインスタンス検索はすべて失敗する。
```scala mdoc:fail
Co[DomesticShorthair]
```

Contravariance fails for any supertype of the type for which the instance is declared.

反変の場合、インスタンスが定義されている型のスーパータイプに対するインスタンス検索はすべて失敗する。

```scala mdoc:fail
Contra[Animal]
```

It's clear there is no perfect system.
The most choice is to use invariant type classes.
This allows us to specify
more specific instances for subtypes if we want.
It does mean that if we have, for example,
a value of type `Some[Int]`,
our type class instance for `Option` will not be used.
We can solve this problem with
a type annotation like `Some(1) : Option[Int]`
or by using "smart constructors"
like the `Option.apply`, `Option.empty`, `some`, and `none` methods
we saw in Section [@sec:type-classes:comparing-options].

言うまでもなく、完全なシステムは存在しない。もっとも用いられるのは非変の型クラスである。これにより、必要に応じてより具体的なインスタンスを部分型に対して指定することが可能となる。ただし、これはたとえば、`Some[Int]` 型の値に対して `Option` 用の型クラスインスタンスが使用されないことを意味する。
この問題は、`Some(1): Option[Int]` のような型注釈を使うか、[@sec:type-classes
]の節で見た `Option.apply`、`Option.empty`、`some`、`none` メソッドといった「スマートコンストラクタ」を使うことで解決できる。
