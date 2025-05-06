<!--

# Monads {#sec:monads}

*Monads* are one of the most common abstractions in Scala.
Many Scala programmers quickly become intuitively familiar with monads,
even if we don't know them by name.

Informally, a monad is anything with a constructor and a `flatMap` method.
All of the functors we saw in the last chapter are also monads,
including `Option`, `List`, and `Future`.
We even have special syntax to support monads: for comprehensions.
However, despite the ubiquity of the concept,
the Scala standard library lacks
a concrete type to encompass "things that can be `flatMapped`".

In this chapter we will take a deep dive into monads.
We will start by motivating them with a few examples.
We'll proceed to their formal definition,
and see how we can create a concrete type as a type class.
We'll then look at their implementation in Cats.
Finally, we'll tour some interesting monads that you may not have seen,
providing introductions and examples of their use.


## What is a Monad?

This is the question that has been posed in a thousand blog posts,
with explanations and analogies involving concepts as diverse as
cats, Mexican food, space suits full of toxic waste,
and monoids in the category of endofunctors (whatever that means).
We're going to solve the problem of explaining monads once and for all
by stating very simply:

> A monad is a mechanism for *sequencing computations*.

That was easy! Problem solved, right?
But then again, last chapter we said functors
were a mechanism for exactly the same thing.
Ok, maybe we need some more discussion...

In Section [@sec:functors:examples]
we said that functors allow us
to sequence computations ignoring some complication.
However, functors are limited in that
they only allow this complication
to occur once at the beginning of the sequence.
They don't account for further complications
at each step in the sequence.

This is where monads come in.
A monad's `flatMap` method allows us to specify what happens next,
taking into account an intermediate complication.
The `flatMap` method of `Option` takes intermediate `Options` into account.
The `flatMap` method of `List` handles intermediate `Lists`. And so on.
In each case, the function passed to `flatMap` specifies
the application-specific part of the computation,
and `flatMap` itself takes care of the complication
allowing us to `flatMap` again.
Let's ground things by looking at some examples.


### Options as Monads

`Option` allows us to sequence computations
that may or may not return values.
Here are some examples:

```scala mdoc:silent
def parseInt(str: String): Option[Int] =
  scala.util.Try(str.toInt).toOption

def divide(a: Int, b: Int): Option[Int] =
  if(b == 0) None else Some(a / b)
```

Each of these methods may "fail" by returning `None`.
The `flatMap` method allows us to ignore this
when we sequence operations:

```scala mdoc:silent
def stringDivideBy(aStr: String, bStr: String): Option[Int] =
  parseInt(aStr).flatMap { aNum =>
    parseInt(bStr).flatMap { bNum =>
      divide(aNum, bNum)
    }
  }
```

The semantics are:

- the first call to `parseInt` returns a `None` or a `Some`;
- if it returns a `Some`, the `flatMap` method calls our function and passes us the integer `aNum`;
- the second call to `parseInt` returns a `None` or a `Some`;
- if it returns a `Some`, the `flatMap` method calls our function and passes us `bNum`;
- the call to `divide` returns a `None` or a `Some`, which is our result.

At each step, `flatMap` chooses whether to call our function,
and our function generates the next computation in the sequence.
This is shown in Figure [@fig:monads:option-type-chart].

![Type chart: flatMap for Option](src/pages/monads/option-flatmap.pdf+svg){#fig:monads:option-type-chart}

The result of the computation is an `Option`,
allowing us to call `flatMap` again and so the sequence continues.
This results in the fail-fast error handling behaviour that we know and love,
where a `None` at any step results in a `None` overall:

```scala mdoc
stringDivideBy("6", "2")
stringDivideBy("6", "0")
stringDivideBy("6", "foo")
stringDivideBy("bar", "2")
```

Every monad is also a functor (see below for proof),
so we can rely on both `flatMap` and `map`
to sequence computations
that do and don't introduce a new monad.
Plus, if we have both `flatMap` and `map`
we can use for comprehensions
to clarify the sequencing behaviour:

```scala mdoc:invisible:reset-object
def parseInt(str: String): Option[Int] =
  scala.util.Try(str.toInt).toOption

def divide(a: Int, b: Int): Option[Int] =
  if(b == 0) None else Some(a / b)
```
```scala mdoc:silent
def stringDivideBy(aStr: String, bStr: String): Option[Int] =
  for {
    aNum <- parseInt(aStr)
    bNum <- parseInt(bStr)
    ans  <- divide(aNum, bNum)
  } yield ans
```


### Lists as Monads

When we first encounter `flatMap` as budding Scala developers,
we tend to think of it as a pattern for iterating over `Lists`.
This is reinforced by the syntax of for comprehensions,
which look very much like imperative for loops:

```scala mdoc
for {
  x <- (1 to 3).toList
  y <- (4 to 5).toList
} yield (x, y)
```

However, there is another mental model we can apply
that highlights the monadic behaviour of `List`.
If we think of `Lists` as sets of intermediate results,
`flatMap` becomes a construct that calculates
permutations and combinations.

For example, in the for comprehension above
there are three possible values of `x` and two possible values of `y`.
This means there are six possible values of `(x, y)`.
`flatMap` is generating these combinations from our code,
which states the sequence of operations:

- get `x`
- get `y`
- create a tuple `(x, y)`

<!--
The type chart in Figure [@fig:monads:list-type-chart]
illustrates this behaviour[^list-lengths].

![Type chart: flatMap for List](src/pages/monads/list-flatmap.pdf+svg){#fig:monads:list-type-chart}

[^list-lengths]: Although the result of `flatMap` (`List[B]`)
is the same type as the result of the user-supplied function,
the end result is actually a larger list
created from combinations of intermediate `As` and `Bs`.
--

### Futures as Monads

`Future` is a monad that sequences computations
without worrying that they may be asynchronous:

```scala mdoc:silent
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def doSomethingLongRunning: Future[Int] = ???
def doSomethingElseLongRunning: Future[Int] = ???

def doSomethingVeryLongRunning: Future[Int] =
  for {
    result1 <- doSomethingLongRunning
    result2 <- doSomethingElseLongRunning
  } yield result1 + result2
```

Again, we specify the code to run at each step,
and `flatMap` takes care of all the horrifying
underlying complexities of thread pools and schedulers.

If you've made extensive use of `Future`,
you'll know that the code above
is running each operation *in sequence*.
This becomes clearer if we expand out the for comprehension
to show the nested calls to `flatMap`:

```scala mdoc:invisible:reset-object
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
def doSomethingLongRunning: Future[Int] = ???
def doSomethingElseLongRunning: Future[Int] = ???
```
```scala mdoc:silent
def doSomethingVeryLongRunning: Future[Int] =
  doSomethingLongRunning.flatMap { result1 =>
    doSomethingElseLongRunning.map { result2 =>
      result1 + result2
    }
  }
```

Each `Future` in our sequence is created
by a function that receives the result from a previous `Future`.
In other words, each step in our computation can only start
once the previous step is finished.
This is born out by the type chart for `flatMap`
in Figure [@fig:monads:future-type-chart],
which shows the function parameter of type `A => Future[B]`.

![Type chart: flatMap for Future](src/pages/monads/future-flatmap.pdf+svg){#fig:monads:future-type-chart}

We *can* run futures in parallel, of course,
but that is another story and shall be told another time.
Monads are all about sequencing.


### Definition of a Monad

While we have only talked about `flatMap` above,
monadic behaviour is formally captured in two operations:

- `pure`, of type `A => F[A]`;
- `flatMap`[^bind], of type `(F[A], A => F[B]) => F[B]`.

[^bind]: In the programming literature and Haskell,
`pure` is referred to as `point` or `return` and
`flatMap` is referred to as `bind` or `>>=`.
This is purely a difference in terminology.
We'll use the term `flatMap` for compatibility
with Cats and the Scala standard library.

`pure` abstracts over constructors,
providing a way to create a new monadic context from a plain value.
`flatMap` provides the sequencing step we have already discussed,
extracting the value from a context and generating
the next context in the sequence.
Here is a simplified version of the `Monad` type class in Cats:

```scala mdoc:silent

trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]
}
```

<div class="callout callout-warning">
#### Monad Laws {-}

`pure` and `flatMap` must obey a set of laws
that allow us to sequence operations freely
without unintended glitches and side-effects:

*Left identity*: calling `pure`
and transforming the result with `func`
is the same as calling `func`:

```scala
pure(a).flatMap(func) == func(a)
```

*Right identity*: passing `pure` to `flatMap`
is the same as doing nothing:

```scala
m.flatMap(pure) == m
```

*Associativity*: `flatMapping` over two functions `f` and `g`
is the same as `flatMapping` over `f` and then `flatMapping` over `g`:

```scala
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
```
</div>

### Exercise: Getting Func-y

Every monad is also a functor.
We can define `map` in the same way for every monad
using the existing methods, `flatMap` and `pure`:

```scala mdoc:silent:reset-object

trait Monad[F[_]] {
  def pure[A](a: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    ???
}
```

Try defining `map` yourself now.

<div class="solution">
At first glance this seems tricky,
but if we follow the types we'll see there's only one solution.
We are passed a `value` of type `F[A]`.
Given the tools available there's only one thing we can do:
call `flatMap`:

```scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    flatMap(value)(a => ???)
}
```

We need a function of type `A => F[B]` as the second parameter.
We have two function building blocks available:
the `func` parameter of type `A => B`
and the `pure` function of type `A => F[A]`.
Combining these gives us our result:

```scala mdoc:invisible:reset-object
```
```scala mdoc
trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    flatMap(value)(a => pure(func(a)))
}
```
</div>


-->

# モナド {#sec:monads}

*モナド*は Scala でもっとも一般的な抽象概念のひとつである。Scala プログラマは、それと知らずに自然とモナドに慣れ親しむことも多い。

大雑把に言えば、モナドはコンストラクタと `flatMap` メソッドを備えたオブジェクトのことをいう。前章で見たファンクターは、`Option` や `List` や `Future` を含め、すべてモナドである。さらに、モナドをサポートする特別な構文として for 内包表記がある。しかし、概念は広く浸透しているにもかかわらず、Scala 標準ライブラリには「flatMap できるもの」全般を表す具体的な型が存在しない。

この章ではモナドについて深く掘り下げる。まずいくつかの例を用いてモナドの必要性を理解し、その後、形式的な定義へと進み、型クラスとして具体的な型を作成する方法を見ていく。その後 Cats における実装を確認する。最後に、馴染みは薄いかもしれないが興味深いモナドをいくつか紹介し、その使用例を示す。

## モナドとは何か

この疑問は数多くのブログ記事で取り上げられ、猫や、メキシコ料理や、有毒な破棄物でいっぱいの宇宙服や、自己関手の圏におけるモノイド対象（意味はよくわからないが）など、さまざまな概念を使った説明やアナロジーが繰り広げられてきた。ここではシンプルに以下のように述べることで、この問題を一言で解決しよう。

> モナドとは、計算を順に連結するための仕組みのことである。

この簡単な説明で問題は解決したと言いたいところだが、これでは、前章で述べたファンクターの仕組みとまったく同じである。どうやら、もうすこし議論が必要かもしれない。

[@sec:functors:examples]節で、ファンクターはコンテキストがもつ何らかの複雑さを無視して計算を順に連結することができると述べた。だが、ファンクターには制限がある。ファンクターは、一連の計算の最初に一度だけ、そのような複雑性を扱うことができる。各ステップで新たな複雑性が生じる場合には対応できない。

ここで登場するのがモナドである。モナドの `flatMap` メソッドでは、途中の処理で発生した複雑性も考慮して、次に何を行うか指定することができる。`Option` の `flatMap` メソッドは、途中で新たに生まれた `Option` を考慮するし、`List` の `flatMap` メソッドは途中で生まれた `List` を処理する。他のモナドでも同様である。どのケースでも、`flatMap` に渡される関数によって計算におけるアプリケーション固有の処理を指定し、そこで生じる複雑性を `flatMap` 自体が処理することで、続けて `flatMap` することを可能にする。いくつか例を見てイメージを具体化してみよう。

### モナドとしての `Option`

`Option` は、値が返されるかどうかわからない計算を順序付けて連結することができる。以下にいくつかの例を示す。

```scala mdoc:silent
def parseInt(str: String): Option[Int] =
  scala.util.Try(str.toInt).toOption

def divide(a: Int, b: Int): Option[Int] =
  if(b == 0) None else Some(a / b)
```

Each of these methods may "fail" by returning `None`.
The `flatMap` method allows us to ignore this
when we sequence operations:

これらのメソッドはいずれも失敗する可能性がある。失敗したときは `None` が返却される。`flatMap` では、操作を連結するにあたりこの失敗を無視できる。

```scala mdoc:silent
def stringDivideBy(aStr: String, bStr: String): Option[Int] =
  parseInt(aStr).flatMap { aNum =>
    parseInt(bStr).flatMap { bNum =>
      divide(aNum, bNum)
    }
  }
```

このコードは以下のように翻訳できる。

- 最初の `parseInt` 呼び出しが `None` または `Some` を返す
- `Some` が返された場合、`flatMap` メソッドは指定された関数を呼び出し、整数 `aNum` を渡す
- 次の `parseInt` 呼び出しも `None` または `Some` を返す
- `Some` が返された場合、`flatMap` メソッドは指定された関数を呼び出し、`bNum` を渡す
- `divide` 呼び出しが `None` または `Some` を返し、それが最終結果となる

各ステップにおいて `flatMap` は指定された関数を呼び出すかどうか判定を行い、連結された次の計算で利用する値を生成する。これを図[@fig:monads:option-type-chart]に示す。

![Type chart: flatMap for Option](src/pages/monads/option-flatmap.pdf+svg){#fig:monads:option-type-chart}

計算の結果は `Option` オブジェクトなので、それに対して再び `flatMap` を呼び出すことができ、一連の計算が続く。これによりフェイルファストなエラー処理が実現する。この仕組みでは、あるステップが `None` を返したら全体として結果が `None` になる。

```scala mdoc
stringDivideBy("6", "2")
stringDivideBy("6", "0")
stringDivideBy("6", "foo")
stringDivideBy("bar", "2")
```

すべてのモナドはファンクターでもある（証明は後述）。したがって、`flatMap` と `map` の両方を使うことで、計算が新しいモナドを生成する場合もしない場合も、順に連結することができる。さらに、`flatMap` と `map` の両方があれば、for 内包表記を使って一連の処理をわかりやすく記述できる。

```scala mdoc:invisible:reset-object
def parseInt(str: String): Option[Int] =
  scala.util.Try(str.toInt).toOption

def divide(a: Int, b: Int): Option[Int] =
  if(b == 0) None else Some(a / b)
```
```scala mdoc:silent
def stringDivideBy(aStr: String, bStr: String): Option[Int] =
  for {
    aNum <- parseInt(aStr)
    bNum <- parseInt(bStr)
    ans  <- divide(aNum, bNum)
  } yield ans
```


### モナドとしての `List`

Scala を学び始めた頃に `flatMap` に出会うと、それを `List` に対する反復処理のひとつのパターンとして捉えがちである。この先入観は、for 内包表記の構文が命令形の for ループとよく似ていることと相まって一層強固なものとなる。

```scala mdoc
for {
  x <- (1 to 3).toList
  y <- (4 to 5).toList
} yield (x, y)
```

しかし、`List` のモナド的な振る舞いに着目した別のメンタルモデルもあり得る。`List` を計算の途中結果の集合として考えれば、`flatMap` は順列や組み合わせを計算する構造となる。

たとえば、上記の for 内包表記では、`x` の取りうる値は3つ、`y` は2つ存在する。その結果、`(x, y)` が取りうる組み合わせは6パターンとなる。`flatMap` は、一連の操作を記述したコードからこれらの組み合わせを生成している。

- `x` を取得する
- `y` を取得する
- タプル `(x, y)` を生成する

<!--
The type chart in Figure [@fig:monads:list-type-chart]
illustrates this behaviour[^list-lengths].

![Type chart: flatMap for List](src/pages/monads/list-flatmap.pdf+svg){#fig:monads:list-type-chart}

[^list-lengths]: Although the result of `flatMap` (`List[B]`)
is the same type as the result of the user-supplied function,
the end result is actually a larger list
created from combinations of intermediate `As` and `Bs`.
-->

### モナドとしての `Future`

`Future` は、計算を、それが非同期であるかどうかを気にすることなく順に連結することができるモナドである。

```scala mdoc:silent
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def doSomethingLongRunning: Future[Int] = ???
def doSomethingElseLongRunning: Future[Int] = ???

def doSomethingVeryLongRunning: Future[Int] =
  for {
    result1 <- doSomethingLongRunning
    result2 <- doSomethingElseLongRunning
  } yield result1 + result2
```

ここでも、各ステップで実行するコードさえ指定すれば、背後に存在するスレッドプールやスケジューラといった恐ろしい複雑性はすべて `flatMap` が処理してくれる。

`Future` を何度も利用していると、上記のコードが各処理を*順次*実行していることに気付くだろう。for 内包表記を展開して、ネストした `flatMap` 呼び出しに置き換えると、そのことは一層明確になる。

```scala mdoc:invisible:reset-object
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
def doSomethingLongRunning: Future[Int] = ???
def doSomethingElseLongRunning: Future[Int] = ???
```
```scala mdoc:silent
def doSomethingVeryLongRunning: Future[Int] =
  doSomethingLongRunning.flatMap { result1 =>
    doSomethingElseLongRunning.map { result2 =>
      result1 + result2
    }
  }
```

一連の計算における各 `Future` は、ひとつ前の `Future` から結果を受け取る関数によって生成されている。言い換えると、計算の各ステップは、前のステップが終了して初めて開始できる。このことは、図[@fig:monads:future-type-chart]に示す `flatMap` の型チャートにおいて関数パラメータの型を `A => Future[B]` としていることからも見て取れる。

![Type chart: flatMap for Future](src/pages/monads/future-flatmap.pdf+svg){#fig:monads:future-type-chart}

もちろん、`Future` を並列実行することもできる。だが、それはまた別の話とし、別の機会に語ることにしよう。モナドは計算を順に連結することがすべてである。

### モナドの定義

ここまで `flatMap` についてだけ述べてきたが、モナド的な振る舞いは、形式的には次のふたつの操作によって捉えられる。

- `pure`。`A => F[A]` という型をもつ
- `flatMap`[^bind]。`(F[A], A => F[B]) => F[B]` という型をもつ

[^bind]: プログラミングに関する文献や Haskell では、`pure` は `point` や `return`、`flatMap` は `bind` や `>>=` と呼ばれることがある。これは単に用語の違いに過ぎない。ここでは、Cats や Scala 標準ライブラリに合わせて `flatMap` という用語を使用する。

`pure` はコンストラクタを抽象化したものであり、通常の値から新たなモナド的コンテキストを作成する手段を提供する。`flatMap` は、すでに見たように計算の順次処理のステップを提供するもので、コンテキストから値を取り出し、一連の計算における次のコンテキストを生成する。以下は、Cats における `Monad` 型クラスの定義を簡略化したものである。

```scala mdoc:silent

trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]
}
```

<div class="callout callout-warning">
#### モナド則 {-}

`pure` と `flatMap` は、一連の操作を副作用や不具合なく自在につなげるために、いくつかの法則を満たす必要がある。

*左単位律（left identity）*。`pure` を呼び出してその結果を `func` で変換することは、`func` を直接呼び出すことと同じである。

```scala
pure(a).flatMap(func) == func(a)
```

*右単位律（right identity）*。`pure` を `flatMap` に渡すことは何もしないのと同じである。

```scala
m.flatMap(pure) == m
```

*結合律*。ふたつの関数 `f` と `g` で順次 `flatMap` することは、`f` を実行した結果を `g` で `flatMap` するという処理で `flatMap` するのと同じである。

```scala
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
```
</div>

### 演習: Getting Func-y

すべてのモナドはファンクターでもある。どのモナドに対しても、既存のメソッドである `flatMap` と `pure` を使った同じ方法で `map` を定義することができる。

```scala mdoc:silent:reset-object

trait Monad[F[_]] {
  def pure[A](a: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    ???
}
```

`map` を定義せよ。

<div class="solution">

これは一見したところ難しそうに見える。だが、型に従って考えれば、解法はひとつしかないことがわかるだろう。`map` メソッドには `F[A]` 型の `value` を渡されている。利用できる手段を考えれば、できることはひとつしかない。`flatMap` を呼び出すことである。

```scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    flatMap(value)(a => ???)
}
```

`flatMap` のふたつ目のパラメータとして `A => F[B]` 型の関数が必要である。関数の実装に使うことのできる部品としては、`A => B` 型の `func` パラメータと、`A => F[A]` 型の `pure` 関数が手元にある。これらを組み合わせることで、目的の結果が得られる。

```scala mdoc:invisible:reset-object
```
```scala mdoc
trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    flatMap(value)(a => pure(func(a)))
}
```
</div>
