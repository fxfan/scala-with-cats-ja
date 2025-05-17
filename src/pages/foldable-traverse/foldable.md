<!--

## Foldable {#sec:foldable}

The `Foldable` type class captures the `foldLeft` and `foldRight` methods
we're used to in sequences like `Lists`, `Vectors`, and `Streams`.
Using `Foldable`, we can write generic folds that work with a variety of sequence types.
We can also invent new sequences and plug them into our code.
`Foldable` gives us great use cases for `Monoids` and the `Eval` monad.

### Folds and Folding

Let's start with a quick recap of the general concept of folding.
We supply an *accumulator* value and a *binary function*
to combine it with each item in the sequence:

```scala mdoc:silent
def show[A](list: List[A]): String =
  list.foldLeft("nil")((accum, item) => s"$item then $accum")
```

```scala mdoc
show(Nil)

show(List(1, 2, 3))
```

The `foldLeft` method works recursively down the sequence.
Our binary function is called repeatedly for each item,
the result of each call becoming the accumulator for the next.
When we reach the end of the sequence,
the final accumulator becomes our final result.

Depending on the operation we're performing,
the order in which we fold may be important.
Because of this there are two standard variants of fold:

- `foldLeft` traverses from "left" to "right" (start to finish);
- `foldRight` traverses from "right" to "left" (finish to start).

Figure [@fig:foldable-traverse:fold] illustrates each direction.

![Illustration of foldLeft and foldRight](src/pages/foldable-traverse/fold.pdf+svg){#fig:foldable-traverse:fold}

`foldLeft` and `foldRight` are equivalent
if our binary operation is associative.
For example, we can sum a `List[Int]` by folding in either direction,
using `0` as our accumulator and addition as our operation:

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ + _)
List(1, 2, 3).foldRight(0)(_ + _)
```

If we provide a non-associative operator
the order of evaluation makes a difference.
For example, if we fold using subtraction,
we get different results in each direction:

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ - _)
List(1, 2, 3).foldRight(0)(_ - _)
```

### Exercise: Reflecting on Folds

Try using `foldLeft` and `foldRight` with an empty list as the accumulator
and `::` as the binary operator. What results do you get in each case?

<div class="solution">
Folding from left to right reverses the list:

```scala mdoc
List(1, 2, 3).foldLeft(List.empty[Int])((a, i) => i :: a)
```

Folding right to left copies the list, leaving the order intact:

```scala mdoc
List(1, 2, 3).foldRight(List.empty[Int])((i, a) => i :: a)
```

Note that we have to carefully specify
the type of the accumulator to avoid a type error.
We use `List.empty[Int]` to avoid
inferring the accumulator type as `Nil.type` or `List[Nothing]`:

```scala mdoc:fail
List(1, 2, 3).foldRight(Nil)(_ :: _)
```
</div>

### Exercise: Scaf-fold-ing Other Methods

`foldLeft` and `foldRight` are very general methods.
We can use them to implement many of the other
high-level sequence operations we know.
Prove this to yourself by implementing substitutes
for `List's` `map`, `flatMap`, `filter`, and `sum` methods
in terms of `foldRight`.

<div class="solution">
Here are the solutions:

```scala mdoc:silent
def map[A, B](list: List[A])(func: A => B): List[B] =
  list.foldRight(List.empty[B]) { (item, accum) =>
    func(item) :: accum
  }
```

```scala mdoc
map(List(1, 2, 3))(_ * 2)
```

```scala mdoc:silent
def flatMap[A, B](list: List[A])(func: A => List[B]): List[B] =
  list.foldRight(List.empty[B]) { (item, accum) =>
    func(item) ::: accum
  }
```

```scala mdoc
flatMap(List(1, 2, 3))(a => List(a, a * 10, a * 100))
```

```scala mdoc:silent
def filter[A](list: List[A])(func: A => Boolean): List[A] =
  list.foldRight(List.empty[A]) { (item, accum) =>
    if(func(item)) item :: accum else accum
  }
```

```scala mdoc
filter(List(1, 2, 3))(_ % 2 == 1)
```

We've provided two definitions of `sum`,
one using `scala.math.Numeric`
(which recreates the built-in functionality accurately)...

```scala mdoc:silent
import scala.math.Numeric

def sumWithNumeric[A](list: List[A])
      (implicit numeric: Numeric[A]): A =
  list.foldRight(numeric.zero)(numeric.plus)
```

```scala mdoc
sumWithNumeric(List(1, 2, 3))
```

and one using `cats.Monoid`
(which is more appropriate to the content of this book):

```scala mdoc:silent
import cats.Monoid

def sumWithMonoid[A](list: List[A])
      (implicit monoid: Monoid[A]): A =
  list.foldRight(monoid.empty)(monoid.combine)

import cats.instances.int._ // for Monoid
```

```scala mdoc
sumWithMonoid(List(1, 2, 3))
```
</div>


```scala mdoc:reset:silent
```
--->

## Foldable {#sec:foldable}

`Foldable` 型クラスは、`List` や `Vector` や `Stream` といったシーケンス（順序付きコレクションのことを指す。以下同じ）で使われる `foldLeft` や `foldRight` メソッドの特徴を捉えたものである。`Foldable` を使えば、さまざまなシーケンス型に対応する汎用的な畳み込み処理を書くことができる。また、新しいシーケンス型を作成してコードに組み込むことも可能である。`Foldable` は、`Monoid` や `Eval` モナドの実用的なユースケースを示してくれる。

### `fold` 関数と畳み込み処理

まず、畳み込みの一般的な概念を簡単におさらいしよう。畳み込み処理では*蓄積変数*と*二項関数*を与え、その二項関数によって蓄積変数とシーケンス内の各要素を順次組み合わせる。

```scala mdoc:silent
def show[A](list: List[A]): String =
  list.foldLeft("nil")((accum, item) => s"$item then $accum")
```

```scala mdoc
show(Nil)

show(List(1, 2, 3))
```

`foldLeft` メソッドは、シーケンスをたどりながら再帰的に動作する。二項関数は各要素に対して繰り返し呼び出され、その結果が次の繰り返しで使用される蓄積変数となる。シーケンスの終端に到達したときの最終的な蓄積変数が最終結果となる。

実行する操作によっては、どちらの順序で畳み込みを行うかが重要になる場合がある。そのため、畳み込みにはふたつの標準的なバリエーションがある。

- `foldLeft` は、シーケンスを左から右（先頭から末尾）へ走査する
- `foldRight` は、シーケンスを右から左（末尾から先頭）へ走査する

図[@fig:foldable-traverse:fold]に各方向の畳み込みの動作を示す。

![foldLeft と foldRight の処理イメージ](src/pages/foldable-traverse/fold.pdf+svg){#fig:foldable-traverse:fold}

二項演算が結合律を満たす場合、`foldLeft` と `foldRight` は同じ結果を返す。たとえば、蓄積変数の初期値を `0` とし、加算を行う二項関数を用いれば、どちらの方向に畳み込んでも `List[Int]` の合計を求めることができる。

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ + _)
List(1, 2, 3).foldRight(0)(_ + _)
```

結合的でない演算を用いた場合、評価順序によって結果は異なってくる。たとえば、減算を用いて畳み込みを行えば、処理の方向によって異なる結果が得られる。

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ - _)
List(1, 2, 3).foldRight(0)(_ - _)
```

### 演習: 畳み込み処理の振り返り

蓄積変数の初期値に空のリストを、二項演算に `::` を用いて `foldLeft` と `foldRight` を試せ。それぞれの場合でどのような結果が得られるだろうか。

<div class="solution">
左から右への畳み込みはリストを逆順にする。

```scala mdoc
List(1, 2, 3).foldLeft(List.empty[Int])((a, i) => i :: a)
```

右から左への畳み込みはリストの順序を保ったまま複製する。

```scala mdoc
List(1, 2, 3).foldRight(List.empty[Int])((i, a) => i :: a)
```

蓄積変数の型を正しく指定しないと型エラーになる点に注意が必要である。`List.empty[Int]` を使用することで、蓄積変数の型が `Nil.type` や `List[Nothing]` と推論されるのを避けている。

```scala mdoc:fail
List(1, 2, 3).foldRight(Nil)(_ :: _)
```
</div>

### 演習: `fold` で他のメソッドを実装する

`foldLeft` と `foldRight` は非常に汎用的なメソッドであり、これらを使って他の多くの高レベルなシーケンス操作を実装することができる。`foldRight` を利用して、`List` の `map`、`flatMap`、`filter`、および `sum` メソッドを再実装し、このことを確かめよ。

<div class="solution">
解答例は以下のとおりである。

```scala mdoc:silent
def map[A, B](list: List[A])(func: A => B): List[B] =
  list.foldRight(List.empty[B]) { (item, accum) =>
    func(item) :: accum
  }
```

```scala mdoc
map(List(1, 2, 3))(_ * 2)
```

```scala mdoc:silent
def flatMap[A, B](list: List[A])(func: A => List[B]): List[B] =
  list.foldRight(List.empty[B]) { (item, accum) =>
    func(item) ::: accum
  }
```

```scala mdoc
flatMap(List(1, 2, 3))(a => List(a, a * 10, a * 100))
```

```scala mdoc:silent
def filter[A](list: List[A])(func: A => Boolean): List[A] =
  list.foldRight(List.empty[A]) { (item, accum) =>
    if(func(item)) item :: accum else accum
  }
```

```scala mdoc
filter(List(1, 2, 3))(_ % 2 == 1)
```

`sum` についてはふたつの定義を示す。ひとつは `scala.math.Numeric` を使う方法で、これは Scala 組み込みの機能を正確に再現している。

```scala mdoc:silent
import scala.math.Numeric

def sumWithNumeric[A](list: List[A])
      (implicit numeric: Numeric[A]): A =
  list.foldRight(numeric.zero)(numeric.plus)
```

```scala mdoc
sumWithNumeric(List(1, 2, 3))
```

もうひとつは `cats.Monoid` を使う。本書の内容としてはこちらのほうが適しているだろう。

```scala mdoc:silent
import cats.Monoid

def sumWithMonoid[A](list: List[A])
      (implicit monoid: Monoid[A]): A =
  list.foldRight(monoid.empty)(monoid.combine)

import cats.instances.int._ // Monoid
```

```scala mdoc
sumWithMonoid(List(1, 2, 3))
```
</div>
