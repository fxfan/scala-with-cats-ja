# ファンクター Functors

In this chapter we will investigate **functors**,
an abstraction that allows us to
represent sequences of operations within a context
such as a `List`, an `Option`,
or any one of thousands of other possibilities.
Functors on their own aren't so useful,
but special cases of functors, 
such as **monads** and **applicative functors**,
are some of the most commonly used abstractions.

この章では**ファンクター**について探っていく。ファンクターは、`List` や `Option` や、その他無数に存在しうる様々なコンテキストの中で行われる操作の連なりを表現するための抽象概念である。ファンクター単体ではあまり役立たないが、**モナド**や**アプリカティブファンクター**といったファンクターの特殊なケースは、極めて頻繁に使用される抽象概念のひとつである。

## ファンクターの例 Examples of Functors {#sec:functors:examples}

Informally, a functor is anything with a `map` method.
You probably know lots of types that have this:
`Option`, `List`, and `Either`, to name a few.

一般的に、ファンクターとは `map` メソッドをもつものを指す。知ってのとおり、`Option` や `List` や `Either` など、多くの型がこれに該当する。

We typically first encounter `map` when iterating over `Lists`.
However, to understand functors
we need to think of the method in another way.
Rather than traversing the list, we should think of it as
transforming all of the values inside in one go.
We specify the function to apply,
and `map` ensures it is applied to every item.
The values change but the structure of the list
(the number of elements and their order)
remains the same:

通常、最初に `map` に出会うのは、`List` に対して繰り返し処理を行うときだろう。しかし、ファンクターを理解するためには、このメソッドを別の見方で捉える必要がある。`map` を単にリストを走査するものではなく、内部のすべての値を一度に変換するものとして考えよう。適用する関数を指定すれば、`map` はそれを各要素に対して適用してくれる。値は変わるが、リストの構造（要素数や順序）は変わらない。

```scala mdoc
List(1, 2, 3).map(n => n + 1)
```

Similarly, when we `map` over an `Option`,
we transform the contents but leave
the `Some` or `None` context unchanged.
The same principle applies to `Either`
with its `Left` and `Right` contexts.
This general notion of transformation,
along with the common pattern of type signatures
shown in Figure [@fig:functors:list-option-either-type-chart],
is what connects the behaviour of `map`
across different data types.

同様に、`Option` に対して `map` を実行すると、`Some` や `None` というコンテキストは変わらずに内部の値が変換される。この原則は、`Either` とそのコンテキストである `Left` や `Right` についても言える。この一般化された変換の概念と、型シグネチャにおけるのパターン（図 [@fig:functors:list-option-either-type-chart] を参照）が、異なるデータ型における `map` の挙動を結びつけている。

![Type chart: mapping over List, Option, and Either](src/pages/functors/list-option-either-map.pdf+svg){#fig:functors:list-option-either-type-chart}

Because `map` leaves the structure of the context unchanged,
we can call it repeatedly to sequence multiple computations
on the contents of an initial data structure:

`map` がコンテキストの構造を変えずに残すため、最初のデータ構造の内容に対して複数の計算を続けて呼び出すことができる。

```scala mdoc
List(1, 2, 3).
  map(n => n + 1).
  map(n => n * 2).
  map(n => s"${n}!")
```

We should think of `map` not as an iteration pattern,
but as a way of sequencing computations
on values ignoring some complication
dictated by the relevant data type:

- `Option`---the value may or may not be present;
- `Either`---there may be a value or an error;
- `List`---there may be zero or more values.

`map` は繰り返し処理ではなく、下記のようなデータ型特有の性質を無視して、値に対する計算を順序付ける方法だと考えるべきである。

- `Option`---値は存在するかもしれないし、しないかもしれない
- `Either`---保持されているのは値かもしれないし、エラーかもしれない
- `List`---ゼロ個もしくは複数の値が含まれるかもしれない

## ファンクターのさらなる例 More Examples of Functors {#sec:functors:more-examples}

The `map` methods of `List`, `Option`, and `Either`
apply functions eagerly.
However, the idea of sequencing computations
is more general than this.
Let's investigate the behaviour of some other functors
that apply the pattern in different ways.

`List`、`Option`、および `Either` の `map` メソッドは、関数を即時に適用する。しかし、計算を順序付けるという発想は、もっと一般化できる。別の方法でこのパターンを適用する他のファンクターについて、その挙動を探ってみよう。

**Future**

`Future` is a functor that
sequences asynchronous computations by queueing them
and applying them as their predecessors complete.
The type signature of its `map` method,
shown in Figure [@fig:functors:future-type-chart],
has the same shape as the signatures above.
However, the behaviour is very different.

`Future` はファンクターの一種で、非同期な計算を順序付けする。計算処理はキューに入れられ、前の計算が完了した後で適用される。図 [@fig:functors:future-type-chart] に示す `map` メソッドの型シグネチャは、前述のシグネチャと同じ形をしている。しかし、その挙動はまったく異なる。

![Type chart: mapping over a Future](src/pages/functors/future-map.pdf+svg){#fig:functors:future-type-chart}

When we work with a `Future` we have no guarantees
about its internal state.
The wrapped computation may be
ongoing, complete, or rejected.
If the `Future` is complete,
our mapping function can be called immediately.
If not, some underlying thread pool queues
the function call and comes back to it later.
We don't know *when* our functions will be called,
but we do know *what order* they will be called in.
In this way, `Future` provides
the same sequencing behaviour
seen in `List`, `Option`, and `Either`:

`Future` を扱うとき、その内部状態については保証がない。ラップされている計算は進行中かもしれないし、完了しているか、あるいは失敗しているかもしれない。`Future` が完了している場合、マッピング関数はただちに呼び出される。しかし、そうでない場合、基盤となるスレッドプールが関数呼び出しをキューに入れ、後で改めて呼び出しを行う。関数が*いつ*呼び出されるかはわからないが、*どの順序*で呼び出されるかはわかる。このように、`Future` は `List`、`Option`、そして `Either` で見られたのと同じ順序付けの振る舞いを提供している。

```scala mdoc:silent
import scala.concurrent.{Future, Await}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

val future: Future[String] =
  Future(123).
    map(n => n + 1).
    map(n => n * 2).
    map(n => s"${n}!")
```

```scala mdoc
Await.result(future, 1.second)
```

<div class="callout callout-info">
*Futures and Referential Transparency*

*`Future` と参照透過性*

Note that Scala's `Futures` aren't a great example
of pure functional programming
because they aren't *referentially transparent*.
`Future` always computes and caches a result
and there's no way for us to tweak this behaviour.
This means we can get unpredictable results
when we use `Future` to wrap side-effecting computations.
For example:

Scalaの `Future` は参照透過性をもっていないため、純粋関数型プログラミングのよい例とは言えない点には注意してほしい。`Future` は常に結果を計算してキャッシュする。その挙動を調整する手段は提供されていない。そのため、副作用を伴う計算を `Future` でラップすると、予測できない結果を引き起こす可能性がある。以下に例を挙げよう。

```scala mdoc:silent
import scala.util.Random

val future1 = {
  // シードを固定してRandomを初期化する
  val r = new Random(0L)

  // nextInt has the side-effect of moving to
  // the next random number in the sequence:
  // nextIntには、乱数シーケンスの中を次の乱数へと進める副作用がある
  val x = Future(r.nextInt())

  for {
    a <- x
    b <- x
  } yield (a, b)
}

val future2 = {
  val r = new Random(0L)

  for {
    a <- Future(r.nextInt())
    b <- Future(r.nextInt())
  } yield (a, b)
}
```

```scala mdoc
val result1 = Await.result(future1, 1.second)
val result2 = Await.result(future2, 1.second)
```

Ideally we would like `result1` and `result2`
to contain the same value.
However, the computation for `future1` calls `nextInt` once
and the computation for `future2` calls it twice.
Because `nextInt` returns a different result every time
we get a different result in each case.

理想的には `result1` と `result2` は等しくなることが望ましいが、両者は異なる結果となる。`nextInt` が毎回異なる結果を返すことに加え、`future1` の計算では `nextInt` が一度しか呼ばれないが、`future2` の計算では二度呼ばれるためである。

This kind of discrepancy makes it hard to reason about
programs involving `Futures` and side-effects.
There also are other problematic aspects of `Future's` behaviour,
such as the way it always starts computations immediately
rather than allowing the user to dictate when the program should run.
For more information
see [this excellent Reddit answer][link-so-future]
by Rob Norris.

このような不一致により、`Future` と副作用を扱うプログラムについて推論するのは難しくなる。`Future` の挙動には他にも問題がある。たとえば、`Future` は常に計算を即座に開始し、プログラムの実行タイミングについてユーザが指定することを認めない。詳細については、[Rob NorrisがRedditに投稿した素晴らしい回答][link-so-future]を参照してほしい。

When we look at Cats Effect 
we'll see that the `IO` type solves these problems.

Cats Effectについて見ていくと、`IO` 型がこれらの問題を解決していることがわかるだろう。
</div>

If `Future` isn't referentially transparent,
perhaps we should look at another similar data-type that is.
You should recognise this one...

`Future` が参照透過性を持たないのであれば、参照透過性を持つ別の類似データ型を検討すべきかもしれない。きっとこれには見覚えがあるはずだ...

<!-- TODO: きっとこれには見覚えがあるはずだ... -->

**関数 (?!)**

It turns out that single argument functions are also functors.
To see this we have to tweak the types a little.
A function `A => B` has two type parameters:
the parameter type `A` and the result type `B`.
To coerce them to the correct shape we can
fix the parameter type and let the result type vary:

 - start with `X => A`;
 - supply a function `A => B`;
 - get back `X => B`.

実は、単一引数の関数もファンクターである。これを理解するには、型をすこし調整する必要がある。関数 `A => B` は、引数の型 `A` と、戻り値の型 `B` という二つの型パラメータをもっている。これをファンクターの形に強制的に当てはめるためには、引数の型を固定し、戻り値の型を可変にすればよい。

- はじめに `X => A` 型の関数があると考える
- 関数 `A => B` を与えて `map` する
- `X => B` 型の関数が得られる

If we alias `X => A` as `MyFunc[A]`,
we see the same pattern of types
we saw with the other examples in this chapter.
We also see this in Figure [@fig:functors:function-type-chart]:

 - start with `MyFunc[A]`;
 - supply a function `A => B`;
 - get back `MyFunc[B]`.

`X => A` に対して `MyFunc[A]` という別名を付けると、この章の他の例で見たのと同じ型のパターンが見えてくる。このことは図 [@fig:functors:function-type-chart] でも確認できる。

- `MyFunc[A]` があるとする
- 関数 `A => B` を与えて `map` する
- `MyFunc[B]` が得られる

![Type chart: mapping over a Function1](src/pages/functors/function-map.pdf+svg){#fig:functors:function-type-chart}

In other words, "mapping" over a `Function1` is function composition:

つまり、`Function1` に対して「マッピング」を行うということは、関数の合成を意味する。

```scala mdoc:silent
import cats.syntax.all.*     // map をインポート

val func1: Int => Double =
  (x: Int) => x.toDouble

val func2: Double => Double =
  (y: Double) => y * 2
```

```scala mdoc
(func1.map(func2))(1)     // mapを用いた合成
(func1.andThen(func2))(1) // andThenを用いた合成
func2(func1(1))           // 手書きで合成
```

How does this relate to our general pattern
of sequencing operations?
If we think about it,
function composition *is* sequencing.
We start with a function that performs a single operation
and every time we use `map` we append another operation to the chain.
Calling `map` doesn't actually *run* any of the operations,
but if we can pass an argument to the final function
all of the operations are run in sequence.
We can think of this as lazily queueing up operations
similar to `Future`:

これが操作の順序付けという一般的なパターンとどのように関連しているか考えてみよう。実は関数合成そのものが順序付けだといえる。ある単一の操作を実行する関数から始め、`map` を使うたびにチェーンのように別の操作が追加されていく。`map` を呼び出しても実際に操作が実行されるわけではないが、最終的に生成された関数に引数を渡せば、すべての操作が順番に実行される。この仕組みは、`Future` と同じように操作をキューに入れて遅延させているようなものだと考えることができる。

```scala mdoc:silent
val func =
  ((x: Int) => x.toDouble).
    map(x => x + 1).
    map(x => x * 2).
    map(x => s"${x}!")
```

```scala mdoc
func(123)
```

<div class="callout callout-warning">
*部分的ユニフィケーション*

For the above examples to work,
in versions of Scala before 2.13,
we need to add the following compiler option to `build.sbt`:

上記の例を動作させるためには、Scala 2.13 より前のバージョンでは、`build.sbt` に次のコンパイラオプションを追加する必要がある。

```scala
scalacOptions += "-Ypartial-unification"
```

otherwise we'll get a compiler error:

これをしないとコンパイルエラーになる。

```scala
func1.map(func2)
// <console>: error: value map is not a member of Int => Double
//        func1.map(func2)
                ^
```

We'll look at why this happens in detail
in Section [@sec:functors:partial-unification].

これがなぜ起きるのか、その詳細は[@sec:functors:partial-unification]節で見ていく。
</div>


## ファンクターの定義 Definition of a Functor

Every example we've looked at so far is a functor:
a class that encapsulates sequencing computations.
Formally, a functor is a type `F[A]`
with an operation `map` with type `(A => B) => F[B]`.
The general type chart is shown in
Figure [@fig:functors:functor-type-chart].

これまでに見てきた例はすべてファンクター、つまり、計算の順序付けをカプセル化するクラスである。形式的には、ファンクターは `F[A]` という型をもち、`(A => B) => F[B]` 型の操作 `map` を備えている。一般的なファンクターの型は、図 [@fig:functors:functor-type-chart] に示されたとおりである。

![Type chart: generalised functor map](src/pages/functors/generic-map.pdf+svg){#fig:functors:functor-type-chart}

Cats encodes `Functor` as a type class,
[`cats.Functor`][cats.Functor],
so the method looks a little different.
It accepts the initial `F[A]` as a parameter
alongside the transformation function.
Here's a simplified version of the definition:

Catsでは、ファンクターをを型クラス [`cats.Functor`][cats.Functor] として提供しており、そのため、メソッドの形がこれまでの例とはすこし異なる。この型クラスは、初期値である `F[A]` を変換関数とともにパラメータとして受け取る。以下に簡略化した定義を示す。

```scala
package cats

trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}
```

If you haven't seen syntax like `F[_]` before,
it's time to take a brief detour to discuss
*type constructors* and *higher kinded types*.

`F[_]` のような構文を見るのは初めてかもしれない。ここですこし寄り道をして*型コンストラクタ*と*高カインド型*について説明しておこう。

<div class="callout callout-warning">
*ファンクター則 Functor Laws*

Functors guarantee the same semantics
whether we sequence many small operations one by one,
or combine them into a larger function before `mapping`.
To ensure this is the case the following laws must hold:

ファンクターは、複数の小さな操作をひとつずつ `map` しても、それらをひとつの大きな関数にまとめてから `map` しても、同じ結果になることを保証する。これを確実なものとするために、以下の法則が満たされる必要がある。

*Identity*: calling `map` with the identity function
is the same as doing nothing:

*恒等則*。恒等関数を使って `map` を呼び出すことは、何もしていないのと同じである。

```scala
fa.map(a => a) == fa
```

*Composition*: `mapping` with two functions `f` and `g` is
the same as `mapping` with `f` and then `mapping` with `g`:

*合成則*。二つの関数 `f` と `g` で `map` することは、まず `f` で `map` してから `g` で `map` するのと同じである。

```scala
fa.map(g(f(_))) == fa.map(f).map(g)
```
</div>

## 高カインド型と型コンストラクタ Aside: Higher Kinds and Type Constructors

Kinds are like types for types.
They describe the number of "holes" in a type.
We distinguish between regular types that have no holes
and "type constructors" that have
holes we can fill to produce types.

カインドは型に対する型のようなものであり、型がもつスロットの数を記述するものである。スロットをもたないのが普通の型で、型パラメータというスロットをもち、それを埋めることで型を生成するのが型コンストラクタである。

For example, `List` is a type constructor with one hole.
We fill that hole by specifying a parameter to produce
a regular type like `List[Int]` or `List[A]`.
The trick is not to confuse type constructors with generic types.
`List` is a type constructor, `List[A]` is a type:

たとえば、`List` はひとつのスロットをもつ型コンストラクタであり、そのスロットに型パラメータを指定することで、`List[Int]` や `List[A]` のような通常の型を作り出す。型コンストラクタとジェネリック型を混同しないよう注意したい。`List `は型コンストラクタであり、`List[A]` は型である。

```scala
List    // 型コンストラクタ。パラメータをひとつとる
List[A] // 型パラメータを指定することで生成される型
```

There's a close analogy here with functions and values.
Functions are "value constructors"---they
produce values when we supply parameters:

型コンストラクタと型との関係は、関数と値との関係と深い類似性がある。関数は「値コンストラクタ」であり、パラメータを与えると値を生成する。

```scala
math.abs    // これは関数で、パラメータをひとつ受け取る
math.abs(x) // 値パラメータを渡すことで生成される値
```

In Scala we declare type constructors using underscores.
This specifies how many "holes" the type constructor has.
However, to use them we refer to just the name.

Scalaでは、アンダースコアを使って型コンストラクタを宣言する。これによって、型コンストラクタがいくつのスロットを持つかを指定する。しかし、実際に使用する際には名前だけで参照する。

```scala
// アンダースコアを用いてFを宣言する
def myMethod[F[_]] = {

  // アンダースコアなしのFという名前で参照する
  val functor = Functor.apply[F]

  // ...
}
```

This is analogous to specifying function parameter types.
When we declare a parameter we also give its type.
However, to use them we refer to just the name.

これは、関数のパラメータ型を指定するのと似ている。パラメータを宣言する際には、その型も指定するが、実際に使用するときには名前だけを参照する。

```scala
// Declare f specifying parameter types
// パラメータの型を指定して f を宣言する
def f(x: Int): Int = 
  // Reference x without type
  // パラメータ x を参照するときに型は不要
  x * 2
```

Armed with this knowledge of type constructors,
we can see that the Cats definition of `Functor`
allows us to create instances for any single-parameter type constructor,
such as `List`, `Option`, `Future`, or a type alias such as `MyFunc`.

型コンストラクタに関するこの知識があれば、Catsの `Functor` 定義で、`List`、`Option`、`Future` といった単一の型パラメータを持つ型コンストラクタや、`MyFunc` のような型エイリアスのためのインスタンスを作成可能である、ということが理解できるだろう。

<div class="callout callout-info">
*Language Feature Imports*

In versions of Scala before 2.13
we need to "enable" the higher kinded type language feature,
to suppress warnings from the compiler,
whenever we declare a type constructor with `A[_]` syntax.
We can either do this with a "language import" as above:

Scala 2.13 より前のバージョンでは、`A[_]` という構文で型コンストラクタを宣言する際には、コンパイラの警告を抑制するために高カインド型の言語機能を有効化する必要がある。

これを行うには、次のように言語機能をインポートするか、

```scala
import scala.language.higherKinds
```

or by adding the following to `scalacOptions` in `build.sbt`:

もしくは以下のような `scalacOptions` 指定を `build.sbt` に追加すればよい。

```scala
scalacOptions += "-language:higherKinds"
```

In practice we find the `scalacOptions` flag to be
the simpler of the two options.

実際のところ、`scalacOptions` フラグを使う方が簡単であるように思われる。
</div>
