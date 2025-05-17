<!--

# Case Study: Map-Reduce {#sec:map-reduce}

<!--
TODO:

- DONE - talk about map-reduce - it's just foldMap
- DONE - introduce/reimplement foldMap
- DONE - implement parallelFoldMap to mimic map-reduce
  - DONE - mention that we're specifically imitating multi-machine
           map-reduce where we need to split data between machines
           in large blocks
  - DONE - implement in terms of our foldMap first
  - DONE - then implement in terms of Cats' foldMap
  - DONE - talk about traverse
- summary
  - DONE - real-world map-reduce has communication costs
  - DONE - multi-cpu map-reduce doesn't have communication costs
  - DONE - parallelFoldMap mimics multi-machine
  - DONE - our final version of parallelFoldMap (based on traverse) is far simpler
  - talk about substitution and the things it doesn't model:
    - performance
    - parallelism
    - side-effects (future starts immediately)
    - etc...

TODO:

- DONE - drop the current foldMapM stuff
- DONE - maybe move it elsewhere
->

In this case study we're going to implement
a simple-but-powerful parallel processing framework
using `Monoids`, `Functors`, and a host of other goodies.

If you have used Hadoop or otherwise worked in "big data"
you will have heard of [MapReduce][link-map-reduce],
which is a programming model for doing parallel data processing
across clusters of machines (aka "nodes").
As the name suggests, the model is built around a *map* phase,
which is the same `map` function we know
from Scala and the `Functor` type class, and a *reduce* phase,
which we usually call `fold`[^hadoop-shuffle] in Scala.

[^hadoop-shuffle]: In Hadoop there is also a shuffle phase
that we will ignore here.

## Parallelizing *map* and *fold*

Recall the general signature for `map` is
to apply a function `A => B` to a `F[A]`,
returning a `F[B]`:

![Type chart: functor map](src/pages/functors/generic-map.pdf+svg){#fig:map-reduce:functor-type-chart}

`map` transforms each individual element in a sequence independently.
We can easily parallelize `map` because
there are no dependencies between
the transformations applied to different elements
(the type signature of the function `A => B` shows us this,
assuming we don't use side-effects not reflected in the types).

What about `fold`?
We can implement this step with an instance of `Foldable`.
Not every functor also has an instance of foldable
but we can implement a map-reduce system
on top of any data type that has both of these type classes.
Our reduction step becomes a `foldLeft`
over the results of the distributed `map`.

![Type chart: fold](src/pages/foldable-traverse/generic-foldleft.pdf+svg){#fig:map-reduce:foldleft-type-chart}

By distributing the reduce step
we lose control over the order of traversal.
Our overall reduction may not be entirely left-to-right---we
may reduce left-to-right across several subsequences
and then combine the results.
To ensure correctness we need
a reduction operation that is *associative*:

```scala
reduce(a1, reduce(a2, a3)) == reduce(reduce(a1, a2), a3)
```

If we have associativity,
we can arbitrarily distribute work
between our nodes provided the subsequences
at every node stay in the same order as the initial dataset.

Our fold operation requires us to seed the computation
with an element of type `B`.
Since fold may be split
into an arbitrary number of parallel steps,
the seed should not affect the result of the computation.
This naturally requires the seed to be an *identity* element:

```scala
reduce(seed, a1) == reduce(a1, seed) == a1
```

In summary, our parallel fold will yield the correct results if:

- we require the reducer function to be associative;
- we seed the computation with the identity of this function.

What does this pattern sound like?
That's right, we've come full circle back to `Monoid`,
the first type class we discussed in this book.
We are not the first to recognise the importance of monoids.
The [monoid design pattern for map-reduce jobs][link-map-reduce-monoid]
is at the core of recent big data systems
such as Twitter's [Summingbird][link-summingbird].

In this project we're going to implement
a very simple single-machine map-reduce.
We'll start by implementing a method called `foldMap`
to model the data-flow we need.

## Implementing *foldMap*

We saw `foldMap` briefly back when we covered `Foldable`.
It is one of the derived operations that sits
on top of `foldLeft` and `foldRight`.
However, rather than use `Foldable`,
we will re-implement `foldMap` here ourselves
as it will provide useful insight into
the structure of map-reduce.

Start by writing out the signature of `foldMap`.
It should accept the following parameters:

 - a sequence of type `Vector[A]`;
 - a function of type `A => B`, where there is a `Monoid` for `B`;

You will have to add implicit parameters or context bounds
to complete the type signature.

<div class="solution">
```scala mdoc:silent
import cats.Monoid

/** Single-threaded map-reduce function.
  * Maps `func` over `values` and reduces using a `Monoid[B]`.
  */
def foldMap[A, B: Monoid](values: Vector[A])(func: A => B): B =
  ???
```
</div>

Now implement the body of `foldMap`.
Use the flow chart in Figure [@fig:map-reduce:fold-map] as a guide
to the steps required:

1. start with a sequence of items of type `A`;
2. map over the list to produce a sequence of items of type `B`;
3. use the `Monoid` to reduce the items to a single `B`.

![*foldMap* algorithm](src/pages/case-studies/map-reduce/fold-map.pdf+svg){#fig:map-reduce:fold-map}

Here's some sample output for reference:

```scala mdoc:invisible:reset
import cats.Monoid
import cats.syntax.semigroup._ // for |+|

def foldMap[A, B: Monoid](values: Vector[A])(func: A => B): B =
  values.foldLeft(Monoid[B].empty)(_ |+| func(_))
```

```scala mdoc:silent
import cats.instances.int._ // for Monoid
```

```scala mdoc
foldMap(Vector(1, 2, 3))(identity)
```

```scala mdoc:silent
import cats.instances.string._ // for Monoid
```

```scala mdoc
// Mapping to a String uses the concatenation monoid:
foldMap(Vector(1, 2, 3))(_.toString + "! ")

// Mapping over a String to produce a String:
foldMap("Hello world!".toVector)(_.toString.toUpperCase)
```

<div class="solution">
We have to modify the type signature to accept a `Monoid` for `B`.
With that change we can use the `Monoid` `empty` and `|+|` syntax
as described in Section [@sec:monoid-syntax]:

```scala mdoc:reset:silent
import cats.Monoid
import cats.syntax.semigroup._ // for |+|

def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.map(func).foldLeft(Monoid[B].empty)(_ |+| _)
```

We can make a slight alteration to this code to do everything in one step:

```scala mdoc:reset:invisible
import cats.Monoid
import cats.syntax.semigroup._
```
```scala mdoc:silent
def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.foldLeft(Monoid[B].empty)(_ |+| func(_))
```
</div>

## Parallelising *foldMap*

Now we have a working single-threaded implementation of `foldMap`,
let's look at distributing work to run in parallel.
We'll use our single-threaded version of `foldMap` as a building block.

We'll write a multi-CPU implementation
that simulates the way we would distribute work
in a map-reduce cluster as shown in Figure [@fig:map-reduce:parallel-fold-map]:

1. we start with an initial list of all the data we need to process;
2. we divide the data into batches, sending one batch to each CPU;
3. the CPUs run a batch-level map phase in parallel;
4. the CPUs run a batch-level reduce phase in parallel,
   producing a local result for each batch;
5. we reduce the results for each batch to a single final result.

![*parallelFoldMap* algorithm](src/pages/case-studies/map-reduce/parallel-fold-map.pdf+svg){#fig:map-reduce:parallel-fold-map}

Scala provides some simple tools
to distribute work amongst threads.
We could use the [parallel collections library][link-parallel-collections]
to implement a solution,
but let's challenge ourselves by diving a bit deeper
and implementing the algorithm ourselves using `Futures`.

### *Futures*, Thread Pools, and ExecutionContexts

We already know a fair amount about
the monadic nature of `Futures`.
Let's take a moment for a quick recap,
and to describe how Scala futures
are scheduled behind the scenes.

`Futures` run on a thread pool,
determined by an implicit `ExecutionContext` parameter.
Whenever we create a `Future`,
whether through a call to `Future.apply` or some other combinator,
we must have an implicit `ExecutionContext` in scope:

```scala mdoc:silent
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
val future1 = Future {
  (1 to 100).toList.foldLeft(0)(_ + _)
}

val future2 = Future {
  (100 to 200).toList.foldLeft(0)(_ + _)
}
```

In this example we've imported a `ExecutionContext.Implicits.global`.
This default context allocates a thread pool
with one thread per CPU in our machine.
When we create a `Future`
the `ExecutionContext` schedules it for execution.
If there is a free thread in the pool,
the `Future` starts executing immediately.
Most modern machines have at least two CPUs,
so in our example it is likely that `future1` and `future2`
will execute in parellel.

Some combinators create new `Futures`
that schedule work based on the results of other `Futures`.
The `map` and `flatMap` methods, for example,
schedule computations that run as soon as
their input values are computed and a CPU is available:

```scala mdoc
val future3 = future1.map(_.toString)

val future4 = for {
  a <- future1
  b <- future2
} yield a + b
```

As we saw in Section [@sec:traverse],
we can convert a `List[Future[A]]` to a `Future[List[A]]`
using `Future.sequence`:

```scala mdoc
Future.sequence(List(Future(1), Future(2), Future(3)))
```

or an instance of `Traverse`:

```scala mdoc:silent
import cats.instances.future._ // for Applicative
import cats.instances.list._   // for Traverse
import cats.syntax.traverse._  // for sequence
```

```scala mdoc
List(Future(1), Future(2), Future(3)).sequence
```

An `ExecutionContext` is required in either case.
Finally, we can use `Await.result`
to block on a `Future` until a result is available:

```scala mdoc:silent
import scala.concurrent._
import scala.concurrent.duration._
```

```scala mdoc
Await.result(Future(1), 1.second) // wait for the result
```

There are also `Monad` and `Monoid` implementations for `Future`
available from `cats.instances.future`:

```scala mdoc:silent
import cats.{Monad, Monoid}
import cats.instances.int._    // for Monoid
import cats.instances.future._ // for Monad and Monoid

Monad[Future].pure(42)

Monoid[Future[Int]].combine(Future(1), Future(2))
```

### Dividing Work

Now we've refreshed our memory of `Futures`,
let's look at how we can divide work into batches.
We can query the number of available CPUs on our machine
using an API call from the Java standard library:

```scala mdoc
Runtime.getRuntime.availableProcessors
```

We can partition a sequence
(actually anything that implements `Vector`)
using the `grouped` method.
We'll use this to split off batches of work for each CPU:

```scala mdoc
(1 to 10).toList.grouped(3).toList
```

### Implementing *parallelFoldMap*

Implement a parallel version of `foldMap` called `parallelFoldMap`.
Here is the type signature:

```scala mdoc:silent
def parallelFoldMap[A, B : Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = ???
```

Use the techniques described above to
split the work into batches, one batch per CPU.
Process each batch in a parallel thread.
Refer back to Figure [@fig:map-reduce:parallel-fold-map]
if you need to review the overall algorithm.

For bonus points, process the batches for each CPU
using your implementation of `foldMap` from above.

<div class="solution">
Here is an annotated solution that
splits out each `map` and `fold`
into a separate line of code:

```scala mdoc:invisible:reset
import cats._
import cats.implicits._
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```
```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  // Calculate the number of items to pass to each CPU:
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  // Create one group for each CPU:
  val groups: Iterator[Vector[A]] =
    values.grouped(groupSize)

  // Create a future to foldMap each group:
  val futures: Iterator[Future[B]] =
    groups map { group =>
      Future {
        group.foldLeft(Monoid[B].empty)(_ |+| func(_))
      }
    }

  // foldMap over the groups to calculate a final result:
  Future.sequence(futures) map { iterable =>
    iterable.foldLeft(Monoid[B].empty)(_ |+| _)
  }
}

val result: Future[Int] =
  parallelFoldMap((1 to 1000000).toVector)(identity)
```

```scala mdoc
Await.result(result, 1.second)
```

We can re-use our definition of `foldMap` for a more concise solution.
Note that the local maps and reduces in steps 3 and 4 of
Figure [@fig:map-reduce:parallel-fold-map]
are actually equivalent to a single call to `foldMap`,
shortening the entire algorithm as follows:

```scala mdoc:reset:invisible
import cats._
import cats.implicits._
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.foldLeft(Monoid[B].empty)(_ |+| func(_))
```
```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  val groups: Iterator[Vector[A]] =
    values.grouped(groupSize)

  val futures: Iterator[Future[B]] =
    groups.map(group => Future(foldMap(group)(func)))

  Future.sequence(futures) map { iterable =>
    iterable.foldLeft(Monoid[B].empty)(_ |+| _)
  }
}

val result: Future[Int] =
  parallelFoldMap((1 to 1000000).toVector)(identity)
```

```scala mdoc
Await.result(result, 1.second)
```
</div>

### *parallelFoldMap* with more Cats

Although we implemented `foldMap` ourselves above,
the method is also available as part of the `Foldable`
type class we discussed in Section [@sec:foldable].

Reimplement `parallelFoldMap` using Cats'
`Foldable` and `Traverseable` type classes.

<div class="solution">
We'll restate all of the necessary imports for completeness:

```scala mdoc:silent:reset
import cats.Monoid

import cats.instances.int._    // for Monoid
import cats.instances.future._ // for Applicative and Monad
import cats.instances.vector._ // for Foldable and Traverse

import cats.syntax.foldable._  // for combineAll and foldMap
import cats.syntax.traverse._  // for traverse

import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```

Here's the implementation of `parallelFoldMap`
delegating as much of the method body to Cats as possible:

```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  values
    .grouped(groupSize)
    .toVector
    .traverse(group => Future(group.toVector.foldMap(func)))
    .map(_.combineAll)
}
```

```scala mdoc:silent
val future: Future[Int] =
  parallelFoldMap((1 to 1000).toVector)(_ * 1000)
```

```scala mdoc
Await.result(future, 1.second)
```

The call to `vector.grouped` returns an `Iterable[Iterator[Int]]`.
We sprinkle calls to `toVector` through the code
to convert the data back to a form that Cats can understand.
The call to `traverse` creates a `Future[Vector[Int]]`
containing one `Int` per batch.
The call to `map` then combines the `match` using
the `combineAll` method from `Foldable`.
</div>

## Summary

In this case study we implemented
a system that imitates map-reduce
as performed on a cluster.
Our algorithm followed three steps:

1. batch the data and send one batch to each "node";
2. perform a local map-reduce on each batch;
3. combine the results using monoid addition.

Our toy system emulates the batching behaviour
of real-world map-reduce systems such as Hadoop.
However, in reality we are running all of our work
on a single machine where communcation between nodes is negligible.
We don't actually need to batch data
to gain efficient parallel processing of a list.
We can simply map using a `Functor` and reduce using a `Monoid`.

Regardless of the batching strategy,
mapping and reducing with `Monoids`
is a powerful and general framework
that isn't limited to simple tasks
like addition and string concatenation.
Most of the tasks data scientists perform
in their day-to-day analyses can be cast as monoids.
There are monoids for all the following:

- approximate sets such as the Bloom filter;
- set cardinality estimators,
  such as the HyperLogLog algorithm;
- vectors and vector operations
  like stochastic gradient descent;
- quantile estimators such as the t-digest

to name but a few.


```scala mdoc:reset:silent
```
--->

# ケーススタディ: MapReduce {#sec:map-reduce}

このケーススタディでは、`Monoid` や `Functor` およびその他さまざまな便利な機能を用いて、シンプルながら強力な並列処理フレームワークを実装する。

もし Hadoop を使ったことがある、もしくはビッグデータを取り扱う仕事をしたことがあるなら、[MapReduce][link-map-reduce] を聞いたことがあるだろう。MapReduce は、複数マシン（いわゆるノード）のクラスタ上で並列データ処理を行うためのプログラミングモデルである。その名前が示すように、このモデルは *map* フェーズと *reduce* フェーズを中心に構築されている[^hadoop-shuffle]。ここでいう *map* とは Scala や `Functor` 型クラスでおなじみの `map` 関数と同じものであり、*reduce* フェーズは、Scala で通常 `fold` と呼ばれる操作を指す。

[^hadoop-shuffle]: Hadoop には shuffle フェーズも存在するが、ここでは無視する。

## 並列化された *map* と *fold*

すでに学んだとおり、`map` の一般的なシグネチャは `A => B` 型の関数を `F[A]` に適用して `F[B]` を返すというものである。

![Type chart: functor map](src/pages/functors/generic-map.pdf+svg){#fig:map-reduce:functor-type-chart}

`map` はシーケンス内の各要素を個別に変換する。異なる要素に適用される変換処理の間には依存関係がないので、この処理は容易に並列化できる。型で表現されない副作用を用いないと仮定すれば、関数 `A => B` の型シグネチャがこの性質、すなわち変換処理の独立性を示している。

`fold` はどうだろうか。このステップは `Foldable` インスタンスを用いて実装できる。すべてのファンクターが `Foldable` インスタンスをもつわけではないが、これら両方の型クラスを備えたデータ型を基礎として MapReduce システムは構築される。Reduce ステップでは、分散処理された `map` の結果に対して `foldLeft` を適用することになる。

![Type chart: fold](src/pages/foldable-traverse/generic-foldleft.pdf+svg){#fig:map-reduce:foldleft-type-chart}

Reduce ステップを分散処理する場合、計算順序についてのコントロールは失われる。Reduce 処理全体としては必ずしも完全に左から右という順序にはならず、複数の部分シーケンスを左から右に Reduce してからその結果を結合する流れとなる。結果の正しさを保証するには、Reduce 処理は*結合的*でなくてはならない。

```scala
reduce(a1, reduce(a2, a3)) == reduce(reduce(a1, a2), a3)
```

結合律が満たされていれば、Reduce 処理を複数ノードに自由に分散することができる。ただし、各ノード内の部分シーケンスは元のデータセットにおける順序を保持する必要がある。

`fold` は計算の初期値として `B` 型の要素を必要とする。`fold` が任意の数の並列ステップに分割される可能性を考えると、初期値は計算結果に影響を与えてならない。このことは、初期値が*単位元*であることを要請する。

```scala
reduce(seed, a1) == reduce(a1, seed) == a1
```

まとめると、並列化された `fold` が正しい結果を得るには、以下の条件を満たす必要がある。

- Reduce 関数が結合的であること
- その関数における単位元で計算を初期化すること

このパターン、どこかで聞き覚えがないだろうか。そのとおり、話は再び `Monoid` に戻る。これは本書で最初に紹介した型クラスである。モノイドの重要性はすでに多くの人に認識されており、[MapReduce ジョブにおけるモノイド設計パターン][link-map-reduce-monoid]は、Twitter の [Summingbird][link-summingbird] のような最近のビッグデータシステムの中核となっている。

このプロジェクトでは、非常にシンプルな単一マシン上の MapReduce を実装する。データフローをモデル化するために、まず `foldMap` というメソッドを実装するところから始めよう。

## *foldMap* の実装

`foldMap` については、以前 `Foldable` を扱った際に簡単に見た。これは `foldLeft` と `foldRight` の上に構築される派生操作のひとつである。ただし、今回は `Foldable` を使わず `foldMap` を自分たちで再実装する。そうすることで MapReduce の構造に関する有用な洞察を得ることができる。

まずは `foldMap` のシグネチャを記述せよ。なお、この関数は以下のパラメータを受け取る必要がある。

 - `Vector[A]` 型のシーケンス
 - `A => B` 型の関数。ただし `B` に対して `Monoid` インスタンスが存在すること

この型シグネチャを完成させるには、暗黙パラメータもしくはコンテキスト境界を追加する必要がある。

<div class="solution">

```scala mdoc:silent
import cats.Monoid

/** シングルスレッドの MapReduce 関数。
  * `values` を `func` でマップし、`Monoid[B]` によって畳み込む。
  */
def foldMap[A, B: Monoid](values: Vector[A])(func: A => B): B =
  ???
```
</div>

つづいて `foldMap` の本体を実装せよ。必要な手順のガイドとして図[@fig:map-reduce:fold-map]のフローチャートをもちいること。

1. 型 `A` の要素をもつシーケンスを用意する
2. それを `func` でマップし、型 `B` の要素をもつシーケンスを生成する
3. `Monoid` を用い、シーケンスを単一の `B` 型の値へと畳み込む

![*foldMap* algorithm](src/pages/case-studies/map-reduce/fold-map.pdf+svg){#fig:map-reduce:fold-map}

参考までに、以下に出力のサンプルを示す。

```scala mdoc:invisible:reset
import cats.Monoid
import cats.syntax.semigroup._ // |+|

def foldMap[A, B: Monoid](values: Vector[A])(func: A => B): B =
  values.foldLeft(Monoid[B].empty)(_ |+| func(_))
```

```scala mdoc:silent
import cats.instances.int._ // Monoid
```

```scala mdoc
foldMap(Vector(1, 2, 3))(identity)
```

```scala mdoc:silent
import cats.instances.string._ // Monoid
```

```scala mdoc
// String へのマッピングでは連結モノイドが使われる。
foldMap(Vector(1, 2, 3))(_.toString + "! ")

// 各文字を String へマッピングして再び String を作る。
foldMap("Hello world!".toVector)(_.toString.toUpperCase)
```

<div class="solution">

`B` に対する `Monoid` を受け取るために型シグネチャを変更する必要がある。そうすることで、[@sec:monoid-syntax]節で述べた `Monoid.empty` や `|+|` 構文が使えるようになる。

```scala mdoc:reset:silent
import cats.Monoid
import cats.syntax.semigroup._ // |+|

def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.map(func).foldLeft(Monoid[B].empty)(_ |+| _)
```

このコードをすこし書き換えて、すべてを一度に行うこともできる。

```scala mdoc:reset:invisible
import cats.Monoid
import cats.syntax.semigroup._
```
```scala mdoc:silent
def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.foldLeft(Monoid[B].empty)(_ |+| func(_))
```
</div>

## *foldMap* の並列化

シングルスレッドの `foldMap` 実装が動作するようになったので、次はこれを並列実行するための分散処理に目を向けてみよう。ここでは、先ほどのシングルスレッド版 `foldMap` を構成要素として利用する。

これから実装するのは、複数の CPU を使って並列処理を行うバージョンである。これは、図[@fig:map-reduce:parallel-fold-map]に示されている MapReduce クラスタにおける処理分散の仕組みを模している。

1. 処理対象となるすべてのデータを最初に用意する
2. データをいくつかのバッチに分割し、それぞれを各 CPU に送る
3. Map フェーズを各 CPU がバッチ単位で並列に実行する
4. Reduce フェーズを各 CPU がバッチ単位で並列に実行し、バッチごとの結果を得る
5. 各バッチの結果をひとつに畳み込んで最終結果を得る

![*parallelFoldMap* algorithm](src/pages/case-studies/map-reduce/parallel-fold-map.pdf+svg){#fig:map-reduce:parallel-fold-map}

Scala には、スレッド間で仕事を分配するための簡単なツールが用意されている。たとえば [parallel collections library][link-parallel-collections] を使ってこの課題を実装することも可能である。しかし今回はもうすこし深く掘り下げ、`Future` を用いて自前でアルゴリズムを実装することに挑戦してみたい。

### *Future* とスレッドプールと ExecutionContext

`Future` がモナド的であることについてはすでにかなり理解が進んでいる。それを簡単におさらいし、Scala の `Future` が裏側でどのようにスケジューリングされているのかを確認しよう。

`Future` はスレッドプール上で実行される。そのプールは暗黙の `ExecutionContext` パラメータによって決定される。`Future.apply` やその他のコンビネータによって `Future` を作成するときは必ず、スコープ内に暗黙の `ExecutionContext` が存在していなければならない。

```scala mdoc:silent
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
val future1 = Future {
  (1 to 100).toList.foldLeft(0)(_ + _)
}

val future2 = Future {
  (100 to 200).toList.foldLeft(0)(_ + _)
}
```

この例では `ExecutionContext.Implicits.global` をインポートしている。このデフォルトのコンテキストでは、マシン上の CPU ごとにひとつスレッドを割り当てたスレッドプールが使用される。`Future` を作成すると、`ExecutionContext` がその実行をスケジューリングする。プール内に空いているスレッドがあれば、`Future` は即座に実行を開始する。現代の多くのマシンはすくなくともふたつの CPU を備えているため、上記の例における `future1` と `future2` は並列に実行される可能性が高い。

他の `Future` の結果に基づいて処理をスケジューリングする新たな `Future` を作成するようなコンビネータもある。たとえば `map` や `flatMap` は、入力値が計算され、かつ利用可能な CPU があれば、即座に計算を実行する。

```scala mdoc
val future3 = future1.map(_.toString)

val future4 = for {
  a <- future1
  b <- future2
} yield a + b
```

[@sec:traverse]節で見たように、`Future.sequence` を使えば `List[Future[A]]` を `Future[List[A]]` に変換できる。

```scala mdoc
Future.sequence(List(Future(1), Future(2), Future(3)))
```

あるいは `Traverse` インスタンスを使ってもよい。

```scala mdoc:silent
import cats.instances.future._ // Applicative
import cats.instances.list._   // Traverse
import cats.syntax.traverse._  // sequence
```

```scala mdoc
List(Future(1), Future(2), Future(3)).sequence
```

いずれの場合も `ExecutionContext` インスタンスが必要である。最後に、`Await.result` を使えば `Future` の結果を得られるまで待機することができる。

```scala mdoc:silent
import scala.concurrent._
import scala.concurrent.duration._
```

```scala mdoc
Await.result(Future(1), 1.second) // 結果を待つ
```

また、`cats.instances.future` では `Future` に対する `Monad` および `Monoid` 実装も提供されている。

```scala mdoc:silent
import cats.{Monad, Monoid}
import cats.instances.int._    // Monoid[Int]
import cats.instances.future._ // Future 用の Monad と Monoid

Monad[Future].pure(42)

Monoid[Future[Int]].combine(Future(1), Future(2))
```

### Dividing Work

`Future` の復習を終えたところで、次は作業をどのようにバッチに分割すればよいか見ていこう。マシン上の利用可能な CPU 数は、Java の標準ライブラリが提供する API を使って調べることができる。

```scala mdoc
Runtime.getRuntime.availableProcessors
```

`grouped` メソッドを使えばシーケンス（実際には `Vector` を実装している任意のコレクション）を分割できる。このメソッドを用いて、各 CPU に割り当てるバッチを切り出す。


```scala mdoc
(1 to 10).toList.grouped(3).toList
```

### *parallelFoldMap* の実装

`foldMap` の並列版、`parallelFoldMap` を実装せよ。以下をその型シグネチャとする。

```scala mdoc:silent
def parallelFoldMap[A, B : Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = ???
```

前述のテクニックを用いて作業を複数のバッチに分割する。バッチは CPU ごとにひとつ、各バッチを並列スレッドで処理する。全体のアルゴリズムを確認したければ図[@fig:map-reduce:parallel-fold-map]を改めて参照するとよい。

可能であれば、各 CPU のバッチ処理には上で定義した `foldMap` 実装を再利用してほしい。

<div class="solution">
コードをフェーズごとに分けてコメントをつけた解答例を以下に示す。

```scala mdoc:invisible:reset
import cats._
import cats.implicits._
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```
```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  // 各 CPU に渡すデータ件数を算出
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  // データを CPU ごとのグループに分割
  val groups: Iterator[Vector[A]] =
    values.grouped(groupSize)

  // 各グループを foldMap するための Future インスタンスを作成
  val futures: Iterator[Future[B]] =
    groups map { group =>
      Future {
        group.foldLeft(Monoid[B].empty)(_ |+| func(_))
      }
    }

  // グループごとの結果を最終結果へと畳み込む
  Future.sequence(futures) map { iterable =>
    iterable.foldLeft(Monoid[B].empty)(_ |+| _)
  }
}

val result: Future[Int] =
  parallelFoldMap((1 to 1000000).toVector)(identity)
```

```scala mdoc
Await.result(result, 1.second)
```

上で定義した `foldMap` を再利用すれば、もっと簡潔な解答が得られる。図[@fig:map-reduce:parallel-fold-map]のステップ3と4にあたるバッチ単位でのマップと畳み込みは、単一の `foldMap` 呼び出しによって実現できる。これにより、全体のアルゴリズムは以下のとおり簡略化される。

```scala mdoc:reset:invisible
import cats._
import cats.implicits._
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
def foldMap[A, B : Monoid](as: Vector[A])(func: A => B): B =
  as.foldLeft(Monoid[B].empty)(_ |+| func(_))
```
```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  val groups: Iterator[Vector[A]] =
    values.grouped(groupSize)

  val futures: Iterator[Future[B]] =
    groups.map(group => Future(foldMap(group)(func)))

  Future.sequence(futures) map { iterable =>
    iterable.foldLeft(Monoid[B].empty)(_ |+| _)
  }
}

val result: Future[Int] =
  parallelFoldMap((1 to 1000000).toVector)(identity)
```

```scala mdoc
Await.result(result, 1.second)
```
</div>

### Cats をもっと活用した *parallelFoldMap* 実装

先ほどは `foldMap` を自前で実装したが、このメソッドは[@sec:foldable]節で紹介した `Foldable` 型クラスにも含まれており、利用することができる。

Cats の `Foldable` と `Traverseable` 型クラスを用いて `parallelFoldMap` を再実装せよ。

<div class="solution">

完全を期すために、必要なすべてのインポート文を改めて記載する。

```scala mdoc:silent:reset
import cats.Monoid

import cats.instances.int._    // Monoid
import cats.instances.future._ // Applicative と Monad
import cats.instances.vector._ // Foldable と Traverse

import cats.syntax.foldable._  // combineAll と foldMap
import cats.syntax.traverse._  // traverse

import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```

メソッド本体にできるかぎり Cats を利用した `parallelFoldMap` 実装を以下に示す。

```scala mdoc:silent
def parallelFoldMap[A, B: Monoid]
      (values: Vector[A])
      (func: A => B): Future[B] = {
  val numCores  = Runtime.getRuntime.availableProcessors
  val groupSize = (1.0 * values.size / numCores).ceil.toInt

  values
    .grouped(groupSize)
    .toVector
    .traverse(group => Future(group.toVector.foldMap(func)))
    .map(_.combineAll)
}
```

```scala mdoc:silent
val future: Future[Int] =
  parallelFoldMap((1 to 1000).toVector)(_ * 1000)
```

```scala mdoc
Await.result(future, 1.second)
```

`vector.grouped` 呼び出しは `Iterable[Iterator[Int]]` を返す。このデータを Cats が理解できる形式に戻すため、コードのいくつかの場所で `toVector` を呼び出している。`traverse` 呼び出しは、バッチごとにひとつの `Int` 値をもった `Future[Vector[Int]]` インスタンスを生成する。そして `map` 呼び出しは `Foldable` の `combineAll` メソッドを用いて各バッチの結果をひとつにまとめている。

</div>

## まとめ

このケーススタディでは、本来クラスタ上で行われる MapReduce を模倣したシステムを実装した。今回のアルゴリズムは次の三ステップで構成されていた。

1. データをバッチに分割し、各バッチをそれぞれの「ノード」に送る
2. バッチごとに MapReduce を実行する
3. モノイドによる加法を用いて結果を統合する

このおもちゃのようなシステムは、Hadoop のような実際の MapReduce システムにおける分散処理の挙動を模倣している。とはいえ現実にはすべての処理は単一マシン上で行われており、ノード間通信のコストは無視できる。厳密に言えば、リストの並列処理を効率的に行うためにデータをバッチに分割する必要はない。単に `Functor` でマッピングし、`Monoid` で畳み込めばよい。

バッチ分割の戦略を採用するかどうかにかかわらず、マッピングおよび `Monoid` を用いた畳み込みは、強力かつ汎用的な枠組みである。それは加算や文字列連結といった単純な処理にとどまらない。データサイエンティストが日々の分析で行っている多くのタスクは、モノイドとして扱える。以下に挙げたものいずれにもモノイドは存在する。

- Bloom フィルタのような近似集合
- HyperLogLog アルゴリズムのような集合のカーディナリティ推定器
- 確率的勾配降下法などに用いるベクトルとその演算
- t-Digest のような分位数推定器

これらはほんの一例にすぎない。
