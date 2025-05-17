<!--

## The Eval Monad {#sec:monads:eval}

[`cats.Eval`][cats.Eval] is a monad that allows us to
abstract over different *models of evaluation*.
We typically talk of two such models: *eager* and *lazy*,
also called *call-by-value* and *call-by-name* respectively.
`Eval` also allows for a result to be *memoized*,
which gives us *call-by-need* evaluation.

`Eval` is also *stack-safe*,
which means we can use it in very deep recursions
without blowing up the stack.


### Eager, Lazy, Memoized, Oh My!

What do these terms for models of evaluation mean?
Let's see some examples.

Let's first look at Scala `vals`.
We can see the evaluation model using
a computation with a visible side-effect.
In the following example,
the code to compute the value of `x`
happens at place where it is defined
rather than on access.
Accessing `x` recalls the stored value
without re-running the code.

```scala mdoc
val x = {
  println("Computing X")
  math.random()
}

x // first access
x // second access
```

This is an example of call-by-value evaluation:

- the computation is evaluated at point where it is defined (eager); and
- the computation is evaluated once (memoized).


Let's look at an example using a `def`.
The code to compute `y` below
is not run until we use it,
and is re-run on every access:

```scala mdoc
def y = {
  println("Computing Y")
  math.random()
}

y // first access
y // second access
```

These are the properties of call-by-name evaluation:

- the computation is evaluated at the point of use (lazy); and
- the computation is evaluated each time it is used (not memoized).

Last but not least,
`lazy vals` are an example of call-by-need evaluation.
The code to compute `z` below
is not run until we use it
for the first time (lazy).
The result is then cached
and re-used on subsequent accesses (memoized):

```scala mdoc
lazy val z = {
  println("Computing Z")
  math.random()
}

z // first access
z // second access
```

Let's summarize. There are two properties of interest:

- evaluation at the point of definition (eager) versus at the point of use (lazy); and
- values are saved once evaluated (memoized) or not (not memoized).

There are three possible combinations of these properties:

- call-by-value which is eager and memoized;
- call-by-name which is lazy and not memoized; and
- call-by-need which is lazy and memoized.

The final combination, eager and not memoized, is not possible.


### Eval's Models of Evaluation

`Eval` has three subtypes: `Now`, `Always`, and `Later`.
They correspond to call-by-value, call-by-name, and call-by-need respectively.
We construct these with three constructor methods,
which create instances of the three classes
and return them typed as `Eval`:

```scala mdoc:silent
import cats.Eval
```

```scala mdoc
val now = Eval.now(math.random() + 1000)
val always = Eval.always(math.random() + 3000)
val later = Eval.later(math.random() + 2000)
```

We can extract the result of an `Eval`
using its `value` method:

```scala mdoc
now.value
always.value
later.value
```

Each type of `Eval` calculates its result
using one of the evaluation models defined above.
`Eval.now` captures a value *right now*.
Its semantics are similar to a `val`---eager and memoized:

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc
val x = Eval.now{
  println("Computing X")
  math.random()
}

x.value // first access
x.value // second access
```

`Eval.always` captures a lazy computation,
similar to a `def`:

```scala mdoc
val y = Eval.always{
  println("Computing Y")
  math.random()
}

y.value // first access
y.value // second access
```

Finally, `Eval.later` captures a lazy, memoized computation,
similar to a `lazy val`:

```scala mdoc
val z = Eval.later{
  println("Computing Z")
  math.random()
}

z.value // first access
z.value // second access
```


The three behaviours are summarized below:

-----------------------------------------------------------------------
Scala              Cats                      Properties
------------------ ------------------------- --------------------------
`val`              `Now`                     eager, memoized

`def`              `Always`                  lazy, not memoized

`lazy val`         `Later`                   lazy, memoized
------------------ ------------------------- --------------------------

### Eval as a Monad

Like all monads, `Eval's` `map` and `flatMap` methods add computations to a chain.
In this case, however, the chain is stored explicitly as a list of functions.
The functions aren't run until we call
`Eval's` `value` method to request a result:

```scala mdoc
val greeting = Eval
  .always{ println("Step 1"); "Hello" }
  .map{ str => println("Step 2"); s"$str world" }

greeting.value
```

Note that, while the semantics of
the originating `Eval` instances are maintained,
mapping functions are always
called lazily on demand (`def` semantics):

```scala mdoc
val ans = for {
  a <- Eval.now{ println("Calculating A"); 40 }
  b <- Eval.always{ println("Calculating B"); 2 }
} yield {
  println("Adding A and B")
  a + b
}

ans.value // first access
ans.value // second access
```

`Eval` has a `memoize` method that
allows us to memoize a chain of computations.
The result of the chain up to the call to `memoize` is cached,
whereas calculations after the call retain their original semantics:

```scala mdoc
val saying = Eval
  .always{ println("Step 1"); "The cat" }
  .map{ str => println("Step 2"); s"$str sat on" }
  .memoize
  .map{ str => println("Step 3"); s"$str the mat" }

saying.value // first access
saying.value // second access
```

### Trampolining and *Eval.defer*

One useful property of `Eval`
is that its `map` and `flatMap` methods are *trampolined*.
This means we can nest calls to `map` and `flatMap` arbitrarily
without consuming stack frames.
We call this property *"stack safety"*.

For example, consider this function for calculating factorials:

```scala mdoc:silent
def factorial(n: BigInt): BigInt =
  if(n == 1) n else n * factorial(n - 1)
```

It is relatively easy to make this method stack overflow:

```scala
factorial(50000)
// java.lang.StackOverflowError
//   ...
```

We can rewrite the method using `Eval` to make it stack safe:

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc:silent
def factorial(n: BigInt): Eval[BigInt] =
  if(n == 1) {
    Eval.now(n)
  } else {
    factorial(n - 1).map(_ * n)
  }
```

```scala
factorial(50000).value
// java.lang.StackOverflowError
//   ...
```

Oops! That didn't work---our stack still blew up!
This is because we're still making all the recursive calls to `factorial`
before we start working with `Eval's` `map` method.
We can work around this using `Eval.defer`,
which takes an existing instance of `Eval` and defers its evaluation.
The `defer` method is trampolined like `map` and `flatMap`,
so we can use it as a quick way to make an existing operation stack safe:

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc:silent
def factorial(n: BigInt): Eval[BigInt] =
  if(n == 1) {
    Eval.now(n)
  } else {
    Eval.defer(factorial(n - 1).map(_ * n))
  }
```

```scala
factorial(50000).value
// res: A very big value
```

`Eval` is a useful tool to enforce stack safety
when working on very large computations and data structures.
However, we must bear in mind that trampolining is not free.
It avoids consuming stack by creating a chain of function objects on the heap.
There are still limits on how deeply we can nest computations,
but they are bounded by the size of the heap rather than the stack.

### Exercise: Safer Folding using Eval

The naive implementation of `foldRight` below is not stack safe.
Make it so using `Eval`:

```scala mdoc:silent
def foldRight[A, B](as: List[A], acc: B)(fn: (A, B) => B): B =
  as match {
    case head :: tail =>
      fn(head, foldRight(tail, acc)(fn))
    case Nil =>
      acc
  }
```

<div class="solution">
The easiest way to fix this is
to introduce a helper method called `foldRightEval`.
This is essentially our original method
with every occurrence of `B` replaced with `Eval[B]`,
and a call to `Eval.defer` to protect the recursive call:

```scala mdoc:silent:reset-object
import cats.Eval

def foldRightEval[A, B](as: List[A], acc: Eval[B])
    (fn: (A, Eval[B]) => Eval[B]): Eval[B] =
  as match {
    case head :: tail =>
      Eval.defer(fn(head, foldRightEval(tail, acc)(fn)))
    case Nil =>
      acc
  }
```

We can redefine `foldRight` simply in terms of `foldRightEval`
and the resulting method is stack safe:

```scala mdoc:silent
def foldRight[A, B](as: List[A], acc: B)(fn: (A, B) => B): B =
  foldRightEval(as, Eval.now(acc)) { (a, b) =>
    b.map(fn(a, _))
  }.value
```

```scala mdoc
foldRight((1 to 100000).toList, 0L)(_ + _)
```
</div>


```scala mdoc:reset:silent
```
--->

## `Eval` モナド {#sec:monads:eval}

[`cats.Eval`][cats.Eval] は、異なる*評価モデル*の抽象化を可能にするモナドである。一般的に、*先行評価（eager）*と*遅延評価（lazy）*というふたつのモデルが議論される。それぞれ*値呼び（call-by-value）*および*名前呼び（call-by-name）*とも呼ばれる。さらに、`Eval` は結果を*メモ化*することも可能で、これにより*必要呼び（call-by-need）*と呼ばれる評価方式が提供される。

`Eval` はスタックセーフでもある。非常に深い再帰処理であってもスタックオーバーフローを起こすことなく用いることができる。

### 先行評価、遅延評価、メモ化

評価モデルに関するこれらの用語は何を意味しているのだろうか。いくつか例を見てみよう。

まずは Scala の `val` について見てみよう。目に見える副作用をともなう計算を用いることで評価モデルを確認できる。次の例では、`x` の値を計算するコードは、アクセス時ではなく `x` が定義された時点で実行される。`x` にアクセスすると、コードを再実行することなく、保存された値が取得される。

```scala mdoc
val x = {
  println("Computing X")
  math.random()
}

x // 最初のアクセス
x // 二回目のアクセス
```

これは値呼び評価の例である。

- 計算は定義された時点で評価される（先行評価）
- 計算は一度だけ評価される（メモ化される）

続いて `def` を使った例を見てみよう。`y` を計算する次のコードは、`y` を使うまで実行されず、そして使うたびに再実行される。

```scala mdoc
def y = {
  println("Computing Y")
  math.random()
}

y // 最初のアクセス
y // 二度目のアクセス
```

名前呼び評価は以下のような性質をもっている。

- 計算は利用された時点で評価される（遅延評価）
- 計算は利用されるたびに評価される（メモ化されない）

最後になるが重要なこととして、`lazy val` は必要呼び評価の例である。`z` を計算する次のコードは、最初に使用されるまで実行されない（遅延評価）。その結果はキャッシュされ、次回以降のアクセス時には再利用される（メモ化される）。

```scala mdoc
lazy val z = {
  println("Computing Z")
  math.random()
}

z // 最初のアクセス
z // 二度目のアクセス
```

まとめよう。重要な特性の軸はふたつある。

- 定義時に評価される（先行評価）か使用時に評価される（遅延評価）か
- 一度評価された値が保存される（メモ化される）か、されないか

これらの特性には三つの組み合わせがあり得る。

- 値呼びは、先行評価でメモ化あり
- 名前呼びは、遅延評価でメモ化なし
- 必要呼びは、遅延評価でメモ化あり

最後に残された「先行評価でメモ化なし」という組み合わせはあり得ない。

### `Eval` の評価モデル

`Eval` には `Now`、`Always`、および `Later` という三つの部分型が存在し、それぞれ値呼び、名前呼び、必要呼びに対応している。これら三つのクラスのインスタンス生成用に三つのコンストラクタメソッドが用意されている。作成されたインスタンスは `Eval` 型として返される。

```scala mdoc:silent
import cats.Eval
```

```scala mdoc
val now = Eval.now(math.random() + 1000)
val always = Eval.always(math.random() + 3000)
val later = Eval.later(math.random() + 2000)
```

`Eval` インスタンスがもつ結果は `value` メソッドを使って展開できる。

```scala mdoc
now.value
always.value
later.value
```

`Eval` のそれぞれの型は、前述の評価モデルのいずれかを使用して結果を計算する。`Eval.now` は、値を*今すぐ*取得するもので、そのセマンティクスは `val` に似ている。つまり、先行評価であり、メモ化される。

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc
val x = Eval.now{
  println("Computing X")
  math.random()
}

x.value // 最初のアクセス
x.value // 二度目のアクセス
```

`Eval.always` は遅延評価される計算を表す。`def` のようなものである。

```scala mdoc
val y = Eval.always{
  println("Computing Y")
  math.random()
}

y.value // 最初のアクセス
y.value // 二度目のアクセス
```

最後に、`Eval.later` は遅延評価されメモ化される計算を表す。`lazy val` のようなものである。

```scala mdoc
val z = Eval.later{
  println("Computing Z")
  math.random()
}

z.value // 最初のアクセス
z.value // 二度目のアクセス
```

これら三つの振る舞いを下表にまとめる。

-----------------------------------------------------------------------
Scala              Cats                      特性
------------------ ------------------------- --------------------------
`val`              `Now`                     先行評価・メモ化あり
`def`              `Always`                  遅延評価・メモ化なし
`lazy val`         `Later`                   遅延評価・メモ化あり
------------------ ------------------------- --------------------------

###　モナドとしての `Eval`

すべてのモナドと同様に、`Eval` の `map` および `flatMap` メソッドは計算をチェーンに追加する。しかし、この場合、チェーンは関数のリストとして明示的に保存される。`Eval` の `value` メソッドを呼び出して結果を要求するまで、それらの関数は実行されない。

```scala mdoc
val greeting = Eval
  .always{ println("Step 1"); "Hello" }
  .map{ str => println("Step 2"); s"$str world" }

greeting.value
```

元の `Eval` インスタンスのセマンティクスは保たれるものの、マッピング関数は常に必要に応じて遅延評価される（`def` のセマンティクスをもつ）という点に注意してほしい。

```scala mdoc
val ans = for {
  a <- Eval.now{ println("Calculating A"); 40 }
  b <- Eval.always{ println("Calculating B"); 2 }
} yield {
  println("Adding A and B")
  a + b
}

ans.value // 最初のアクセス
ans.value // 二度目のアクセス
```

`Eval` は、計算のチェーンをメモ化できる `memoize` メソッドをもっている。`memoize` 呼び出しまでのチェーンの結果はキャッシュされるが、その後の計算は元のセマンティクスを保持する。

```scala mdoc
val saying = Eval
  .always{ println("Step 1"); "The cat" }
  .map{ str => println("Step 2"); s"$str sat on" }
  .memoize
  .map{ str => println("Step 3"); s"$str the mat" }

saying.value // 最初のアクセス
saying.value // 二度目のアクセス
```

### トランポリン化と `Eval.defer`

`Eval` の便利な特性のひとつは `map` および `flatMap` メソッドが*トランポリン化*されることである。これは、スタックフレームを消費することなく、`map` や `flatMap` の呼び出しを任意の深さにネストできることを意味する。この特性は*スタックセーフ（stack safety）*と呼ばれる。

たとえば、階乗の計算を行う次のような関数を考えてみよう。

```scala mdoc:silent
def factorial(n: BigInt): BigInt =
  if(n == 1) n else n * factorial(n - 1)
```

このメソッドは比較的簡単にスタックオーバーフローを引き起こす。

```scala
factorial(50000)
// java.lang.StackOverflowError
//   ...
```

`Eval` を使えば、このメソッドをスタックセーフに書き換えることができる。

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc:silent
def factorial(n: BigInt): Eval[BigInt] =
  if(n == 1) {
    Eval.now(n)
  } else {
    factorial(n - 1).map(_ * n)
  }
```

```scala
factorial(50000).value
// java.lang.StackOverflowError
//   ...
```

……はずだったが、またしてもスタックがあふれてしまった。これは、`Eval` の `map` メソッドを実行する前に、再帰的な `factorial` 呼び出しがすべて行われているためである。`Eval.defer` を使えば、この問題を回避できる。`Eval.defer` は既存の `Eval` インスタンスを引数に取り、その評価を遅延させる。`defer` メソッドも `map` および `flatMap` と同様にトランポリン化されるため、既存の操作をスタックセーフにするための簡単な方法として利用できる。

```scala mdoc:invisible:reset-object
import cats.Eval
```
```scala mdoc:silent
def factorial(n: BigInt): Eval[BigInt] =
  if(n == 1) {
    Eval.now(n)
  } else {
    Eval.defer(factorial(n - 1).map(_ * n))
  }
```

```scala
factorial(50000).value
// res: A very big value
```

`Eval` は、大規模な計算やデータ構造を扱う際にスタックセーフ性を確保するための有用なツールである。しかし、トランポリン化も無料ではないことを心に留めておかなくてはならない。スタックの消費を避ける代わりに、ヒープ上に関数オブジェクトのチェーンを作成する。そのため、計算のネストの深さには依然として限界があり、その限界はスタックサイズではなくヒープサイズに依存する。

### 演習: `Eval` を使ったより安全な畳み込み

以下の素朴な `foldRight` 実装はスタックセーフではない。`Eval` を使ってこれをスタックセーフにせよ。

```scala mdoc:silent
def foldRight[A, B](as: List[A], acc: B)(fn: (A, B) => B): B =
  as match {
    case head :: tail =>
      fn(head, foldRight(tail, acc)(fn))
    case Nil =>
      acc
  }
```

<div class="solution">
この問題を解決するもっとも簡単な方法は、`foldRightEval` という補助メソッドを導入することである。これは基本的には元のメソッドと同じだが、すべての `B` を `Eval[B]` に置き換え、再帰呼び出しをスタックオーバーフローから守るために `Eval.defer` を用いる。

```scala mdoc:silent:reset-object
import cats.Eval

def foldRightEval[A, B](as: List[A], acc: Eval[B])
    (fn: (A, Eval[B]) => Eval[B]): Eval[B] =
  as match {
    case head :: tail =>
      Eval.defer(fn(head, foldRightEval(tail, acc)(fn)))
    case Nil =>
      acc
  }
```

あとは `foldRightEval` を使って `foldRight` を再定義すればスタックセーフなメソッドができあがる。

```scala mdoc:silent
def foldRight[A, B](as: List[A], acc: B)(fn: (A, B) => B): B =
  foldRightEval(as, Eval.now(acc)) { (a, b) =>
    b.map(fn(a, _))
  }.value
```

```scala mdoc
foldRight((1 to 100000).toList, 0L)(_ + _)
```
</div>
