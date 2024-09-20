## Identityモナド The Identity Monad {#sec:monads:identity}

In the previous section we demonstrated Cats' `flatMap` and `map` syntax
by writing a method that abstracted over different monads:

前節では、異なる種類のモナドを抽象化したメソッドを定義して、Catsが提供する `flatMap` と `map` の構文を実際に使ってみせた。


```scala mdoc:silent
import cats.Monad
import cats.syntax.functor.* // for map
import cats.syntax.flatMap.* // for flatMap

def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  for {
    x <- a
    y <- b
  } yield x*x + y*y
```

This method works well on `Options` and `Lists`
but we can't call it passing in plain values:

このメソッドは `Option` や `List` に対しては正しく動作するが、単なる値を引数にして呼び出すことはできない。

```scala mdoc:fail
sumSquare(3, 4)
```

It would be incredibly useful if we could use `sumSquare`
with parameters that were either in a monad or not in a monad at all.
This would allow us to abstract over monadic and non-monadic code.
Fortunately, Cats provides the `Id` type to bridge the gap:

もし、モナドに包まれているかどうかと関係なく `sumSquare` を使えるとしたら、非常に便利だろう。これは、モナド的なコードと非モナド的なコードを抽象化できるということである。ありがたいことに、Catsはこのギャップを埋めるための `Id` 型を提供している。

```scala mdoc:silent
import cats.Id
```

```scala mdoc
sumSquare(3 : Id[Int], 4 : Id[Int])
```

`Id` allows us to call our monadic method using plain values.
However, the exact semantics are difficult to understand.
We cast the parameters to `sumSquare` as `Id[Int]`
and received an `Id[Int]` back as a result!

`Id` によって、通常の値を使ってモナド用のメソッドを呼び出すことが可能となる。だが、その正確な意味はすこしわかりにくい。`sumSquare` のパラメータを `Id[Int]` 型に変換し、結果として `Id[Int]` 型の値を受け取ったことになる。

What's going on? Here is the definition of `Id` to explain:

一体何が起こっているのだろうか。`Id` の定義は以下のとおりである。

```scala
package cats

type Id[A] = A
```

`Id` is actually a type alias
that turns an atomic type into a single-parameter type constructor.
We can cast any value of any type to a corresponding `Id`:

実は `Id` は、基本的な型を単一の型パラメータをもつ型コンストラクタとして書き表すための型エイリアスである。これにより、任意の型の値を、対応する `Id` 型へとキャストすることができる。

```scala mdoc
"Dave" : Id[String]
123 : Id[Int]
List(1, 2, 3) : Id[List[Int]]
```

Cats provides instances of various type classes for `Id`,
including `Functor` and `Monad`.
These let us call `map`, `flatMap`, and `pure`
on plain values:

Catsは、`Functor` や `Monad` を含むさまざまな型クラスに対して、`Id` 用のインスタンスを提供している。それらのおかげで、単なる値に対して `map` や `flatMap` や `pure` を呼び出すことができる。

```scala mdoc
val a = Monad[Id].pure(3)
val b = Monad[Id].flatMap(a)(_ + 1)
```

```scala mdoc:silent
import cats.syntax.functor.* // for map
import cats.syntax.flatMap.* // for flatMap
```

```scala mdoc
for {
  x <- a
  y <- b
} yield x + y
```

The ability to abstract over monadic and non-monadic code
is extremely powerful.
For example,
we can run code asynchronously in production using `Future`
and synchronously in test using `Id`.
We'll see this in our first case study
in Chapter [@sec:case-studies:testing].

モナド的なコードとそうでないコードを抽象化する能力は極めて強力である。たとえば、コードを、本番環境では `Future` を使って非同期に、テスト環境では `Id` を使って同期的に実行できる。これについては[@sec:case-studies:testing]章の最初のケーススタディで見るつもりである。

### 演習: モナドの隠された正体 Monadic Secret Identities

Implement `pure`, `map`, and `flatMap` for `Id`!
What interesting discoveries do you uncover about the implementation?

`Id` のための `pure` と `map` と `flatMap` を実装せよ。実装の中でどのような興味深い発見があるか確認すること。

<div class="solution">
Let's start by defining the method signatures:

まず、メソッドのシグネチャを定義しよう。

```scala mdoc:silent
import cats.Id

def pure[A](value: A): Id[A] =
  ???

def map[A, B](initial: Id[A])(func: A => B): Id[B] =
  ???

def flatMap[A, B](initial: Id[A])(func: A => Id[B]): Id[B] =
  ???
```

Now let's look at each method in turn.
The `pure` operation creates an `Id[A]` from an `A`.
But `A` and `Id[A]` are the same type!
All we have to do is return the initial value:

次に、それぞれのメソッドを順に見ていく。

`pure` は `A` から `Id[A]` を生成する操作である。しかし `A` と `Id[A]` は同じ型なので、単に初期値を返せばよい。

```scala mdoc:invisible:reset-object
import cats.{Id,Monad}
import cats.syntax.functor.* 
import cats.syntax.flatMap.*
def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  for {
    x <- a
    y <- b
  } yield x*x + y*y
```
```scala mdoc:silent
def pure[A](value: A): Id[A] =
  value
```

```scala mdoc
pure(123)
```

The `map` method takes a parameter of type `Id[A]`,
applies a function of type `A => B`, and returns an `Id[B]`.
But `Id[A]` is simply `A` and `Id[B]` is simply `B`!
All we have to do is call the function---no boxing or unboxing required:

`map` メソッドは `Id[A]` 型のパラメータを取り、`A => B` 型の関数を適用して `Id[B]` を返す。だが、`Id[A]` は `A` で、`Id[B]` は `B` なので、必要なのは関数を呼び出すことだけである。コンテキストに出し入れする必要はない。

```scala mdoc:silent
def map[A, B](initial: Id[A])(func: A => B): Id[B] =
  func(initial)
```

```scala mdoc
map(123)(_ * 2)
```

The final punch line is that,
once we strip away the `Id` type constructors,
`flatMap` and `map` are actually identical:

そして

```scala mdoc
def flatMap[A, B](initial: Id[A])(func: A => Id[B]): Id[B] =
  func(initial)
```

```scala mdoc
flatMap(123)(_ * 2)
```

This ties in with our understanding of functors and monads
as sequencing type classes.
Each type class allows us to sequence operations
ignoring some kind of complication.
In the case of `Id` there is no complication,
making `map` and `flatMap` the same thing.

これは、ファンクターやモナドが計算を順番に繋げていくための型クラスであるという理解と結びつく。それらの型クラスはいずれも、ある種の複雑性を無視した計算の連結を可能にしてくれる。だが、`Id` の場合は複雑性が存在しないため、`map` と `flatMap` は同じものになる。

Notice that we haven't had to write type annotations
in the method bodies above.
The compiler is able to interpret values of type `A` as `Id[A]` and vice versa
by the context in which they are used.

また、上記のメソッド本体には型注釈を書く必要がないことに注目してほしい。コンパイラは文脈に基づいて `A` 型の値を `Id[A]` として解釈することができるし、その逆も同様である。

The only restriction we've seen to this is that Scala cannot unify
types and type constructors when searching for given instances.
Hence our need to re-type `Int` as `Id[Int]`
in the call to `sumSquare` at the opening of this section:

ここで見た唯一の制約は、Scalaがgivenインスタンスを検索する際に、型と型コンストラクタを統一できないことだ。そのため、このセクションの冒頭で `sumSquare` を呼び出す際に、`Int` を `Id[Int]` として再型付けする必要があった。

<!--
TODO: Scala cannot unify types and type constructors
よくわからないが、おそらく問題は、givenインスタンスの検索という文脈に限定されるものではなく
Id[Int]が求められる場所にIntが来た場合は、「IntをId[Int]に変換可能か」がチェックされ、それは可能と判断されるが、F[Int]が求められるところにIntが来たときに、無数にありうるFの中からIdを選択するような推論はできない、みたいなことなんじゃないかと思う。
-->

```scala mdoc:silent
sumSquare(3 : Id[Int], 4 : Id[Int])
```
</div>
