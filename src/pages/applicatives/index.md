<!--

# Semigroupal and Applicative {#sec:applicatives}

In previous chapters we saw
how functors and monads let us
sequence operations using `map` and `flatMap`.
While functors and monads are
both immensely useful abstractions,
there are certain types of program flow
that they cannot represent.

One such example is form validation.
When we validate a form we want to
return *all* the errors to the user,
not stop on the first error we encounter.
If we model this with a monad like `Either`,
we fail fast and lose errors.
For example, the code below
fails on the first call to `parseInt`
and doesn't go any further:

```scala mdoc:silent
import cats.syntax.either._ // for catchOnly

def parseInt(str: String): Either[String, Int] =
  Either.catchOnly[NumberFormatException](str.toInt).
    leftMap(_ => s"Couldn't read $str")
```

```scala mdoc
for {
  a <- parseInt("a")
  b <- parseInt("b")
  c <- parseInt("c")
} yield (a + b + c)
```

Another example is the concurrent evaluation of `Futures`.
If we have several long-running independent tasks,
it makes sense to execute them concurrently.
However, monadic comprehension
only allows us to run them in sequence.
`map` and `flatMap` aren't quite capable
of capturing what we want because
they make the assumption that each computation
is *dependent* on the previous one:

```scala
// context2 is dependent on value1:
context1.flatMap(value1 => context2)
```

The calls to `parseInt` and `Future.apply` above
are *independent* of one another,
but `map` and `flatMap` can't exploit this.
We need a weaker construct---one
that doesn't guarantee sequencing---to
achieve the result we want.
In this chapter we will look at three type classes
that support this pattern:

  - `Semigroupal` encompasses
    the notion of composing pairs of contexts.
    Cats provides a [`cats.syntax.apply`][cats.syntax.apply] module
    that makes use of `Semigroupal` and `Functor`
    to allow users to sequence functions with multiple arguments.
    
  - `Parallel` converts
    types with a `Monad` instance
    to a related type with a `Semigroupal` instance.

  - `Applicative` extends `Semigroupal` and `Functor`.
    It provides a way of applying functions to parameters within a context.
    `Applicative` is the source of the `pure` method
    we introduced in Chapter [@sec:monads].

Applicatives are often formulated in terms of function application,
instead of the semigroupal formulation that is emphasised in Cats.
This alternative formulation provides a link
to other libraries and languages such as Scalaz and Haskell.
We'll take a look at different formulations of Applicative,
as well as the relationships between
`Semigroupal`, `Functor`, `Applicative`, and `Monad`,
towards the end of the chapter.


```scala mdoc:reset:silent
```
--->

# `Semigroupal` と `Applicative` {#sec:applicatives}

ここまで、ファンクターやモナドに対して `map` や `flatMap` を使用し、操作を順序付けて連結する方法について見てきた。ファンクターとモナドはどちらも極めて有用な抽象概念だが、これらでは表現できない種類のプログラムフローも存在する。

その一例がフォームのバリデーションである。フォームのバリデーションにおいては、最初に見つかったエラーで処理を中断するのではなく、*すべての*エラーをユーザに返したい。これを `Either` のようなモナドでモデリングすると、最初のエラーが発生した時点で処理は終了してしまい、他のエラーは失われる。たとえば、以下のコードは最初の `parseInt` の呼び出しで失敗し、それ以降の処理は行われない。

```scala mdoc:silent
import cats.syntax.either._ // catchOnly

def parseInt(str: String): Either[String, Int] =
  Either.catchOnly[NumberFormatException](str.toInt).
    leftMap(_ => s"Couldn't read $str")
```

```scala mdoc
for {
  a <- parseInt("a")
  b <- parseInt("b")
  c <- parseInt("c")
} yield (a + b + c)
```

もうひとつの例は `Future` の並行処理である。時間のかかる独立したタスクが複数ある場合に、それらを並行して実行するのは理にかなっている。しかし、モナド内包表記ではそれらを順次実行することしかできない。`map` や `flatMap` は、各計算が前の結果に依存していることを想定している。そのようなモデルでは並行処理に求められる動作を捉えることはできない。

```scala
// context2 は value1 に依存している
context1.flatMap(value1 => context2)
```

前述の `parseInt` や `Future.apply` 呼び出しは互いに独立しているが、`map` と `flatMap` はその事実を活用できない。ここで必要なのは計算順序を保証しないもっと制約の弱い構造である。この章では、このパターンをサポートする三つの型クラスを紹介する。

  - `Semigroupal` は、コンテキストをペアとして組み合わせるという発想を広く包含している。Cats が提供している [`cats.syntax.apply`][cats.syntax.apply] パッケージの構文を使えば、`Semigroupal` と `Functor` を利用することで、コンテキストに包まれた値に対して複数の引数をもつ関数を段階的に適用できる。
  - `Parallel` は、`Monad` インスタンスをもつ型を、それに対応する `Semigroupal` インスタンスをもつ型に変換する。
  - `Applicative` は `Semigroupal` と `Functor` を拡張し、コンテキストの中で関数をパラメータに適用する方法を提供する。`Applicative` は、[@sec:monads]章で紹介した `pure` メソッドの起源である。

アプリカティブは、Cats で強調されている Semigroupal による定式化ではなく、関数適用の観点から定式化されることが多い。このもうひとつの定式化は、Scalaz や Haskell のような他のライブラリや言語との接点を提供してくれる。この章の終盤では、アプリカティブのさまざまな定式化について学ぶ。また、`Semigroupal`、`Functor`、`Applicative`、`Monad` といった計算を連結する一連の型クラス同士の関係性についても合わせて見ていく。
