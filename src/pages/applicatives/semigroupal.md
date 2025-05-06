<!--

## Semigroupal {#sec:semigroupal}

[`cats.Semigroupal`][cats.semigroupal] is a type class that
allows us to combine contexts[^semigroupal-name].
If we have two objects of type `F[A]` and `F[B]`,
a `Semigroupal[F]` allows us to combine them to form an `F[(A, B)]`.
Its definition in Cats is:

```scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

As we discussed at the beginning of this chapter,
the parameters `fa` and `fb` are independent of one another:
we can compute them in either order before passing them to `product`.
This is in contrast to `flatMap`,
which imposes a strict order on its parameters.
This gives us more freedom when defining
instances of `Semigroupal` than we get when defining `Monads`.

[^semigroupal-name]: It
is also the winner of Underscore's 2017 award for
the most difficult functional programming term
to work into a coherent English sentence.

### Joining Two Contexts

While `Semigroup` allows us to join values,
`Semigroupal` allows us to join contexts.
Let's join some `Options` as an example:

```scala mdoc:silent:reset-object
import cats.Semigroupal
import cats.instances.option._ // for Semigroupal
```

```scala mdoc
Semigroupal[Option].product(Some(123), Some("abc"))
```

If both parameters are instances of `Some`,
we end up with a tuple of the values within.
If either parameter evaluates to `None`,
the entire result is `None`:

```scala mdoc
Semigroupal[Option].product(None, Some("abc"))
Semigroupal[Option].product(Some(123), None)
```

### Joining Three or More Contexts

The companion object for `Semigroupal` defines
a set of methods on top of `product`.
For example, the methods `tuple2` through `tuple22`
generalise `product` to different arities:

```scala mdoc:silent
import cats.instances.option._ // for Semigroupal
```

```scala mdoc
Semigroupal.tuple3(Option(1), Option(2), Option(3))
Semigroupal.tuple3(Option(1), Option(2), Option.empty[Int])
```

The methods `map2` through `map22`
apply a user-specified function
to the values inside 2 to 22 contexts:

```scala mdoc
Semigroupal.map3(Option(1), Option(2), Option(3))(_ + _ + _)

Semigroupal.map2(Option(1), Option.empty[Int])(_ + _)
```

There are also methods `contramap2` through `contramap22`
and `imap2` through `imap22`,
that require instances of `Contravariant` and `Invariant` respectively.

### Semigroupal Laws

There is only one law for `Semigroupal`:
the `product` method must be associative.

```scala
product(a, product(b, c)) == product(product(a, b), c)
```

## Apply Syntax

Cats provides a convenient *apply syntax*
that provides a shorthand for the methods described above.
We import the syntax from [`cats.syntax.apply`][cats.syntax.apply].
Here's an example:

```scala mdoc:silent
import cats.instances.option._ // for Semigroupal
import cats.syntax.apply._     // for tupled and mapN
```

The `tupled` method is implicitly added to the tuple of `Options`.
It uses the `Semigroupal` for `Option` to zip the values inside the
`Options`, creating a single `Option` of a tuple:

```scala mdoc
(Option(123), Option("abc")).tupled
```

We can use the same trick on tuples of up to 22 values.
Cats defines a separate `tupled` method for each arity:

```scala mdoc
(Option(123), Option("abc"), Option(true)).tupled
```

In addition to `tupled`, Cats' apply syntax provides
a method called `mapN` that accepts an implicit `Functor`
and a function of the correct arity to combine the values.

```scala mdoc:silent
final case class Cat(name: String, born: Int, color: String)
```

```scala mdoc
(
  Option("Garfield"),
  Option(1978),
  Option("Orange & black")
).mapN(Cat.apply)
```

Of all the methods mentioned here,
it is most common to use `mapN`.

Internally `mapN` uses the `Semigroupal`
to extract the values from the `Option`
and the `Functor` to apply the values to the function.

It's nice to see that this syntax is type checked.
If we supply a function that
accepts the wrong number or types of parameters,
we get a compile error:

```scala mdoc
val add: (Int, Int) => Int = (a, b) => a + b
```

```scala mdoc:fail
(Option(1), Option(2), Option(3)).mapN(add)
```

```scala mdoc:fail
(Option("cats"), Option(true)).mapN(add)
```

### Fancy Functors and Apply Syntax

Apply syntax also has `contramapN` and `imapN` methods
that accept Contravariant and Invariant functors
(Section [@sec:functors:contravariant-invariant]).
For example, we can combine `Monoids` using `Invariant`.
Here's an example:

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.instances.int._        // for Monoid
import cats.instances.invariant._  // for Semigroupal
import cats.instances.list._       // for Monoid
import cats.instances.string._     // for Monoid
import cats.syntax.apply._         // for imapN

final case class Cat(
  name: String,
  yearOfBirth: Int,
  favoriteFoods: List[String]
)

val tupleToCat: (String, Int, List[String]) => Cat =
  Cat.apply _

val catToTuple: Cat => (String, Int, List[String]) =
  cat => (cat.name, cat.yearOfBirth, cat.favoriteFoods)

implicit val catMonoid: Monoid[Cat] = (
  Monoid[String],
  Monoid[Int],
  Monoid[List[String]]
).imapN(tupleToCat)(catToTuple)
```

Our `Monoid` allows us to create "empty" `Cats`,
and add `Cats` together using the syntax from Chapter [@sec:monoids]:

```scala mdoc:silent
import cats.syntax.semigroup._ // for |+|

val garfield   = Cat("Garfield", 1978, List("Lasagne"))
val heathcliff = Cat("Heathcliff", 1988, List("Junk Food"))
```

```scala mdoc
garfield |+| heathcliff
```


```scala mdoc:reset:silent
```
--->

## `Semigroupal` {#sec:semigroupal}

[`cats.Semigroupal`][cats.semigroupal] はコンテキスト同士の結合を可能にする型クラスである[^semigroupal-name]。`F[A]` 型と `F[B]` 型のオブジェクトがひとつずつあるとき、`Semigroupal[F]` はそれらを結合して `F[(A, B)]` を形成することができる。Cats においては以下のように定義されている。

```scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

この章の冒頭で述べたように、パラメータ `fa` と `fb` は互いに独立している。これらは、`product` に渡す前にどちらを先に計算してもかまわない。このことは、一連の `flatMap` 呼び出しにおいて計算の実行に厳密な順序が課されることとは対照的である。そのため、`Monad` よりも `Semigroupal` のインスタンスを定義する時のほうが自由度が高い。

[^semigroupal-name]: この用語はUnderscoreの2017アワードにおいて、もっとも意味のわからない関数型プログラミング用語にも選ばれた。

### ふたつのコンテキストの結合

`Semigroup` が値同士を結合するのに対して、`Semigroupal` はコンテキスト同士を結合することができる。例として、ふたつの `Option` 値を結合してみよう。

```scala mdoc:silent:reset-object
import cats.Semigroupal
import cats.instances.option._ // Semigroupal
```

```scala mdoc
Semigroupal[Option].product(Some(123), Some("abc"))
```

両方のパラメータが `Some` であれば、内部の値を組み合わせたタプルが得られる。一方、どちらかのパラメータが `None` の場合は、全体の結果も `None` となる。

```scala mdoc
Semigroupal[Option].product(None, Some("abc"))
Semigroupal[Option].product(Some(123), None)
```

### 三つ以上のコンテキストの結合

`Semigroupal` のコンパニオンオブジェクトには `product` を元にした一連のメソッドが定義されている。たとえば、メソッド `tuple2` 〜 `tuple22` は `product` をさまざまな引数の個数に対応させたものである。

```scala mdoc:silent
import cats.instances.option._ // Semigroupal
```

```scala mdoc
Semigroupal.tuple3(Option(1), Option(2), Option(3))
Semigroupal.tuple3(Option(1), Option(2), Option.empty[Int])
```

メソッド `map2` 〜 `map22` は、コンテキストに包まれた値をそれぞれ2〜22個受け取り、それに指定の関数を適用する。

```scala mdoc
Semigroupal.map3(Option(1), Option(2), Option(3))(_ + _ + _)

Semigroupal.map2(Option(1), Option.empty[Int])(_ + _)
```

メソッド `contramap2` 〜 `contramap22` および `imap2` 〜 `imap22` も存在する。これらはそれぞれ `Contravariant` と `Invariant` のインスタンスを必要とする。

### Semigroupal 則

`Semigroupal` が要求する法則はひとつしかない。`product` メソッドが結合律を満たすことである。

```scala
product(a, product(b, c)) == product(product(a, b), c)
```

## apply 構文

Cats は、上述のメソッドを短縮形で呼び出せる便利な *apply 構文*を提供している。構文は [`cats.syntax.apply`][cats.syntax.apply] からインポートする。以下に例を示す。

```scala mdoc:silent
import cats.instances.option._ // Semigroupal
import cats.syntax.apply._     // tupled および mapN
```

`tupled` メソッドは `Option` のタプルに暗黙的に追加される。このメソッドは `Option` 用の `Semigroupal` を使用して `Option` 内部の値を結合し、ひとつのタプルを内部に保持する単一の `Option` を生成する。

```scala mdoc
(Option(123), Option("abc")).tupled
```

最大で22個の値をもつタプルに対して同じ方法を用いることができる。Cats は引数の個数それぞれに対応する `tupled` メソッドを定義している。

```scala mdoc
(Option(123), Option("abc"), Option(true)).tupled
```

Cats の apply 構文は `tupled` の他に `mapN` というメソッドも提供している。このメソッドは、暗黙の `Functor` と、レシーバとなったタプルの値と同数の引数をもつ関数を受け取り、値を結合する。

```scala mdoc:silent
final case class Cat(name: String, born: Int, color: String)
```

```scala mdoc
(
  Option("Garfield"),
  Option(1978),
  Option("Orange & black")
).mapN(Cat.apply)
```

これらのメソッドの中では `mapN` がもっともよく用いられる。

`mapN` は内部的に `Semigroupal` を使用して `Option` から値を抽出し、`Functor` を使用して値を関数に適用する。

この構文が型安全である点も魅力的である。指定した関数の引数の個数や型が誤っている場合、コンパイルエラーが発生する。

```scala mdoc
val add: (Int, Int) => Int = (a, b) => a + b
```

```scala mdoc:fail
(Option(1), Option(2), Option(3)).mapN(add)
```

```scala mdoc:fail
(Option("cats"), Option(true)).mapN(add)
```

### ファンシーなファンクターと apply 構文

apply 構文は、反変ファンクターを受け取る `contramapN` と、非変ファンクターを受け取る `imapN` メソッドも提供している（これらのファンクターについては[@sec:functors:contravariant-invariant]節を参照）。たとえば `Invariant` を使って `Monoid` を結合することができる。以下に例を示す。

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.instances.int._        // Monoid
import cats.instances.invariant._  // Semigroupal
import cats.instances.list._       // Monoid
import cats.instances.string._     // Monoid
import cats.syntax.apply._         // imapN

final case class Cat(
  name: String,
  yearOfBirth: Int,
  favoriteFoods: List[String]
)

val tupleToCat: (String, Int, List[String]) => Cat =
  Cat.apply _

val catToTuple: Cat => (String, Int, List[String]) =
  cat => (cat.name, cat.yearOfBirth, cat.favoriteFoods)

implicit val catMonoid: Monoid[Cat] = (
  Monoid[String],
  Monoid[Int],
  Monoid[List[String]]
).imapN(tupleToCat)(catToTuple)
```

この `Monoid` は空の `Cat` の作成と、[@sec:monoids]章で紹介した構文を用いた `Cat` 同士の結合を行うことができる。

```scala mdoc:silent
import cats.syntax.semigroup._ // |+|

val garfield   = Cat("Garfield", 1978, List("Lasagne"))
val heathcliff = Cat("Heathcliff", 1988, List("Junk Food"))
```

```scala mdoc
garfield |+| heathcliff
```
