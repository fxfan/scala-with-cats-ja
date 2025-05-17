<!--

## Aside: Partial Unification {#sec:functors:partial-unification}

In Section [@sec:functors:more-examples]
we saw a functor instance for `Function1`.

```scala mdoc:silent
import cats.*
import cats.syntax.functor.*     // for map

val func1 = (x: Int)    => x.toDouble
val func2 = (y: Double) => y * 2
```
```scala mdoc
val func3 = func1.map(func2)
```

`Function1` has two type parameters
(the function argument and the result type):

```scala
trait Function1[-A, +B] {
  def apply(arg: A): B
}
```

However, `Functor` accepts a type constructor with one parameter:

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(func: A => B): F[B]
}
```

The compiler has to fix one of the two parameters
of `Function1` to create a type constructor
of the correct kind to pass to `Functor`.
It has two options to choose from:

```scala
type F[A] = Int => A
type F[A] = A => Double
```

*We* know that the former of these is the correct choice.
However the compiler doesn't understand what the code means.
Instead it relies on a simple rule, 
implementing what is called "partial unification".

The partial unification in the Scala compiler
works by fixing type parameters from left to right.
In the above example, the compiler fixes
the `Int` in `Int => Double`
and looks for a `Functor` for functions of type `Int => ?`:

```scala mdoc:silent
type F[A] = Int => A

val functor = Functor[F]
```

This left-to-right elimination works for
a wide variety of common scenarios,
including `Functors` for
types such as `Function1` and `Either`:

```scala mdoc
val either: Either[String, Int] = Right(123)

either.map(_ + 1)
```


<div class="callout callout-warning">
Partial unification is the default behaviour in Scala 2.13.
In earlier versions of Scala
we need to add the `-Ypartial-unification` compiler flag.
In sbt we would add the compiler flag in `build.sbt`:

```scala
scalacOptions += "-Ypartial-unification"
```

The rationale behind this change is discussed in [SI-2712][link-si2712].
</div>



### Limitations of Partial Unification

There are situations where
left-to-right elimination is not the correct choice.
One example is the `Or` type in [Scalactic][link-scalactic],
which is a conventionally left-biased equivalent of `Either`:

```scala
type PossibleResult = ActualResult Or Error
```

Another example is the `Contravariant` functor for `Function1`.

While the covariant `Functor` for `Function1` implements
`andThen`-style left-to-right function composition,
the `Contravariant` functor implements `compose`-style
right-to-left composition.
In other words, the following expressions are all equivalent:

```scala mdoc:silent
val func3a: Int => Double =
  a => func2(func1(a))

val func3b: Int => Double =
  func2.compose(func1)
```

```scala mdoc:fail:silent
// Hypothetical example. This won't actually compile:
val func3c: Int => Double =
  func2.contramap(func1)
```

If we try this for real, however,
our code won't compile:

```scala mdoc:silent
import cats.syntax.contravariant.* // for contramap
```

```scala mdoc:fail
val func3c = func2.contramap(func1)
```

The problem here is that the `Contravariant` for `Function1`
fixes the return type and leaves the parameter type varying,
requiring the compiler to eliminate type parameters
from right to left, as shown below and in Figure [@fig:functors:function-contramap-type-chart]:

```scala
type F[A] = A => Double
```

![Type chart: contramapping over a Function1](src/pages/functors/function-contramap.pdf+svg){#fig:functors:function-contramap-type-chart}

The compiler fails simply because of its left-to-right bias.
We can prove this by creating a type alias
that flips the parameters on Function1:

```scala mdoc:silent
type <=[B, A] = A => B
```
``` scala
type F[A] = Double <= A
```

If we re-type `func2` as an instance of `<=`,
we reset the required order of elimination and
we can call `contramap` as desired:

```scala mdoc:silent
val func2b: Double <= Double = func2
```

```scala mdoc
val func3c = func2b.contramap(func1)
```

The difference between `func2` and `func2b` is
purely syntactic---both refer to the same value
and the type aliases are otherwise completely compatible.
Incredibly, however,
this simple rephrasing is enough to
give the compiler the hint it needs
to solve the problem.

It is rare that we have to do
this kind of right-to-left elimination.
Most multi-parameter type constructors
are designed to be right-biased,
requiring the left-to-right elimination
that is supported by the compiler
out of the box.
However, it is useful to know about
this quirk of elimination order
in case you ever come across
an odd scenario like the one above.


```scala mdoc:reset:silent
```
--->

## 補足: 部分的ユニフィケーション {#sec:functors:partial-unification}

[@sec:functors:more-examples]節では `Function1` に対するファンクターインスタンスについて見た。

```scala mdoc:silent
import cats.*
import cats.syntax.functor.*     // map

val func1 = (x: Int)    => x.toDouble
val func2 = (y: Double) => y * 2
```
```scala mdoc
val func3 = func1.map(func2)
```

`Function1` は引数をひとつと戻り値型、合わせてふたつの型パラメータをもっている。

```scala
trait Function1[-A, +B] {
  def apply(arg: A): B
}
```

だが、`Functor` が受け付けるのは単一のパラメータをもつ型コンストラクタである。

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(func: A => B): F[B]
}
```

`Functor` に渡すための適切なカインドをもつ型コンストラクタを作るためには、ふたつの型パラメータのうちのひとつを固定する必要がある。固定の仕方には以下のふたつの選択肢がある。

```scala
type F[A] = Int => A
type F[A] = A => Double
```

プログラマから見れば前者が正しい選択肢だとわかるが、コンパイラはコードの意味を理解していない。その代わり、コンパイラは単純な規則に従って「部分的ユニフィケーション」と呼ばれる処理を実行する。

Scala コンパイラの部分的ユニフィケーションは、型パラメータを左から右へと固定していくことによって行われる。上記の例だと、コンパイラは `Int => Double` という型のうち `Int` の部分を固定し、`Int => ?` 型の関数に対する `Functor` インスタンスを探す。

```scala mdoc:silent
type F[A] = Int => A

val functor = Functor[F]
```

この左から右への型パラメータ除去は、`Function1` や `Either` といった型に対する `Functor` インスタンスを含め、多くの一般的なシナリオに対して正しく動作する。

```scala mdoc
val either: Either[String, Int] = Right(123)

either.map(_ + 1)
```

<div class="callout callout-warning">
Scala 2.13 ではデフォルトで部分的ユニフィケーションが有効だが、それ以前のバージョンでは `-Ypartial-unification` コンパイラオプションを追加する必要がある。SBT では `build.sbt` に次のように追加する。

```scala
scalacOptions += "-Ypartial-unification"
```

この変更の背景は [SI-2712][link-si2712] で議論されている。
</div>

### 部分的ユニフィケーションの限界

左から右へたどる型パラメータ除去が正しい選択とならない場合もある。[Scalactic][link-scalactic] の `Or` 型がその一例である。`Or` は `Either` と同じ構造をもつが、左の値にバイアスがかけられている。

```scala
type PossibleResult = ActualResult Or Error
```

もうひとつの例は `Function1` に対する `Contravariant` ファンクターである。

`Function1` に対する共変ファンクターが `andThen` スタイルの、つまり左から右への関数合成を実装しているのに対して、反変ファンクターは右から左への `compose` スタイルの関数合成を実装している。言い換えると、下記の式はすべて等価である。

```scala mdoc:silent
val func3a: Int => Double =
  a => func2(func1(a))

val func3b: Int => Double =
  func2.compose(func1)
```

```scala mdoc:fail:silent
// 仮の例。実際にはコンパイルできない
val func3c: Int => Double =
  func2.contramap(func1)
```

しかし、この `contramap` を用いた例は、実際にはコンパイルできない。

```scala mdoc:silent
import cats.syntax.contravariant.* // contramap
```

```scala mdoc:fail
val func3c = func2.contramap(func1)
```

`Function1` に `Contravariant` 型クラスインスタンスを定義するには、戻り値の型を固定し引数の型を可変にする。これを型エイリアスで表すと以下のようになる。また、`contramap` の型チャートを図[@fig:functors:function-contramap-type-chart]に示す。

```scala
type F[A] = A => Double
```

![Type chart: contramapping over a Function1](src/pages/functors/function-contramap.pdf+svg){#fig:functors:function-contramap-type-chart}

そのように定義された `Contravariant` 型クラスインスタンスを探すにあたって、コンパイラは型パラメータを右から左へと除去する必要がある。

コンパイラが失敗するのは、単純に、型パラメータの除去を左から右の順に行うというバイアスのせいである。このことは、`Function1` のパラメータを反転する型エイリアスを定義すれば確かめられる。

```scala mdoc:silent
type <=[B, A] = A => B
```
``` scala
type F[A] = Double <= A
```

`func2` を `<=` のインスタンスとして型注釈すれば、型パラメータ除去の順序が逆になり、望みどおりに `contramap` を呼び出すことが可能となる。

```scala mdoc:silent
val func2b: Double <= Double = func2
```

```scala mdoc
val func3c = func2b.contramap(func1)
```

`func2` と `func2b` の違いは純粋に構文的なものである。どちらも参照している値は同じだし、型エイリアスには構文上の違いを除けば完全に互換性がある。しかし驚くべきことに、この簡単な書き換えだけで、問題を解決するために必要なヒントをコンパイラに与えることができる。

このような右から左への型パラメータ除去を行わなければならないことは稀である。ほとんどの多パラメータ型コンストラクタは右バイアスで設計されており、コンパイラが標準でサポートしている左から右への除去に適合する。しかし、上述のような奇妙なケースに遭遇した場合に備えて、この除去順序の特性を知っておくことは有益である。
