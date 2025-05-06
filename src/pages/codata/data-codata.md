<!--

## Relating Data and Codata

In this section we'll explore the relationship between data and codata, and in paritcular converting one to the other. We'll look at it in two ways: firstly a very surface-level relationship between the two, and then a deep connection via `fold`.

Remember that data is a sum of products, where the products are constructors and we can view constructors as functions. So we can view data as a sum of functions. Meanwhile, codata is a product of functions. We can easily make a direct correspondence between the functions-as-constructors and the functions in codata. What about the difference between the sum and the product that remains.
Well, when we have a product of functions we only call one at any point in our code. So the logical or is in the choice of function to call.

Let's see how this works with a familiar example of data, `List`. As an algebraic data type we can define

```scala
enum List[A] {
  case Pair(head: A, tail: List[A])
  case Empty()
}
```

The codata equivalent is

```scala
trait List[A] {
  def pair(head: A, tail: List[A]): List[A]
  def empty: List[A]
}
```

In the codata implementation we are explicitly representing the constructors as methods, and pushing the choice of constructor to the caller. In a few chapters we'll see a use for this relationship, but for now we'll leave it and move on.

The other way to view the relationship is a connection via `fold`.
We've already learned how to derive the `fold` for any algebraic data type. For `Bool`, defined as

```scala
enum Bool {
  case True
  case False
}
```

the `fold` method is

```scala mdoc:silent
enum Bool {
  case True
  case False
  
  def fold[A](t: A)(f: A): A =
    this match {
      case True => t
      case False => f
    }
}
```

We know that `fold` is universal: we can write any other method in terms of it. It therefore provides a universal destructor and is the key to treating data as codata. In this case the `fold` is something we use all the time, except we usually call it `if`.

Here's the codata version of `Bool`, with `fold` renamed to `if`. (Note that Scala allows us to define methods with the same name as key words, in this case `if`, but we have to surround them in backticks to use them.)

```scala mdoc:reset:silent
trait Bool {
  def `if`[A](t: A)(f: A): A
}
```

Now we can define the two instances of `Bool` purely as codata.

```scala mdoc:silent
val True = new Bool {
  def `if`[A](t: A)(f: A): A = t
}

val False = new Bool {
  def `if`[A](t: A)(f: A): A = f
}
```

Let's see this in use by defining `and` in terms of `if`, and then creating some examples.
First the definition of `and`.

```scala mdoc:silent
def and(l: Bool, r: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      l.`if`(r)(False).`if`(t)(f)
  }
```

Now the examples. This is simple enough that we can try the entire truth table.

```scala mdoc
and(True, True).`if`("yes")("no")
and(True, False).`if`("yes")("no")
and(False, True).`if`("yes")("no")
and(False, False).`if`("yes")("no")
```

#### Exercise: Or and Not {-}

Test your understanding of `Bool` by implementing `or` and `not` in the same way we implemented `and` above.

<div class="solution">
We can follow the same structure as `and`.

```scala mdoc:silent
def or(l: Bool, r: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      l.`if`(True)(r).`if`(t)(f)
  }

def not(b: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      b.`if`(False)(True).`if`(t)(f)
  }
```

Once again, we can test the entire truth table.

```scala mdoc
or(True, True).`if`("yes")("no")
or(True, False).`if`("yes")("no")
or(False, True).`if`("yes")("no")
or(False, False).`if`("yes")("no")

not(True).`if`("yes")("no")
not(False).`if`("yes")("no")
```
</div>

Notice that, once again, computation only happens on demand. In this case, nothing happens until `if` is actually called. Until that point we're just building up a representation of what we want to happen. This again points to how codata can handle infinite data, by only computing the finite amount required by the actual computation.


The rules here for converting from data to codata are:

1. On the interface (`trait`) defining the codata, define a method with the same signature as `fold`.
2. Define an implementation of the interface for each product case in the data. The data's constructor arguments become constructor arguments on the codata `classes`. If there are no constructor arguments, as in `Bool`, we can define values instead of classes.
3. Each implementation implements the case of `fold` that it corresponds to.

Let's apply this to a slightly more complex example: `List`. We'll start by defining it as data and implementing `fold`. I've chosen to implement `foldRight` but `foldLeft` would be just as good.

```scala mdoc:silent
enum List[A] {
  case Pair(head: A, tail: List[A])
  case Empty()
  
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    this match { 
      case Pair(head, tail) => f(head, tail.foldRight(empty)(f))
      case Empty() => empty
    }
}
```

Now let's implement it as codata. We start by defining the interface with the `fold` method. In this case I'm calling it `foldRight` as it's going to exactly mirror the `foldRight` we just defined.

```scala mdoc:reset:silent
trait List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B
}
```

Now we define the implementations. There is one for `Pair` and one for `Empty`, which are the two cases in data definition of `List`. Notice that in this case the classes have constructor arguments, which correspond to the constructor arguments on the correspnding product types.

```scala
final class Pair[A](head: A, tail: List[A]) extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    ???
}

final class Empty[A]() extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    ???
}
```

I didn't implement the bodies of` foldRight` so I could show this as a separate step. The implementation here directly mirrors `foldRight` on the data implementation, and we can use the same strategies to implement the codata equivalents. That is to say, we can use the recursion rule, reasoning by case, and following the types. I'm going to skip these details as we've already gone through them in depth. The final code is shown below.

```scala mdoc:silent
final class Pair[A](head: A, tail: List[A]) extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    f(head, tail.foldRight(empty)(f))
}

final class Empty[A]() extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    empty
}
```

This code is almost the same as the dynamic dispatch implementation, which again shows the relationship between codata and object-oriented code.

The transformation from data to codata goes under several names: **refunctionalization**, **Church encoding**, and **Böhm-Berarducci encoding**. The latter two terms specifically refer to transformations into the untyped and typed lambda calculus respectively. The lambda calculus is a simple model programming language that contains only functions. We're going to take a quick detour to show that we can, indeed, encode lists using just functions. This demonstrates that objects and functions have equivalent power.

The starting point is creating a type alias `List`, which defines a list as a fold. This uses a polymorphic function type, which is new in Scala 3. Inspect the type signature and you'll see it is the same as `foldRight` above.

```scala mdoc:reset:silent
type List[A, B] = (B, (A, B) => B) => B
```
Now we can define `Pair` and `Empty` as functions. The first parameter list is the constructor arguments, and the second parameter list is the parameters for `foldRight`.

```scala mdoc:silent
val Empty: [A, B] => () => List[A, B] = 
  [A, B] => () => (empty, f) => empty

val Pair: [A, B] => (A, List[A, B]) => List[A, B] =
  [A, B] => (head: A, tail: List[A, B]) => (empty, f) => 
    f(head, tail(empty, f))
```

Finally, let's see an example to show it working.
We will first define the list containing `1`, `2`, `3`.
Due to a restriction in polymorphic function types, I have to add the useless empty parameter.

```scala mdoc:silent
val list: [B] => () => List[Int, B] = 
  [B] => () => Pair(1, Pair(2, Pair(3, Empty())))
```

Now we can compute the sum and product of the elements in this list.

```scala mdoc
val sum = list()(0, (a, b) => a + b)
val product = list()(1, (a, b) => a * b)
```

It works!

The purpose of this little demonstration is to show that functions are just objects (in the codata sense) with a single method. Scala this makes apparent, as functions *are* objects with an `apply` method.

We've seen that data can be translated to codata. The reverse is also possible: we simply tabulate the results of each possible method call. In other words, the data representation is memoisation, a lookup table, or a cache.

Although we can convert data to codata and vice versa, there are good reasons to choose one over the other. We've already seen one reason: with codata we can represent infinite structures. In this next section we'll see another difference: the extensibility that data and codata permit.


```scala mdoc:reset:silent
```
--->

## データと余データの関係

この節では、データと余データの関係、特にそれらの間の相互変換について探求する。まずは両者の表面的な関係を確認し、その後、`fold` を通じた深い結びつきを見ていく。

データは積の和で構成されており、その積の部分がコンストラクタとなることを思い出してほしい。コンストラクタは関数として見ることができるので、データは関数の和として捉えることができる。一方、余データは関数の積である。コンストラクタとしての関数と余データにおける関数の間には、直接的な対応関係を簡単に見出すことができる。では、和と積の違いはどうだろうか。関数の積があるとき、コードのある場所において呼び出すことのできる関数はどれかひとつだけである。つまり、論理和はどの関数を呼び出すかという選択のなかに存在する。

そのことを、データの代表例である `List` を使って見てみよう。`List` は代数的データ型として次のように定義できる。

```scala
enum List[A] {
  case Pair(head: A, tail: List[A])
  case Empty()
}
```

余データに相当するものは以下のとおりである。

```scala
trait List[A] {
  def pair(head: A, tail: List[A]): List[A]
  def empty: List[A]
}
```

この余データの定義では、コンストラクタを明示的にメソッドとして表現し、コンストラクタの選択を呼び出し側に委ねている。この関係性が役に立つ場面をすこし先の章で見ることになるが、今はその話題から離れ、次に進むことにする。

両者の関係を見るもうひとつの方法は `fold` を通じたつながりである。任意の代数的データ型に対して `fold` を導出する方法はすでに学んだ。たとえば、次のように定義される `Bool` について考えてみよう。

```scala
enum Bool {
  case True
  case False
}
```

`fold` メソッドは以下のように実装される。

```scala mdoc:silent
enum Bool {
  case True
  case False
  
  def fold[A](t: A)(f: A): A =
    this match {
      case True => t
      case False => f
    }
}
```

知ってのとおり、`fold` には、どんなメソッドでもこれを用いて記述できるという汎用性がある。それゆえ `fold` は汎用デストラクタでもあり、データを余データとして扱うための鍵でもある。`Bool` において `fold` は、我々が普段は `if` 式と呼び、頻繁に使っているものである。

以下は、`Bool` の余データバージョンである。`fold` は `if` にリネームした（ Scala では、`if` のようなキーワードと同名のメソッドを定義できるが、キーワードを識別子として利用する際にはバッククォートで囲む必要がある）。

```scala mdoc:reset:silent
trait Bool {
  def `if`[A](t: A)(f: A): A
}
```

次に、`Bool` のふたつのインスタンスを純粋に余データとして定義する。

```scala mdoc:silent
val True = new Bool {
  def `if`[A](t: A)(f: A): A = t
}

val False = new Bool {
  def `if`[A](t: A)(f: A): A = f
}
```

この `if` メソッドを実際に使ってみよう。`if` を用いて `and` を定義し、その使用例をいくつか示す。

```scala mdoc:silent
def and(l: Bool, r: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      l.`if`(r)(False).`if`(t)(f)
  }
```

以下がその例である。`and` は単純なので、真理値表のすべての組み合わせを試すことができる。

```scala mdoc
and(True, True).`if`("yes")("no")
and(True, False).`if`("yes")("no")
and(False, True).`if`("yes")("no")
and(False, False).`if`("yes")("no")
```

#### 演習: `Bool` 余データへの `or` と `not`　の実装 {-}

`Bool` 余データについての理解度を確認するため、上記の `and` を実装したのと同じ方法で `or` と `not` を実装せよ。

<div class="solution">
`and` の構造に倣って実装することができる。

```scala mdoc:silent
def or(l: Bool, r: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      l.`if`(True)(r).`if`(t)(f)
  }

def not(b: Bool): Bool =
  new Bool {
    def `if`[A](t: A)(f: A): A =
      b.`if`(False)(True).`if`(t)(f)
  }
```

これらについても、真理値表のすべての組み合わせを確認しておこう。

```scala mdoc
or(True, True).`if`("yes")("no")
or(True, False).`if`("yes")("no")
or(False, True).`if`("yes")("no")
or(False, False).`if`("yes")("no")

not(True).`if`("yes")("no")
not(False).`if`("yes")("no")
```
</div>

ここでもやはり、計算は要求されたときにだけ実行されるという点に注目してほしい。これらの例では、`if` が実際に呼び出されるまで何も起こらない。その時点までは、実行したい処理の表現を組み立てているだけである。余データがいかにして無限のデータを扱うか、実際に必要な有限の量だけが計算される仕組みによってそれが実現されている、ということが、ここでも示されている。

データから余データに変換する際のルールを以下に挙げる。

1. 余データを定義するインターフェース（`trait`）上に、`fold` と同シグネチャのメソッドを定義する
2. データにおける積のそれぞれについて、上記インターフェースの実装を定義する。データのコンストラクタ引数は、余データの実装クラスにおけるコンストラクタ引数となる。`Bool` のようにコンストラクタが引数をもたない場合は、クラスの代わりに値を定義すればよい
3. 実装クラスはそれぞれ、データにおける `fold` の対応するケースを実装する

もうすこし複雑な例として、これらのルールを `List` に適用してみよう。まずは `List` をデータとして定義し、そこに `fold` を実装するところから始める。ここでは `foldRight` を実装することにしたが、`foldLeft` を選んでも構わない。

```scala mdoc:silent
enum List[A] {
  case Pair(head: A, tail: List[A])
  case Empty()
  
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    this match { 
      case Pair(head, tail) => f(head, tail.foldRight(empty)(f))
      case Empty() => empty
    }
}
```

次にこれを余データとして実装しよう。`fold` メソッドをもったインターフェースを定義するところから始める。ただし、この `fold` は先ほど定義した `foldRight` をそのまま移植したものとなるので、ここでは `foldRight` を呼ぶことにする。

```scala mdoc:reset:silent
trait List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B
}
```

次は実装クラスを定義する。データとしての `List` の定義がもつふたつのケースに従い、`Pair` と `Empty` それぞれのための実装クラスが必要である。実装クラス `Pair` は、データ側の直和型 `Pair` に対応するコンストラクタ引数をもつことに留意してほしい。

```scala
final class Pair[A](head: A, tail: List[A]) extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    ???
}

final class Empty[A]() extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    ???
}
```

最後に、前段では実装しなかった `foldRight` のボディ部について見ていく。余データにおけるこの実装は、データにおける `foldRight` をそのまま移植したものなので、そのボディ部を導き出すのにも同じ戦略を用いることができる。具体的には、再帰におけるルール、ケース別の推論、型追従がそれにあたるが、それらの戦略の詳細はすでに見てきたのでここでは触れない。最終的なコードは以下のとおりである。

```scala mdoc:silent
final class Pair[A](head: A, tail: List[A]) extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    f(head, tail.foldRight(empty)(f))
}

final class Empty[A]() extends List[A] {
  def foldRight[B](empty: B)(f: (A, B) => B): B =
    empty
}
```

このコードは、動的ディスパッチを学んだときの実装とほとんど同じである。これもやはり、余データとオブジェクト指向コードとの関係を示している。

データから余データへの変換は、**再関数化（refunctionalization）**、**チャーチエンコーディング（Church encoding）**、**ボーム・ベラルドゥッチ・エンコーディング（Böhm-Berarducci encoding）**などいくつかの名前で呼ばれる。うしろふたつの用語は、それぞれ型なしラムダ計算および型付きラムダ計算への変換を指している。ラムダ計算とは関数のみで構成されるシンプルな計算モデルであり、プログラミング言語である。ここですこし本題から脱線し、リストがたしかに関数だけを使った形にエンコード可能であることを確認したい。このことは、オブジェクトと関数が同等の表現力をもっていることを示している。

まず、型エイリアスを用いて、`List` を `fold` 関数型として定義する。この定義には Scala3 で導入された多相的関数型を用いる。型シグネチャを見れば、これが前述の `foldRight` と同じであることがわかるだろう。

```scala mdoc:reset:silent
type List[A, B] = (B, (A, B) => B) => B
```

さらに、`Pair` と `Empty` を関数として定義することができる。コンストラクタ引数だったものが最初のパラメータリストになり、`foldRight` の引数がふたつ目のパラメータリストになる。

```scala mdoc:silent
val Empty: [A, B] => () => List[A, B] = 
  [A, B] => () => (empty, f) => empty

val Pair: [A, B] => (A, List[A, B]) => List[A, B] =
  [A, B] => (head: A, tail: List[A, B]) => (empty, f) => 
    f(head, tail(empty, f))
```

最後に、これが動作することを示す使用例を見てみよう。まず、`1`、`2`、`3` という要素が格納されたリストを定義する。多相的関数型における制約のため、無意味な空のパラメータを追加しなければならない。

```scala mdoc:silent
val list: [B] => () => List[Int, B] = 
  [B] => () => Pair(1, Pair(2, Pair(3, Empty())))
```

そして次のように書けば、このリスト内にある要素すべての合計と積を計算することができる。

```scala mdoc
val sum = list()(0, (a, b) => a + b)
val product = list()(1, (a, b) => a * b)
```

この小さなデモの目的は、関数が（余データ的な意味で）単一のメソッドをもつオブジェクトにすぎないと示すことである。Scala ではそのことが明確で、関数は `apply` メソッドをもつオブジェクトとして実装されている。

データが余データに変換できることを見てきたが、その逆も可能である。それには、各メソッド呼び出しの全パターンについて結果を列挙すればよい。言い換えれば、データ表現とはメモ化であり、検索テーブルであり、またキャッシュのことである。

データと余データは相互に変換可能であるとはいえ、一方を選択するにあたってはそれなりの理由がある。余データを用いることで無限の構造を表現できるというのもそのひとつである。次節では、両者のもうひとつの違いとして、データと余データが許容する拡張性について見ていく。
