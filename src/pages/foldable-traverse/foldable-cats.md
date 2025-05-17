<!--

### Foldable in Cats

Cats' `Foldable` abstracts `foldLeft` and `foldRight` into a type class.
Instances of `Foldable` define these two methods
and inherit a host of derived methods.
Cats provides out-of-the-box instances of `Foldable`
for a handful of Scala data types:
`List`, `Vector`, `LazyList`, and `Option`.

We can summon instances as usual using `Foldable.apply`
and call their implementations of `foldLeft` directly.
Here is an example using `List`:

```scala mdoc:silent
import cats.Foldable
import cats.instances.list._ // for Foldable

val ints = List(1, 2, 3)
```

```scala mdoc
Foldable[List].foldLeft(ints, 0)(_ + _)
```

Other sequences like `Vector` and `LazyList` work in the same way.
Here is an example using `Option`,
which is treated like a sequence of zero or one elements:

```scala mdoc:silent
import cats.instances.option._ // for Foldable

val maybeInt = Option(123)
```

```scala mdoc
Foldable[Option].foldLeft(maybeInt, 10)(_ * _)
```

#### Folding Right

`Foldable` defines `foldRight` differently to `foldLeft`,
in terms of the `Eval` monad:

```scala
def foldRight[A, B](fa: F[A], lb: Eval[B])
                     (f: (A, Eval[B]) => Eval[B]): Eval[B]
```

Using `Eval` means folding is always *stack safe*,
even when the collection's default definition of `foldRight` is not.
For example, the default implementation of `foldRight` for `LazyList` is not stack safe.
The longer the lazy list, the larger the stack requirements for the fold.
A sufficiently large lazy list will trigger a `StackOverflowError`:

```scala mdoc:silent
import cats.Eval
import cats.Foldable

def bigData = (1 to 100000).to(LazyList)
```

```scala mdoc:fail:invisible
// This example isn't printed... it's here to check the next code block is ok:
bigData.foldRight(0L)(_ + _)
```

```scala
bigData.foldRight(0L)(_ + _)
// java.lang.StackOverflowError ...
```

Using `Foldable` forces us to use stack safe operations,
which fixes the overflow exception:

```scala mdoc:silent
import cats.instances.lazyList._ // for Foldable
```

```scala mdoc:silent
val eval: Eval[Long] =
  Foldable[LazyList].
    foldRight(bigData, Eval.now(0L)) { (num, eval) =>
      eval.map(_ + num)
    }
```

```scala mdoc
eval.value
```

<div class="callout callout-info">
*Stack Safety in the Standard Library*

Stack safety isn't typically an issue when using the standard library.
The most commonly used collection types, such as `List` and `Vector`,
provide stack safe implementations of `foldRight`:

```scala mdoc
(1 to 100000).toList.foldRight(0L)(_ + _)
(1 to 100000).toVector.foldRight(0L)(_ + _)
```

We've called out `Stream` because it is an exception to this rule.
Whatever data type we're using, though,
it's useful to know that `Eval` has our back.
</div>

#### Folding with Monoids

`Foldable` provides us with
a host of useful methods defined on top of `foldLeft`.
Many of these are facsimiles of familiar methods from the standard library:
`find`, `exists`, `forall`, `toList`, `isEmpty`, `nonEmpty`, and so on:

```scala mdoc
Foldable[Option].nonEmpty(Option(42))

Foldable[List].find(List(1, 2, 3))(_ % 2 == 0)
```

In addition to these familiar methods,
Cats provides two methods that make use of `Monoids`:

- `combineAll` (and its alias `fold`) combines
  all elements in the sequence using their `Monoid`;

- `foldMap` maps a user-supplied function over the sequence
  and combines the results using a `Monoid`.

For example, we can use `combineAll` to sum over a `List[Int]`:

```scala mdoc:silent
import cats.instances.int._ // for Monoid
```

```scala mdoc
Foldable[List].combineAll(List(1, 2, 3))
```

Alternatively, we can use `foldMap`
to convert each `Int` to a `String` and concatenate them:

```scala mdoc:silent
import cats.instances.string._ // for Monoid
```

```scala mdoc
Foldable[List].foldMap(List(1, 2, 3))(_.toString)
```

Finally, we can compose `Foldables`
to support deep traversal of nested sequences:

```scala mdoc:invisible:reset-object
import cats.Foldable
import cats.instances.list._
import cats.instances.int._
import cats.instances.string._
```
```scala mdoc:silent
import cats.instances.vector._ // for Monoid

val ints = List(Vector(1, 2, 3), Vector(4, 5, 6))
```

```scala mdoc
(Foldable[List] compose Foldable[Vector]).combineAll(ints)
```

#### Syntax for Foldable

Every method in `Foldable` is available in syntax form
via [`cats.syntax.foldable`][cats.syntax.foldable].
In each case, the first argument to the method on `Foldable`
becomes the receiver of the method call:

```scala mdoc:silent
import cats.syntax.foldable._ // for combineAll and foldMap
```

```scala mdoc
List(1, 2, 3).combineAll

List(1, 2, 3).foldMap(_.toString)
```

<div class="callout callout-info">
*Explicits over Implicits*

Remember that Scala will only use an instance of `Foldable`
if the method isn't explicitly available on the receiver.
For example, the following code will
use the version of `foldLeft` defined on `List`:

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ + _)
```

whereas the following generic code will use `Foldable`:

```scala mdoc:silent
```

```scala mdoc
def sum[F[_]: Foldable](values: F[Int]): Int =
  values.foldLeft(0)(_ + _)
```

We typically don't need to worry about this distinction. It's a feature!
We call the method we want and the compiler uses a `Foldable` when needed
to ensure our code works as expected.
If we need a stack-safe implementation of `foldRight`,
using `Eval` as the accumulator is enough to
force the compiler to select the method from Cats.
</div>


```scala mdoc:reset:silent
```
--->

### Cats における `Foldable`

Cats の `Foldable` は、`foldLeft` と `foldRight` を型クラスとして抽象化している。`Foldable` インスタンスはこれらふたつのメソッドを定義し、多くの派生メソッドを継承する。Cats では、`List`、`Vector`、`LazyList`、`Option` といったいくつかの Scala 組み込みデータ型に対する `Foldable` インスタンスが、はじめから用意されている。

いつもと同じように `Foldable.apply` を使ってインスタンスを入手し、その `foldLeft` 実装を直接呼び出すことができる。以下に `List` を用いた例を示す。

```scala mdoc:silent
import cats.Foldable
import cats.instances.list._ // Foldable

val ints = List(1, 2, 3)
```

```scala mdoc
Foldable[List].foldLeft(ints, 0)(_ + _)
```

Other sequences like `Vector` and `LazyList` work in the same way.
Here is an example using `Option`,
which is treated like a sequence of zero or one elements:

`Vector` や `LazyList` のような他のシーケンスでも同じように動作する。以下は `Option` を使用する例である。`Option` はゼロ個またはひとつの要素をもったシーケンスとして扱われる。

```scala mdoc:silent
import cats.instances.option._ // Foldable

val maybeInt = Option(123)
```

```scala mdoc
Foldable[Option].foldLeft(maybeInt, 10)(_ * _)
```

#### 右畳み込み

`Foldable` では、`foldRight` が `foldLeft` とは異なり `Eval` モナドを用いて定義されている。

```scala
def foldRight[A, B](fa: F[A], lb: Eval[B])
                     (f: (A, Eval[B]) => Eval[B]): Eval[B]
```

`Eval` を使うことで、常に*スタックセーフ*な畳み込みを実現できる。コレクションがもつ `foldRight` のデフォルト定義がスタックセーフでなくても関係ない。たとえば、`LazyList` における `foldRight` のデフォルト実装はスタックセーフではなく、リストが長くなるほど畳み込みに必要なスタックも増大する。一定以上の長さの `LazyList` は `StackOverflowError` を引き起こしてしまう。

```scala mdoc:silent
import cats.Eval
import cats.Foldable

def bigData = (1 to 100000).to(LazyList)
```

```scala mdoc:fail:invisible
// This example isn't printed... it's here to check the next code block is ok:
bigData.foldRight(0L)(_ + _)
```

```scala
bigData.foldRight(0L)(_ + _)
// java.lang.StackOverflowError ...
```

`Foldable` を使用することでスタックセーフな操作を強制されるため、このスタックオーバーフローの例外は解消される。

```scala mdoc:silent
import cats.instances.lazyList._ // Foldable
```

```scala mdoc:silent
val eval: Eval[Long] =
  Foldable[LazyList].
    foldRight(bigData, Eval.now(0L)) { (num, eval) =>
      eval.map(_ + num)
    }
```

```scala mdoc
eval.value
```

<div class="callout callout-info">
*標準ライブラリのスタックセーフ性*

標準ライブラリを使用する際、スタックセーフ性が問題になることは通常ない。`List` や `Vector` といった使用頻度のもっとも高いコレクション型は、スタックセーフな `foldRight` 実装を提供している。

```scala mdoc
(1 to 100000).toList.foldRight(0L)(_ + _)
(1 to 100000).toVector.foldRight(0L)(_ + _)
```

`Stream` はこのルールの例外で、スタックセーフではないことに注意してほしい。とはいえ、どのデータ型を使用している場合でも、いざというときには `Eval` に頼れると知っていれば心強い。
</div>

#### モノイドを用いた畳み込み

`Foldable` は `foldLeft` を用いて定義された便利なメソッドを数多く提供している。その多くは、標準ライブラリのよく知られたメソッドを模倣したもので、`find`、`exists`、`forall`、`toList`、`isEmpty`、`nonEmpty` などが含まれる。

```scala mdoc
Foldable[Option].nonEmpty(Option(42))

Foldable[List].find(List(1, 2, 3))(_ % 2 == 0)
```

なじみあるこれらのメソッドに加えて、`Monoid` を用いたふたつのメソッドを Cats は提供している。

- `combineAll`（およびそのエイリアスである `fold`）は、シーケンス内の全要素を `Monoid` を用いて結合する
- `foldMap` は、指定された関数でシーケンスをマッピングし、その結果を `Monoid` を使って結合する

たとえば `combineAll` を使って `List[Int]` の合計を求めることができる。

```scala mdoc:silent
import cats.instances.int._ // Monoid
```

```scala mdoc
Foldable[List].combineAll(List(1, 2, 3))
```

あるいは、`foldMap` を使って各 `Int` を `String` に変換し、それらを連結することもできる。

```scala mdoc:silent
import cats.instances.string._ // Monoid
```

```scala mdoc
Foldable[List].foldMap(List(1, 2, 3))(_.toString)
```

最後に、`Foldable` 同士を合成すれば、入れ子になったシーケンスの深い走査も可能となる。

```scala mdoc:invisible:reset-object
import cats.Foldable
import cats.instances.list._
import cats.instances.int._
import cats.instances.string._
```
```scala mdoc:silent
import cats.instances.vector._ // for Monoid

val ints = List(Vector(1, 2, 3), Vector(4, 5, 6))
```

```scala mdoc
(Foldable[List] compose Foldable[Vector]).combineAll(ints)
```

#### `Foldable` の構文

`Foldable` のメソッドはすべて [`cats.syntax.foldable`][cats.syntax.foldable] を介して構文形式で利用できる。いずれも `Foldable` に定義されているメソッドの最初の引数がメソッド呼び出しのレシーバとなる。

```scala mdoc:silent
import cats.syntax.foldable._ // combineAll と foldMap
```

```scala mdoc
List(1, 2, 3).combineAll

List(1, 2, 3).foldMap(_.toString)
```

<div class="callout callout-info">
*暗黙より明示*

Scala が `Foldable` インスタンスを使用するのは、呼び出そうとするメソッドがレシーバに明示的に定義されていない場合だけだということを覚えておこう。たとえば、次のコードでは `List` に定義された `foldLeft` が使用される。

```scala mdoc
List(1, 2, 3).foldLeft(0)(_ + _)
```

一方で次のジェネリックなコードでは `Foldable` が使用される。

```scala mdoc:silent
```

```scala mdoc
def sum[F[_]: Foldable](values: F[Int]): Int =
  values.foldLeft(0)(_ + _)
```

通常、この区別を意識する必要はない。これは便利な仕組みである。呼び出したいメソッドを指定すれば、コンパイラは必要に応じて `Foldable` を使用し、コードを期待どおりに動かしてくれる。スタックセーフな `foldRight` 実装が必要な場合は、蓄積変数として `Eval` を使用するだけで、Cats のメソッドを選択するようコンパイラに強制できる。
</div>
