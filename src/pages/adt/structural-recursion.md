## 構造的再帰

Structural recursion is our second programming strategy. 
Algebraic data types tell us how to create data given a certain structure.
Structural recursion tells us how to transform an algebraic data types into any other type.
Given an algebraic data type, the transformation can be implemented using structural recursion.

二つ目のプログラミング戦略は構造的再帰である。代数的データ型は、特定の構造に基づいたデータをどのように定義するかを示してくれた。構造的再帰は代数的データ型を他の任意の型に変換する方法を教えてくれる。代数的データ型が与えられた場合、その変換は構造的再帰を使って実装することができる。

As with algebraic data types, there is distinction between the concept of structural recursion and the implementation in Scala.
This is more obvious because there are two ways to implement structural recursion in Scala: via pattern matching or via dynamic dispatch.
We'll look at each in turn.

代数的データ型と同様、構造的再帰の概念とScalaにおける実装は区別して考える必要がある。そのことは、Scalaには構造的再帰を実装する方法が二つあることからも分かるだろう。その二つとは、パターンマッチングを使う方法と動的ディスパッチを使う方法である。これらを順に見ていこう。

### パターンマッチング

I'm assuming you're familiar with pattern matching in Scala, so I'll only talk about how to implement structural recursion using pattern matching.
Remember there are two kinds of algebraic data types: sum types (logical ors) and product types (logical ands).
We have corresponding rules for structural recursion implemented using pattern matching:

Scalaにおけるパターンマッチングについては基本的な知識があることを前提としているので、ここではパターンマッチングを使って構造的再帰をどのように実装するかについてのみ解説する。代数的データ型には直和型（論理和）と直積型（論理積）の二種類があったことを思い出そう。構造的再帰をパターンマッチングで実装する際には、この二種類それぞれに対応するルールがある。

1. For each branch in a sum type we have a distinct `case` in the pattern match; and
2. Each `case` corresponds to a product type with the pattern written in the usual way.

1. 直和型の各分岐は、パターンマッチ内で別々の `case` となる
2. 各 `case` は直積型に対応し、通常の方法でパターンが書かれる

Let's see this in code, using an example ADT that includes both sum and product types:

これをコードで見てみよう。以下のような直和型と直積型の両方を組み合わせた代数的データ型の例を用いる。

- `A` is `B` or `C`; and
- `B` is `D` and `E`; and
- `C` is `F` and `G`

- `A` は `B` または `C` の直和型
- `B` は `D` および `E` の直積型
- `C` は `F` および `G` の直積型

which we represent (in Scala 3) as

Scala3ではこれは次のように表現される。

```scala
enum A {
  case B(d: D, e: E)
  case C(f: F, g: G)
}
```

Following the rules above means a structural recursion would look like

先ほど示したルールに従うと、構造的再帰は以下のようになる。

```scala
anA match {
  case B(d, e) => ???
  case C(f, g) => ???
}
```

The `???` bits are problem specific, and we cannot give a general solution for them. 
However we'll soon see strategies to help create them.

`???` の部分をどう書くかは何をやりたいか次第であり、一般的な解決策を示すことはできないが、それらを実装するのに役立つ戦略についてはこの後に見ていく。

### 構造的再帰における再帰

At this point you might be wondering where the recursion in structural recursion comes from.
This is an additional rule for recursion: whenever the data is recursive the method is recursive in the same place.

この時点では、構造的再帰における「再帰」という言葉がどこから来ているのか疑問に思うかもしれない。再帰には追加のルールがある。データが再帰的である場合、そのメソッドも同じ箇所で再帰的になる、というものである。

Let's see this in action for a real data type.

これを実際のデータ型で見てみよう。

We can define a list with elements of type `A` as:

`A` 型の要素をもつリストは以下のように定義できる。

- the empty list; or
- a pair containing an `A` and a tail, which is a list of `A`.

- リストは空であるか、もしくは
- ひとつの `A` 型の値と、それ自体が `A` のリストである残りの部分のペア

This is exactly the definition of `List` in the standard library.
Notice it's an algebraic data type as it consists of sums and products.
It is also recursive: in the pair case the tail is itself a list.

これは標準ライブラリにおける `List` の定義そのものである。直和型と直積型から構成された代数的データ型である点に注目してほしい。また、この構造は再帰的でもある。ペアのケースでは、末尾の部分自体がひとつのリストになっている。

We can directly translate this to code, using the strategy for algebraic data types we saw previously.
In Scala 3 we write

すでに学んだ代数的データ型の戦略を使えば、これを直接コードに置き換えることができる。Scala3では次のようになる。

```scala mdoc:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

Let's implement `map` for `MyList`.
We start with the method skeleton specifying just the name and types.

試しに `MyList` に `map` を実装してみよう。まずは名前と型のみを指定したメソッドの骨組みからスタートする。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    ???
}
```

Our first step is to recognize that `map` can be written using a structural recursion.
`MyList` is an algebraic data type, `map` is transforming this algebraic data type, and therefore structural recursion is applicable.
We now apply the structural recursion strategy, giving us

最初のステップは、`map` が構造的再帰を使って記述可能であると認識することである。`MyList` は代数的データ型であり、`map` はこの代数的データ型を変換するものなので、構造的再帰が適用できる。実際に構造的再帰の戦略を適用すると、次のようになる。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    this match {
      case Empty() => ???
      case Pair(head, tail) => ???
    }
}
```

I forgot the recursion rule! 
The data is recursive in the `tail` of `Pair`, so `map` is recursive there as well.

再帰のルールについても考慮しておこう。データが再帰的構造をもつ場合、それと同じ場所でメソッドも再帰呼び出しされる。このデータは `Pair` の `tail` の部分が再帰的なので、`map` もそこで再帰的になる。

```scala
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    this match {
      case Empty() => ???
      case Pair(head, tail) => ??? tail.map(f)
    }
}
```

I left the `???` to indicate that we haven't finished with that case.

なお、`???` は、その部分のコードがまだ完成していないことを示している。

Now we can move on to the problem specific parts.
Here we have three strategies to help us:

骨組みが完成したら、取り組んでいる課題に固有の部分に進むことができる。ここでは、以下の三つの戦略が役に立つ。

1. reasoning independently by case; 
2. assuming recursion the is correct; and
3. following the types

1. 各ケースを独立して考えること
2. 再帰呼び出しの結果は正しいと仮定すること
3. 型に従うこと

The first two are specific to structural recursion, while the final one is a general strategy we can use in many situations.
Let's briefly discuss each and then see how they apply to our example.

最初の二つは構造的再帰に特有のもので、最後のひとつは多くの状況で使える一般的な戦略である。それぞれについて簡単に検討し、今回の例にどのように適用するか見ていこう。

The first strategy is relatively simple: when we consider the problem specific code on the right hand side of a pattern matching `case`, we can ignore the code in any other pattern match cases. So, for example, when considering the case for `Empty` above we don't need to worry about the case for `Pair`, and vice versa.

最初の戦略は比較的シンプルである。パターンマッチングにおいて、ある `case` の右側にある問題固有のコードを考える際、他のケースにあるコードは無視してかまわない。たとえば、上の `Empty` のケースを考えるとき、`Pair` のケースを気にする必要はないし、逆も同じことが言える。

The next strategy is a little bit more complicated, and has to do with recursion. Remember that the structural recursion strategy tells us where to place any recursive calls. This means we don't have to think through the recursion. Instead we assume the recursive call will correctly compute what it claims, and only consider how to further process the result of the recursion. The result is guaranteed to be correct so long as we get the non-recursive parts correct. 

次の戦略は少し複雑で、再帰に関係している。構造的再帰の戦略は、再帰呼び出しをどこに配置すればよいか示してくれることを思い出してほしい。このとき、再帰呼び出しの処理内容について深く考える必要はない。代わりに、再帰呼び出しが正しく計算されると仮定し、再帰の結果をどう処理するかだけを考えればよい。再帰以外の部分さえ正確であれば、結果が正しいことは保証される。

In the example above we have the recursion `tail.map(f)`. We can assume this correctly computes `map` on the tail of the list, and we only need to think about what we should do with the remaining data: the `head` and the result of the recursive call. 

上の例には、`tail.map(f)` という再帰呼び出しがある。この再帰呼び出しがリストの `tail` に対する `map` を正しく計算してくれると仮定し、残りのデータ、つまり `head` と再帰呼び出しの結果をどう扱うかだけを考えればよい。

It's this property that allows us to consider cases independently. Recursive calls are the only thing that connect the different cases, and they are given to us by the structural recursion strategy.

各ケースを独立して考えることができるのはこの特性による。再帰呼び出しは異なるケースをつなぐ唯一のものであり、その書き方は構造的再帰戦略によって与えられる。

Our final strategy is **following the types**. It can be used in many situations, not just structural recursion, so I consider it a separate strategy. The core idea is to use the information in the types to restrict the possible implementations. We can look at the types of inputs and outputs to help us.

最後の戦略は、**型に従うこと**である。これは構造的再帰に限らず、多くの状況で使えるため、別の戦略として扱おうと思う。コアとなる考え方は、入出力の型情報を活用して、ありうる実装を限定していくというものである。

Now let's use these strategies to finish the implementation of `map`. We start with

それでは、これらの戦略を使って `map` の実装を完成させよう。すでに実装した部分を以下に再掲し、この続きを考えていく。

```scala
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    this match {
      case Empty() => ???
      case Pair(head, tail) => ??? tail.map(f)
    }
}
```

Our first strategy is to consider the cases independently. Let's start with the `Empty` case. There is no recursive call here, so reasoning about recursion doesn't come into play. Let's instead use the types. There is no input here other than the `Empty` case we have already matched, so we cannot use the input types to further restrict the code. Let's instead consider the output type. We're trying to create a `MyList[B]`. There are only two ways to create a `MyList[B]`: an `Empty` or a `Pair`. To create a `Pair` we need a `head` of type `B`, which we don't have. So we can only use `Empty`. *This is the only possible code we can write*. The types are sufficiently restrictive that we cannot write incorrect code for this case.

最初の戦略は、各ケースを独立に考えることである。まずは `Empty` のケースから始めよう。このケースには再帰呼び出しがないため、再帰について考える必要はない。代わりに型情報を利用しよう。`Empty` ケースにマッチしたということ以外に入力はないため、入力の型を使ってコードを制約することはできない。一方で、出力の型について考えてみると、`map` は `MyList[B]` を作成しようとしていることが分かる。`MyList[B]` を作成する方法は `Empty` か `Pair` の二通りだけである。そして、`Pair` を作成するには、`B` 型 の `head` が必要だが、それに相当する情報は与えられていないので、`Empty` を使うしかない。*これが書くことのできる唯一のコードである*。型が十分に制約を与えているため、このケースで誤ったコードを書くことはできない。


```scala
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    this match {
      case Empty() => Empty()
      case Pair(head, tail) => ??? tail.map(f)
    }
}
```

Now let's move to the `Pair` case. We can apply both the structural recursion reasoning strategy and following the types. Let's use each in turn.

次に `Pair` のケースに移ろう。ここでは構造的再帰の戦略と、型に従うこと、両方を適用することができる。以下に `Pair` のケースを再掲する。

```scala 
case Pair(head, tail) => ??? tail.map(f)
```

Remember we can consider this independently of the other case. We assume the recursion is correct. This means we only need to think about what we should do with the `head`, and how we should combine this result with `tail.map(f)`. Let's now follow the types to finish the code. Our goal is to produce a `MyList[B]`. We already the following available:

他のケースとは独立に考えることができること、そして再帰呼び出しの結果は正しいと仮定することを思い出してほしい。そうすると、ここでは `head` をどう処理し、 `tail.map(f)` の結果とどう結合するかだけを考えればよい、ということになる。あとは型に従ってコードを完成させよう。ゴールは `MyList[B]` 型の値を生成することで、そのために利用できる情報として以下のものがある。

- `tail.map(f)`, which has type `MyList[B]`;
- `head`, with type `A`;
- `f`, with type `A => B`; and
- the constructors `Empty` and `Pair`.

- `tail.map(f)` の呼び出し結果（`MyList[B]` 型）
- `A` 型の値 `head`
- `A` を受けとり `B` を返す関数 `f`
- `Empty` と `Pair` のコンストラクタ

We could return just `Empty`, matching the case we've already written. This has the correct type but we might expect it is not the correct answer because it does not use the result of the recursion, `head`, or `f` in any way.

すでに記述した `Empty` ケースと同じように、単に `Empty` を返すこともできる。これは型的には正しいが、再帰呼び出しの結果や `head`、関数 `f` をまったく使用していないし、正しい答えではないだろうと予想できる。

We could return just `tail.map(f)`. This has the correct type but we might expect it is not correct because we don't use `head` or `f` in any way.

また、単に `tail.map(f)` を返すこともでき、これも型的には正しいが、`head` を使用していないので、やはり正しい答えではないだろう。

We can call `f` on `head`, producing a value of type `B`, and then combine this value and the result of the recursive call using `Pair` to produce a `MyList[B]`. This is the correct solution.

`head` に対して `f` を呼び出して `B` 型の値を生成し、その値と再帰呼び出しの結果を `Pair` を使って組み合わせて `MyList[B]` 型の値を作成することができる。これが正しい解決策である。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    this match {
      case Empty() => Empty()
      case Pair(head, tail) => Pair(f(head), tail.map(f))
    }
}
```

If you've followed this example you've hopefully see how we can use the three strategies to systematically find the correct implementation. Notice how we interleaved the recursion strategy and following the types to guide us to a solution for the `Pair` case. Also note how following the types alone gave us three possible implementations for the `Pair` case. In this code, and as is usually the case, the solution was the implementation that used all of the available inputs.

この例題についてここまで読み進めてきたなら、三つの戦略を使って体系的に正しい実装を見つける方法について理解していただけたと思う。再帰戦略と型に従う戦略を交互に用いることで、`Pair` ケースの解決策へと導かれたことに注目してほしい。また、ただ型に従うだけでも `Pair` ケースの実装方法が三つに絞り込まれた点にも留意してほしい。今回のコードでは、そして大抵の場合もそうだが、利用できるすべての入力を使用する実装が正しい解決策となる。

### 網羅性チェック

Remember that algebraic data types are a closed world: they cannot be extended once defined. 
The Scala compiler can use this to check that we handle all possible cases in a pattern match,
so long as we write the pattern match in a way the compiler can work with.
This is known as exhaustivity checking.

代数的データ型は閉じた世界であり、一旦定義されると拡張できないということを思い出そう。Scalaコンパイラは、この性質を利用し、パターンマッチングにおいてすべてのケースが処理されているかどうかをチェックすることができる。ただし、パターンマッチをコンパイラが処理できる形式で記述する必要がある。これは網羅性チェックと呼ばれている。

Here's a simple example.
We start by defining a straight-forward algebraic data type.

次にシンプルな例を挙げる。まずは素直に代数的データ型を定義するところから始める。

```scala mdoc:silent
// Some of the possible units for lengths in CSS
// CSSにおいて用いられる可能性のある長さの単位
enum CssLength {
  case Em(value: Double)
  case Rem(value: Double)
  case Pt(value: Double)
}
```

If we write a pattern match using the structural recursion strategy,
the compiler will complain if we're missing a case.

構造的再帰の戦略を使ってパターンマッチを書いていれば、ありうるケースが欠けているときにコンパイラが警告を出してくれる。

```scala
import CssLength.*

CssLength.Em(2.0) match {
  case Em(value) => value
  case Rem(value) => value
}
// -- [E029] Pattern Match Exhaustivity Warning: ----------------------------------
// 1 |CssLength.Em(2.0) match {
//   |^^^^^^^^^^^^^^^^^
//   |match may not be exhaustive.
//   |
//   |It would fail on pattern case: CssLength.Pt(_)
//   |
//   | longer explanation available when compiling with `-explain`
```

Exhaustivity checking is incredibly useful.
For example, if we add or remove a case from an algebraic data type, the compiler will tell us all the pattern matches that need to be updated.

網羅性チェックは非常に有用である。たとえば、代数的データ型に新しいケースを追加したり、ケースを削除したりしたときに、どのパターンマッチを更新する必要があるかをコンパイラが教えてくれる。

### 動的ディスパッチ

Using dynamic dispatch to implement structural recursion is an implementation technique that may feel more natural to people with a background in object-oriented programming.

構造的再帰の実装として動的ディスパッチを使用する方法は、オブジェクト指向プログラミングの経験がある人にとって、より自然に感じられる実装テクニックかもしれない。

The dynamic dispatch approach consists of:

動的ディスパッチのアプローチは、以下の手順で構成される。

1. defining an *abstract method* at the root of the algebraic data types; and
2. implementing that abstract method at every leaf of the algebraic data type.

1. 代数的データ型のルートに*抽象メソッド*を定義すること
2. 代数的データ型の各リーフでその抽象メソッドを実装すること

This implementation technique is only available if we use the Scala 2 encoding of algebraic data types.

この実装テクニックは、Scala2における代数的データ型の表現方法を用いている場合にのみ利用可能である。

Let's see it in the `MyList` example we just looked at.
Our first step is to rewrite the definition of `MyList` to the Scala 2 style.

これを、先ほど使った `MyList` の例で見ていこう。最初のステップは `MyList` の定義をScala2のスタイルに書き換えることである。

```scala mdoc:reset:silent
sealed abstract class MyList[A] extends Product with Serializable
final case class Empty[A]() extends MyList[A]
final case class Pair[A](head: A, tail: MyList[A]) extends MyList[A]
```

Next we define an abstract method for `map` on `MyList`.

次に、`MyList` に抽象メソッドとして `map` を定義する。

```scala
sealed abstract class MyList[A] extends Product with Serializable {
  def map[B](f: A => B): MyList[B]
}
final case class Empty[A]() extends MyList[A]
final case class Pair[A](head: A, tail: MyList[A]) extends MyList[A]
```

Then we implement `map` on the concrete subtypes `Empty` and `Pair`.

続いて、具象サブタイプである `Empty` と `Pair` に、`map` メソッドを実装する。

```scala mdoc:reset:silent
sealed abstract class MyList[A] extends Product with Serializable {
  def map[B](f: A => B): MyList[B]
}
final case class Empty[A]() extends MyList[A] {
  def map[B](f: A => B): MyList[B] = 
    Empty()
}
final case class Pair[A](head: A, tail: MyList[A]) extends MyList[A] {
  def map[B](f: A => B): MyList[B] =
    Pair(f(head), tail.map(f))
}
```

We can use exactly the same strategies we used in the pattern matching case to create this code.
The implementation technique is different but the underlying concept is the same.

この `map` の実装を考える際には、パターンマッチングの場合とまったく同じ戦略を使用することができる。実装テクニックは違っても、基礎的な概念は同じである。

Given we have two implementation strategies, which should we use?
If we're using `enum` in Scala 3 we don't have a choice; we must use pattern matching.
In other situations we can choose between the two.
I prefer to use pattern matching when I can, as it puts the entire method definition in one place.
However, Scala 2 in particular has problems inferring types in some pattern matches.
In these situations we can use dynamic dispatch instead.
We'll learn more about this when we look at generalized algebraic data types.

これら二つの実装戦略のうち、どちらを使うべきだろうか。Scala3で `enum` を使用している場合、選択肢はなく、パターンマッチングを使うしかない。他の状況では、どちらを使うか選択できる。私は、可能であればパターンマッチングを使うほうが好みである。そうすることでメソッド定義全体を一か所にまとめることができる。だが、特にScala2では、一部のパターンマッチで型推論に問題が生じることがあり、そのような場合には、動的ディスパッチを使用することもできる。これについては、一般化された代数的データ型について見ていく際にさらに詳しく学ぶ予定である。

#### 演習: `Tree` へのメソッド定義

In a previous exercise we created a `Tree` algebraic data type:

ひとつ前の演習では代数的データ型 `Tree` を作成した。

```scala mdoc:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
}
```

Or, in the Scala 2 encoding:

Scala2の表現方法であれば以下のとおり。

```scala mdoc:reset:silent
sealed abstract class Tree[A] extends Product with Serializable
final case class Leaf[A](value: A) extends Tree[A]
final case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
```

Let's get some practice with structural recursion and write some methods for `Tree`. Implement

構造的再帰の練習として、この `Tree` に対して以下のメソッドを実装せよ。

* `size`, which returns the number of values (`Leafs`) stored in the `Tree`;
* `contains`, which returns `true` if the `Tree` contains a given element of type `A`, and `false` otherwise; and
* `map`, which creates a `Tree[B]` given a function `A => B`

* `size` メソッド: `Tree` に格納されている値（`Leafs`）の件数を返す
* `contains` メソッド: `Tree` が指定された要素を含んでいる場合 `true` を返し、そうでなければ `false` を返す
* `map` メソッド: `A` を `B` に変換する関数を受け取って、`Tree[B]` を返す

Use whichever you prefer of pattern matching or dynamic dispatch to implement the methods.

実装にはパターンマッチングと動的ディスパッチどちらでも好きな方を使ってかまわない。

<div class="solution">

<!-- 
TODO: ここまで "strategy" という言葉を当てていた概念に "reasoning rule" などの別の言葉が使われている。
まぎらわしいので統一を検討する。
-->

I chose to use pattern matching to implement these methods. I'm using the Scala 3 encoding so I have no choice.

この解答では、`Tree` における論理和の表現として `enum` を使い、メソッドの実装にパターンマッチングを用いている。

I start by creating the method declarations with empty bodies.

まずは、ボディは空のままメソッドを宣言するところから始めよう。

```scala mdoc:reset:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def size: Int = 
    ???

  def contains(element: A): Boolean =
    ???
    
  def map[B](f: A => B): Tree[B] =
    ???
}
```

Now these methods all transform an algebraic data type so I can implement them using structural recursion. I write down the structural recursion skeleton for `Tree`, remembering to apply the recursion rule.

これらのメソッドはすべて代数的データ型を変換するので、構造的再帰を使って実装することができる。`Tree` に対する構造的再帰の骨組みを以下のように記述する。再帰呼び出しはデータが再帰的である場所で行われる、というルールも適用済である。

```scala
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def size: Int = 
    this match { 
      case Leaf(value)       => ???
      case Node(left, right) => left.size ??? right.size
    }

  def contains(element: A): Boolean =
    this match { 
      case Leaf(value)       => ???
      case Node(left, right) => left.contains(element) ??? right.contains(element)
    }
    
  def map[B](f: A => B): Tree[B] =
    this match { 
      case Leaf(value)       => ???
      case Node(left, right) => left.map(f) ??? right.map(f)
    }
}
```

Now I can use the other reasoning techniques to complete the method declarations.
Let's work through `size`. 

ここまで書けば、次の推論テクニックを使ってメソッドの定義を完成させることができる。では `size` を実装していこう。

```scala
def size: Int = 
  this match { 
    case Leaf(value)       => 1
    case Node(left, right) => left.size ??? right.size
  }
```

I can reason independently by case.
The size of a `Leaf` is, by definition, 1.

ケースはそれぞれ独立に考えることができる。`Leaf` から見ていこう。 `Leaf` のサイズは、定義により常に1である。

```scala
def size: Int = 
  this match { 
    case Leaf(value)       => 1
    case Node(left, right) => left.size ??? right.size
  }
```

Now I can use the rule for reasoning about recursion: I assume the recursive calls successfully compute the size of the left and right children. What is the size then of the combined tree? It must be the sum of the size of the children. With this, I'm done.

`Node` ケースには、再帰呼び出しの結果を正しいと仮定する考え方を利用することができる。再帰呼び出しが左右の子のサイズを正しく計算していると仮定すると、結合された木のサイズは、左右の子のサイズの合計になるはずである。`size` の実装はこれで完成となる。

```scala
def size: Int = 
  this match { 
    case Leaf(value)       => 1
    case Node(left, right) => left.size + right.size
  }
```

I can use the same process to work through the other two methods, giving me the complete solution shown below.

残りの二つのメソッドも同じプロセスを使って実装できる。以下に、完成した解答を示す。

```scala mdoc:reset:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def size: Int = 
    this match { 
      case Leaf(value)       => 1
      case Node(left, right) => left.size + right.size
    }

  def contains(element: A): Boolean =
    this match { 
      case Leaf(value)       => element == value
      case Node(left, right) => left.contains(element) || right.contains(element)
    }
    
  def map[B](f: A => B): Tree[B] =
    this match { 
      case Leaf(value)       => Leaf(f(value))
      case Node(left, right) => Node(left.map(f), right.map(f))
    }
}
```
</div>


### 構造的再帰としての畳み込み

Let's finish by looking at the fold method as an abstraction over structural recursion.
If you did the `Tree` exercise above, you will have noticed that we wrote the same kind of code again and again. 
Here are the methods we wrote.
Notice the left-hand sides of the pattern matches are all the same, and the right-hand sides are very similar.

最後に、構造的再帰をさらに抽象化したものとして畳み込みメソッドについて見ていこう。先ほどの `Tree` の演習を行ったなら、同じパターンのコードを何度も書いたことに気付いたはずである。演習で作成したメソッドを以下に再掲する。パターンマッチの左側はすべて同じだし、右側も非常に似ていることに注目してほしい。

```scala
def size: Int = 
  this match { 
    case Leaf(value)       => 1
    case Node(left, right) => left.size + right.size
  }

def contains(element: A): Boolean =
  this match { 
    case Leaf(value)       => element == value
    case Node(left, right) => left.contains(element) || right.contains(element)
  }
  
def map[B](f: A => B): Tree[B] =
  this match { 
    case Leaf(value)       => Leaf(f(value))
    case Node(left, right) => Node(left.map(f), right.map(f))
  }
```

This is the point of structural recursion: to recognize and formalize this similarity.
However, as programmers we might want to abstract over this repetition.
Can we write a method that captures everything that doesn't change in a structural recursion, and allows the caller to pass arguments for everything that does change?
It turns out we can. For any algebraic data type we can define at least one method, called a fold, that captures all the parts of structural recursion that don't change and allows the caller to specify all the problem specific parts.

この類似性を認識し戦略として公式化することが、構造的再帰のポイントである。だが、プログラマとしては、この繰り返しを抽象化したくなるかもしれない。構造的再帰の中で変わらない部分をすべて抽出し、変わる部分については呼び出し側が引数として渡せるようなメソッドを作ることはできるだろうか。これは実は可能である。任意の代数的データ型に対して、そういうメソッドを少なくともひとつは定義できる。畳み込みと呼ばれるそのメソッドが構造的再帰の変わらない部分をすべてキャプチャし、個別のやりたいことに特化した部分を呼び出し側が指定できるようにしてくれる。

Let's see how this is done using the example of `MyList`.
Recall the definition of `MyList` is

どのように定義されるのか `MyList` の例を使って見ていこう。`MyList` の定義を再掲する。

```scala mdoc:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

We know the structural recursion skeleton for `MyList` is

`MyList` で使われる構造的再帰の骨組みは、すでに理解しているとおり、以下のようになる。

```scala
def doSomething[A](list: MyList[A]) =
  list match {
    case Empty()          => ???
    case Pair(head, tail) => ??? doSomething(tail)
  } 
```

Implementing fold for `MyList` means defining a method

`MyList` に畳み込みを実装するとは、次のような `fold` メソッドを定義するということである。

```scala
def fold[A, B](list: MyList[A]): B =
  list match {
    case Empty() => ???
    case Pair(head, tail) => ??? fold(tail)
  }
```

where `B` is the type the caller wants to create. 

ここで `B` は、呼び出し元が生成したい値の型とする。

To complete `fold` we add method parameters for the problem specific (`???`) parts.
In the case for `Empty`, we need a value of type `B` (notice that I'm following the types here).

`fold` メソッドを完成させるには、都度のやりたいことに固有の `???` の部分を埋めるために引数を加える必要がある。`Empty` のケースには `B` 型の値が必要である（型に従って考えていることに注目してほしい）。

```scala
def fold[A, B](list: MyList[A], empty: B): B =
  list match {
    case Empty() => empty
    case Pair(head, tail) => ??? fold(tail, empty)
  }
```

For the `Pair` case, we have the head of type `A` and the recursion producing a value of type `B`. This means we need a function to combine these two values.

`Pair` のケースでは、 `A` 型の `head` と、 `B` 型の値を生成する再帰処理がすでにあるので、必要なのはそれら二つを結合する関数ということになる。

```scala mdoc:invisible
import MyList.*
```
```scala mdoc:silent
def foldRight[A, B](list: MyList[A], empty: B, f: (A, B) => B): B =
  list match {
    case Empty() => empty
    case Pair(head, tail) => f(head, foldRight(tail, empty, f))
  }
```

This is `foldRight` (and I've renamed the method to indicate this).
You might have noticed there is another valid solution.
Both `empty` and the recursion produce values of type `B`.
If we follow the types we can come up with

これが `foldRight` メソッドである（この後に見せるもうひとつの解法と区別できるよう名前を変更した）。これを見て、もうひとつの有効な解があることに気付いたかもしれない。 `empty` も再帰呼び出しも `B` の値を生成する。型に従って考えれば、次のような結論に至ることもできるだろう。

```scala mdoc:silent
def foldLeft[A,B](list: MyList[A], empty: B, f: (A, B) => B): B =
  list match {
    case Empty() => empty
    case Pair(head, tail) => foldLeft(tail, f(head, empty), f)
  }
```

which is `foldLeft`, the tail-recursive variant of fold for a list.
(We'll talk about tail-recursion in a later chapter.)

これが `foldLeft` メソッドで、リストに対する末尾再帰型の畳み込みの変種である。末尾再帰については後の章で解説する。

We can follow the same process for any algebraic data type to create its folds. 
The rules are:

決まった手順をたどることで、任意の代数的データ型に対してその畳み込みメソッドを作成することができる。そのルールは以下の通りである。

- a fold is a function from the algebraic data type and additional parameters to some generic type that I'll call `B` below for simplicity;
- the fold has one additional parameter for each case in a logical or;
- each parameter is a function, with result of type `B` and parameters that have the same type as the corresponding constructor arguments *except* recursive values are replaced with `B`; and
- if the constructor has no arguments (for example, `Empty`) we can use a value of type `B` instead of a function with no arguments.

- 畳み込みは、代数的データ型と追加のパラメータを取り、何らかの汎用型（以下では便宜上 `B` と呼ぶ）に変換する関数である
- 畳み込みには、論理和の各ケースに対してひとつの追加パラメータがある
- 各パラメータは関数で、結果の型は `B`、対応するコンストラクタの引数と同じ型のパラメータを取る。ただし、再帰的な値は `B` に置き換えられる
- コンストラクタに引数がない場合（たとえば `Empty`）、引数のない関数の代わりに、型 `B` の値を使用できる

Returning to `MyList`, it has:

`MyList` に当てはめると次のようになる。

- two cases, and hence two parameters to fold (other than the parameter that is the list itself);
- `Empty` is a constructor with no arguments and hence we use a parameter of type `B`; and
- `Pair` is a constructor with one parameter of type `A` and one recursive parameter, and hence the corresponding function has type `(A, B) => B`.

- 二つのケースがあるので、畳み込みには二つのパラメータが必要（リストそれ自体以外に）
- `Empty` は引数なしコンストラクタなので、`B` 型のパラメータをひとつ
- `Pair` は `A` 型の引数をひとつと、再帰的な引数をひとつもったコンストラクタなので、対応する関数の型は `(A, B) => B` となる

#### 演習: `Tree` の畳み込み {-}

Implement a fold for `Tree` defined earlier.
There are several different ways to traverse a tree (pre-order, post-order, and in-order). 
Just choose whichever seems easiest.

以前定義した `Tree` に対して畳み込みを実装せよ。二分木の走査には、先行順、後行順、中間順などいくつかの方法があるが、最も簡単だと思うものを選んで実装してかまわない。

<div class="solution">
I start by add the method declaration without a body.

まずはボディのないメソッド宣言を追加することから始める。

```scala mdoc:reset:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def fold[B]: B =
    ???
}
```

Next step is to add the structural recursion skeleton.

次に、構造的再帰の骨組みを追加する。

```scala
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def fold[B]: B =
    this match {
      case Leaf(value)       => ???
      case Node(left, right) => left.fold ??? right.fold
    }
}
```

Now I follow the types to add the method parameters.
For the `Leaf` case we want a function of type `A => B`.

これで、型に従ってメソッドにパラメータを追加する準備が整った。`Leaf` のケースには `A => B` 型の関数が必要であることが分かる。

```scala
enum Tree[A] {
  case Leaf(value: A => B)
  case Node(left: Tree[A], right: Tree[A])
  
  def fold[B](leaf: A => B): B =
    this match {
      case Leaf(value)       => leaf(value)
      case Node(left, right) => left.fold ??? right.fold
    }
}
```

For the `Node` case we want a function that combines the two recursive results, and therefore has type `(B, B) => B`.

`Node` のケースには二つの再帰呼び出しの結果を結合する関数が必要なので、追加するパラメータは `(B, B) => B` という型をもつことになる。

```scala mdoc:reset:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def fold[B](leaf: A => B)(node: (B, B) => B): B =
    this match {
      case Leaf(value)       => leaf(value)
      case Node(left, right) => node(left.fold(leaf)(node), right.fold(leaf)(node))
    }
}
```
</div>

#### 演習: 畳み込みの利用 {-}

Prove to yourself that you can replace structural recursion with calls to fold, by redefining `size`, `contains`, and `map` for `Tree` using only fold.

構造的再帰が畳み込みの呼び出しに置き換えられることを確認するため、`Tree` の `size`、`contains`、`map` を、`fold` だけを使って再定義せよ。

<div class="solution">
```scala mdoc:reset:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
  
  def fold[B](leaf: A => B)(node: (B, B) => B): B =
    this match {
      case Leaf(value)       => leaf(value)
      case Node(left, right) => node(left.fold(leaf)(node), right.fold(leaf)(node))
    }
    
  def size: Int = 
    this.fold(_ => 1)(_ + _)

  def contains(element: A): Boolean =
    this.fold(_ == element)(_ || _)
    
  def map[B](f: A => B): Tree[B] =
    this.fold(v => Leaf(f(v)))((l, r) => Node(l, r))
}
```
</div>
