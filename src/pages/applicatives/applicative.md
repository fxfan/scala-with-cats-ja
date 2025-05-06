<!--

## Apply and Applicative

Semigroupals aren't mentioned frequently
in the wider functional programming literature.
They provide a subset of the functionality of a related type class
called an *applicative functor* ("applicative" for short).

`Semigroupal` and `Applicative` effectively provide
alternative encodings of the same notion of joining contexts.
Both encodings are introduced in
the [same 2008 paper][link-applicative-programming]
by Conor McBride and Ross Paterson[^semigroupal-monoidal].

[^semigroupal-monoidal]: Semigroupal is referred to as "monoidal" in the paper.

Cats models applicatives using two type classes.
The first, [`cats.Apply`][cats.Apply],
extends `Semigroupal` and `Functor`
and adds an `ap` method that applies a parameter
to a function within a context.
The second, [`cats.Applicative`][cats.Applicative],
extends `Apply` and adds the `pure` method
introduced in Chapter [@sec:monads].
Here's a simplified definition in code:

```scala
trait Apply[F[_]] extends Semigroupal[F] with Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)] =
    ap(map(fa)(a => (b: B) => (a, b)))(fb)
}

trait Applicative[F[_]] extends Apply[F] {
  def pure[A](a: A): F[A]
}
```

Breaking this down, the `ap` method applies a parameter `fa`
to a function `ff` within a context `F[_]`.
The `product` method from `Semigroupal`
is defined in terms of `ap` and `map`.

Don't worry too much about the implementation of `product`---it's
difficult to read and the details aren't particuarly important.
The main point is that there is a tight relationship
between `product`, `ap`, and `map`
that allows any one of them to be defined
in terms of the other two.

`Applicative` also introduces the `pure` method.
This is the same `pure` we saw in `Monad`.
It constructs a new applicative instance from an unwrapped value.
In this sense, `Applicative` is related to `Apply`
as `Monoid` is related to `Semigroup`.

### The Hierarchy of Sequencing Type Classes

With the introduction of `Apply` and `Applicative`,
we can zoom out and see a whole family of type classes
that concern themselves with sequencing computations in different ways.
Figure [@fig:applicatives:hierarchy] shows
the relationship between the type classes covered in this book[^cats-infographic].

![Monad type class hierarchy](src/pages/applicatives/hierarchy.png){#fig:applicatives:hierarchy}

[^cats-infographic]: See
[Rob Norris' infographic][link-cats-infographic]
for a the complete picture.

Each type class in the hierarchy
represents a particular set of sequencing semantics,
introduces a set of characteristic methods,
and defines the functionality of its supertypes
in terms of them:

- every monad is an applicative;
- every applicative a semigroupal;
- and so on.

Because of the lawful nature of
the relationships between the type classes,
the inheritance relationships are constant
across all instances of a type class.
`Apply` defines `product` in terms of `ap` and `map`;
`Monad` defines `product`, `ap`, and `map`,
in terms of `pure` and `flatMap`.

To illustrate this let's consider two hypothetical data types:

- `Foo` is a monad.
  It has an instance of the `Monad` type class
  that implements `pure` and `flatMap`
  and inherits standard definitions of `product`, `map`, and `ap`;

- `Bar` is an applicative functor.
  It has an instance of `Applicative`
  that implements `pure` and `ap`
  and inherits standard definitions of `product` and `map`.

What can we say about these two data types
without knowing more about their implementation?

We know strictly more about `Foo` than `Bar`:
`Monad` is a subtype of `Applicative`,
so we can guarantee properties of `Foo` (namely `flatMap`)
that we cannot guarantee with `Bar`.
Conversely, we know that `Bar`
may have a wider range of behaviours than `Foo`.
It has fewer laws to obey (no `flatMap`),
so it can implement behaviours that `Foo` cannot.

This demonstrates the classic trade-off of power
(in the mathematical sense) versus constraint.
The more constraints we place on a data type,
the more guarantees we have about its behaviour,
but the fewer behaviours we can model.

Monads happen to be a sweet spot in this trade-off.
They are flexible enough to model a wide range of behaviours
and restrictive enough to give strong guarantees about those behaviours.
However, there are situations where monads
aren't the right tool for the job.
Sometimes we want Thai food,
and burritos just won't satisfy.

Whereas monads impose a strict *sequencing*
on the computations they model,
applicatives and semigroupals impose no such restriction.
This puts them in a different sweet spot in the hierarchy.
We can use them to represent
classes of parallel / independent computations
that monads cannot.

We choose our semantics by choosing our data structures.
If we choose a monad, we get strict sequencing.
If we choose an applicative, we lose the ability to `flatMap`.
This is the trade-off enforced by the consistency laws.
So choose your types carefully!


```scala mdoc:reset:silent
```
--->

## `Apply` と `Applicative`

`Semigroupal` は関数型プログラミングについての多くの文献ではあまり言及されない。`Semigroupal` が、*アプリカティブファンクタ*（略してアプリカティブ）と呼ばれる型クラスの機能の一部を提供するものだからである。

`Semigroupal` と `Applicative` は実質的に、コンテキストの結合という同じアイデアを別々のやり方で表現したものである。これらふたつの表現は Conor McBride と Ross Paterson による[2008年の論文][link-applicative-programming]で紹介されている[^semigroupal-monoidal]。

[^semigroupal-monoidal]: Semigroupalは、この論文ではモノイダル（monoidal）と呼ばれている。

Cats ではふたつの型クラスを用いてアプリカティブをモデリングしている。最初の型クラスである [`cats.Apply`][cats.Apply] は、`Semigroupal` と `Functor` を拡張し、コンテキストの中で関数をパラメータに適用する `ap` メソッドを追加する。ふたつ目の型クラスである [`cats.Applicative`][cats.Applicative] は、`Apply` を拡張し、[@sec:monads]章で紹介した `pure` メソッドを追加する。簡略化した定義を以下にコードで示す。

```scala
trait Apply[F[_]] extends Semigroupal[F] with Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)] =
    ap(map(fa)(a => (b: B) => (a, b)))(fb)
}

trait Applicative[F[_]] extends Apply[F] {
  def pure[A](a: A): F[A]
}
```

この定義の中身を説明すると、`ap` メソッドは、コンテキスト `F[_]` の内部で関数 `ff` をパラメータ `fa` に適用する操作である。`Semigroupal` の `product` メソッドは `ap` と `map` を用いて定義されている。

`product` の実装についてあまり深く考える必要はない。理解しづらいのに加え、詳細はそれほど重要ではない。重要なのは、`product`、`ap`、`map` には密接な関係があり、これらはいずれも自分以外のふたつを用いて定義できるという点である。

`Applicative` では、更に `pure` メソッドが導入される。これは `Monad` で見たのと同じ `pure` である。ラップされていない値からアプリカティブ型のコンテキストに包まれた値を新しく構築する。この意味で、`Applicative` と `Apply` の関係は、`Monoid` と `Semigroup` の関係に似ている。

### 計算を連結する型クラスの階層構造

`Apply` と `Applicative` を紹介したことで、計算をそれぞれの方法で連結する一連の型クラスについて全体像を見渡すことができるようになった。図[@fig:applicatives:hierarchy]は、本書で扱った型クラス同士の関係を示している[^cats-infographic]。

![モナド型クラスの階層図](src/pages/applicatives/hierarchy.png){#fig:applicatives:hierarchy}

[^cats-infographic]: 完全な図については [Rob Norris 氏の Cats Infographic][link-cats-infographic] を参照。

階層図に記載されている型クラスはそれぞれ特定の連結セマンティクスを表す。特徴的な一連のメソッドを導入し、それらのメソッドを用いて上位型の機能を定義する。

- すべてのモナドはアプリカティブである
- すべてのアプリカティブは Semigroupal である
- など

型クラス間の関係は法則に従うため、ある型クラスのすべてのインスタンスにおいて継承関係は一貫している。たとえば、`Apply` は `ap` と `map` を使って `product` を定義するし、`Monad` は `pure` と `flatMap` を使って `product`、`ap`、`map` を定義する。

これを説明するために、ふたつの架空のデータ型について考えてみよう。

- `Foo` はモナドである。`pure` と `flatMap` を実装し、`product`、`map`、`ap` の標準的な定義を継承した `Monad` 型クラスインスタンスをもっている。
- `Bar` はアプリカティブファンクタである。`pure` と `ap` を実装し、`product` と `map` の標準的な定義を継承した `Applicative` 型クラスインスタンスをもっている

これらふたつのデータ型について、実装の詳細を知らずに何が言えるだろうか？

`Foo` についての情報量は `Bar` よりも多い。なぜなら、`Monad` は `Applicative` の部分型であり、`Bar` には保証できない特性である `flatMap` の存在を `Foo` は保証できるからである。 逆に、`Bar` は `Foo` よりも広範囲な振る舞いをもつ場合がある。`Bar` のほうが従うべき法則が少ないので（`flatMap` の存在を保証しなくてよい）、`Foo` には実現できない振る舞いを実装することができる。

このことは、数学的な意味でのパワーと、制約との間のよくあるトレードオフを示している。データ型に多くの制約を課すほど、その振る舞いについて強い保証が得られるが、モデリングできる振る舞いの幅は狭くなる。

モナドはこのトレードオフの中で絶妙な位置にある。幅広い振る舞いをモデリングできる柔軟性をもちながら、それらの振る舞いに強い保証を与える制約も、同時にもち合わせている。しかし、モナドが課題解決の正しい選択肢とは言えない場面も存在する。タイ料理が食べたい時にブリトーで満足することはできないのである。

モナドが計算に厳密な順序付けを課す一方で、アプリカティブや Semigroupal にはそのような制約がない。そういった特性の違いから、これらの型クラスはそれぞれ、階層の中に適したポジションをもっている。アプリカティブや Semigroupal には、モナドが扱うことのできない並列的で独立した計算を表現することができるのである。

データ構造を選べばセマンティクスも決まる。モナドを選べば、厳密な順序付けが得られる。アプリカティブを選べば、`flatMap` は使えなくなる。これが一貫性のある法則によって強いられるトレードオフである。だから、型の選択は慎重に行ってほしい。
