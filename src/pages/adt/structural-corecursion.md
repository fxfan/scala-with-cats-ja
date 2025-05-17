<!--

## Structural Corecursion

Structural corecursion is the opposite—more correctly, the dual—of structural recursion.
Whereas structural recursion tells us how to take apart an algebraic data type, 
structural corecursion tells us how to build up, or construct, an algebraic data type.
Whereas we can use structural recursion whenever the input of a method or function is an algebraic data type,
we can use structural corecursion whenever the output of a method or function is an algebraic data type.


<div class="callout callout-info">
#### Duality in Functional Programming {-}

Two concepts or structures are duals if one can be translated in a one-to-one fashion to the other.
Duality is one of the main themes of this book.
By relating concepts as duals we can transfer knowledge from one domain to another.

Duality is often indicated by attaching the co- prefix to one of the structures or concepts.
For example, corecursion is the dual of recursion, and sum types, also known as coproducts, are the dual of product types.
</div>


Structural recursion works by considering all the possible inputs (which we usually represent as patterns), and then working out what we do with each input case.
Structural corecursion works by considering all the possible outputs, which are the constructors of the algebraic data type, and then working out the conditions under which we'd call each constructor.

Let's return to the list with elements of type `A`, defined as:

- the empty list; or
- a pair containing an `A` and a tail, which is a list of `A`.

In Scala 3 we write

```scala mdoc:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

We can use structural corecursion if we're writing a method that produces a `MyList`.
A good example is `map`:

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    ???
}
```

The output of this method is a `MyList`, which is an algebraic data type.
Since we need to construct a `MyList` we can use structural corecursion.
The structural corecursion strategy says we write down all the constructors and then consider the conditions that will cause us to call each constructor.
So our starting point is to just write down the two constructors, and put in dummy conditions.

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    if ??? then Empty()
    else Pair(???, ???)
}
```

We can also apply the recursion rule: where the data is recursive so is the method.

```scala
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    if ??? then Empty()
    else Pair(???, ???.map(f))
}
```

To complete the left-hand side we can use the strategies we've already seen:

* we can use structural recursion to tell us there are two possible conditions; and
* we can follow the types to align these conditions with the code we have already written.

In short order we arrive at the correct solution

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

There are a few interesting points here. 
Firstly, we should acknowledge that `map` is both a structural recursion and a structural corecursion.
This is not always the case.
For example, `foldLeft` and `foldRight` are not structural corecursions because they are not constrained to only produce an algebraic data type.
Secondly, note that when we walked through the process of creating `map` as a structural recursion we implicitly used the structural corecursion pattern, as part of following the types.
We recognised that we were producing a `List`, that there were two possibilities for producing a `List`, and then worked out the correct conditions for each case.
Formalizing structural corecursion as a separate strategy allows us to be more conscious of where we apply it.
Finally, notice how I switched from an `if` expression to a pattern match expression as we progressed through defining `map`.
This is perfectly fine.
Both kinds of expression achieve the same effect.
Pattern matching is a little bit safer due to exhaustivity checking.
If we wanted to continue using an `if` we'd have to define a method (for example, `isEmpty`) that allows us to distinguish an `Empty` element from a `Pair`.
This method would have to use pattern matching in its implementation, so avoiding pattern matching directly is just pushing it elsewhere.


### Unfolds as Structural Corecursion

Just as we could abstract structural recursion as a fold, for any given algebraic data type we can abstract structural corecursion as an unfold. Unfolds are much less commonly used than folds, but they are still a nice tool to have.

Let's work through the process of deriving unfold, using `MyList` as our example again.

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

The corecursion skeleton is

```scala
if ??? then MyList.Empty()
else MyList.Pair(???, recursion(???))
```

Our starting point is writing the skeleton for `unfold`. It's a little bit unusual in that I've added a parameter `seed`. This is the information we use to create an element. We'll need this, but we cannot derive it from our strategies, so I've added it in here as a starting assumption.

```scala
def unfold[A, B](seed: A): MyList[B] =
  ???
```

Now we start using our strategies to fill in the missing pieces. I'm using the corecursion skeleton and I've applied the recursion rule immediately in the code below, to save a bit of time in the derivation.

```scala
def unfold[A, B](seed: A): MyList[B] =
  if ??? then MyList.Empty()
  else MyList.Pair(???, unfold(seed))
```

We can abstract the condition using a function from `A => Boolean`.

```scala
def unfold[A, B](seed: A, stop: A => Boolean): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(???, unfold(seed, stop))
```

Now we need to handle the case for `Pair`. 
We have a value of type `A` (`seed`), so to create the `head` element of `Pair` we can ask for a function `A => B`

```scala
def unfold[A, B](seed: A, stop: A => Boolean, f: A => B): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(f(seed), unfold(???, stop, f))
```

Finally we need to update the current value of `seed` to the next value. That's a function `A => A`.

```scala mdoc:silent
def unfold[A, B](seed: A, stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(f(seed), unfold(next(seed), stop, f, next))
```

At this point we're done.
Let's see that `unfold` is useful by declaring some other methods in terms of it.
We're going to declare `map`, which we've already seen is a structural corecursion, using `unfold`.
We will also define `fill` and `iterate`, which are methods that construct lists and correspond to the methods with the same names on `List` in the Scala standard library.

To make this easier to work with I'm going to declare `unfold` as a method on the `MyList` companion object. 
I have made a slight tweak to the definition to make type inference work a bit better.
In Scala, types inferred for one method parameter cannot be used for other method parameters in the same parameter list.
However, types inferred for one method parameter list can be used in subsequent lists.
Separating the function parameters from the `seed` parameter means that the value inferred for `A` from `seed` can be used for inference of the function parameters' input parameters.

I have also declared some **destructor** methods, which are methods that take apart an algebraic data type.
For `MyList` these are `head`, `tail`, and the predicate `isEmpty`.
We'll talk more about these a bit later.

Here's our starting point.

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(_head: A, _tail: MyList[A])

  def isEmpty: Boolean =
    this match {
      case Empty() => true
      case _       => false
    }
    
  def head: A =
    this match {
      case Pair(head, _) => head
    }
    
  def tail: MyList[A] =
    this match {
      case Pair(_, tail) => tail
    }
}
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
}
```

Now let's define the constructors `fill` and `iterate`, and `map`, in terms of `unfold`. I think the constructors are a bit simpler, so I'll do those first.

```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    ???
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    ???
}
```

Here I've just added the method skeletons, which are taken straight from the `List` documentation.
To implement these methods we can use one of two strategies:

- reasoning about loops in the way we might in an imperative language; or
- reasoning about structural recursion over the natural numbers.

Let's talk about each in turn.

You might have noticed that the parameters to `unfold` are almost exactly those you need to create a for-loop in a language like Java. A classic for-loop, of the `for(i = 0; i < n; i++)` kind, has four components:

1. the initial value of the loop counter;
2. the stopping condition of the loop; 
3. the statement that advances the counter; and
4. the body of the loop that uses the counter.

These correspond to the `seed`, `stop`, `next`, and `f` parameters of `unfold` respectively.

Loop variants and invariants are the standard way of reasoning about imperative loops. I'm not going to describe them here, as you have probably already learned how to reason about loops (though perhaps not using these terms). Instead I'm going to discuss the second reasoning strategy, which relates writing `unfold` to something we've already discussed: structural recursion.

Our first step is to note that natural numbers (the integers 0 and larger) are conceptually algebraic data types even though the implementation in Scala---using `Int`---is not. A natural number is either:

- zero; or
- 1 + a natural number.

It's the simplest possible algebraic data type that is both a sum and a product type.

Once we see this, we can use the reasoning tools for structural recursion for creating the parameters to `unfold`.
Let's show how this works with `fill`. The `n` parameter tells us how many elements there are in the `List` we're creating. The `elem` parameter creates those elements, and is called once for each element. So our starting point is to consider this as a structural recursion over the natural numbers. We can take `n` as `seed`, and `stop` as the function `x => x == 0`. These are the standard conditions for a structural recursion over the natural numbers. What about `next`? Well, the definition of natural numbers tells us we should subtract one in the recursive case, so `next` becomes `x => x - 1`. We only need `f`, and that comes from the definition of how `fill` is supposed to work. We create the value from `elem`, so `f` is just `_ => elem`

```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0, _ => elem, _ - 1)
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    ???
}
```

```scala mdoc:reset:invisible
enum MyList[A] {
  case Empty()
  case Pair(_head: A, _tail: MyList[A])

  def isEmpty: Boolean =
    this match {
      case Empty() => true
      case _       => false
    }
    
  def head: A =
    this match {
      case Pair(head, _) => head
    }
    
  def tail: MyList[A] =
    this match {
      case Pair(_, tail) => tail
    }
    
  def map[B](f: A => B): MyList[B] =
    MyList.unfold(this)(
      _.isEmpty,
      pair => f(pair.head),
      pair => pair.tail
    )
    
  override def toString(): String = {
    def loop(list: MyList[A]): List[A] =
      list match {
        case Empty() => List.empty
        case Pair(h, t) => h :: loop(t)
      }
    s"MyList(${loop(this).mkString(", ")})"
  }
}
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))

  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0, _ => elem, _ - 1)

  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    unfold((len, start))(
      (len, _) => len == 0,
      (_, start) => start,
      (len, start) => (len - 1, f(start))
    )
}
```

We should check that our implementation works as intended. We can do this by comparing it to `List.fill`.

```scala mdoc:to-string
List.fill(5)(1)
MyList.fill(5)(1)
```

Here's a slightly more complex example, using a stateful method to create a list of ascending numbers.
First we define the state and method that uses it.

```scala mdoc:silent
var counter = 0
def getAndInc(): Int = {
  val temp = counter
  counter = counter + 1
  temp 
}
```

Now we can create it to create lists.

```scala mdoc:to-string
List.fill(5)(getAndInc())
counter = 0
MyList.fill(5)(getAndInc())
```


#### Exercise: Iterate {-}

Implement `iterate` using the same reasoning as we did for `fill`.
This is slightly more complex than `fill` as we need to keep two bits of information: the value of the counter and the value of type `A`.

<div class="solution">
```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0)(_ => elem, _ - 1)
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    unfold((len, start)){
      (len, _) => len == 0,
      (_, start) => start,
      (len, start) => (len - 1, f(start))
    }
}
```

We should check that this works.

```scala mdoc:to-string
List.iterate(0, 5)(x => x - 1)
MyList.iterate(0, 5)(x => x - 1)
```
</div>


#### Exercise: Map {-}

Once you've completed `iterate`, try to implement `map` in terms of `unfold`. You'll need to use the destructors to implement it.

<div class="solution">
```scala
def map[B](f: A => B): MyList[B] =
  MyList.unfold(this)(
    _.isEmpty,
    pair => f(pair.head),
    pair => pair.tail
  )
```

```scala mdoc:to-string
List.iterate(0, 5)(x => x + 1).map(x => x * 2)
MyList.iterate(0, 5)(x => x + 1).map(x => x * 2)
```
</div>

Now a quick discussion on destructors. The destructors do two things:

1. distinguish the different cases within a sum type; and
2. extract elements from each product type.

So for `MyList` the minimal set of destructors is `isEmpty`, which distinguishes `Empty` from `Pair`, and `head` and `tail`.
The extractors are partial functions in the conceptual, not Scala, sense; they are only defined for a particular product type and throw an exception if used on a different case. You may have also noticed that the functions we passed to `fill` are exactly the destructors for natural numbers.

The destructors are another part of the duality between structural recursion and corecursion. Structural recursion is:

- defined by pattern matching on the constructors; and
- takes apart an algebraic data type into smaller pieces.

Structural corecursion instead is:

- defined by conditions on the input, which may use destructors; and
- build up an algebraic data type from smaller pieces.

One last thing before we leave `unfold`. If we look at the usual definition of `unfold` we'll probably find the following definition.

```scala
def unfold[A, B](in: A)(f: A => Option[(A, B)]): List[B]
```

This is equivalent to the definition we used, but a bit more compact in terms of the interface it presents. We used a more explicit definition that makes the structure of the method clearer.


```scala mdoc:reset:silent
```
--->

## 構造的余再帰

構造的余再帰（structural corecursion）は、構造的再帰の反対、より正確な用語を使うなら双対（dual）である。構造的再帰が代数的データ型をどのように分解するかを教えてくれるのに対して、構造的余再帰は代数的データ型をどのように構築するかを教えてくれる。構造的再帰は、メソッドや関数の入力が代数的データ型である場合に使用できるが、構造的余再帰はメソッドや関数の出力が代数的データ型である場合に使用できる。

<div class="callout callout-info">
#### 関数型プログラミングにおける双対性 {-}

ふたつの概念や構造が一対一で対応付けられる場合、それらを双対であると言う。双対性は本書の主要なテーマのひとつである。概念同士を双対として関連付けることで、ある領域の知識を別の領域に流用することができるようになる。

双対性は、しばしば構造や概念に co- という接頭辞を付けることで示される。たとえば、英語で corecursion と呼ばれる余再帰は再帰の双対であり、coproduct （余積）は積の双対である。
</div>

構造的再帰は、考えられるすべての入力（通常はパターンマッチングのパターンとして表現される）について、それぞれの入力ケースに対して何を行うかを決定することによって機能する。一方、構造的余再帰は、考えられるすべての出力、すなわち代数的データ型のコンストラクタについて考え、それぞれのコンストラクタを呼び出す条件を決定することによって機能する。

`A` 型の要素を持つリストの話に戻ろう。これは次のように定義することができた。

- リストは空であるか、もしくは
- ひとつの `A` 型の値と、それ自体が `A` のリストである残りの部分のペア

Scala3 では次のように書かれる。

```scala mdoc:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

`MyList` インスタンスを生成するメソッドを書きたいときに、構造的余再帰を利用することができる。そのよい例として `map` がある。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    ???
}
```

このメソッドの出力は代数的データ型である `MyList` である。`MyList` インスタンスを構築したいのだから、構造的余再帰を利用することができる。構造的余再帰の戦略では、まずすべてのコンストラクタを書き出し、それぞれのコンストラクタ呼び出しを引き起こす条件を考える。したがって、最初のステップは、ふたつのコンストラクタを書き出し、仮の条件を記述することである。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    if ??? then Empty()
    else Pair(???, ???)
}
```

データが再帰的である場所で再帰呼び出しを行うというルールを適用すると、メソッドは以下のようになる。

```scala
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
  
  def map[B](f: A => B): MyList[B] = 
    if ??? then Empty()
    else Pair(???, ???.map(f))
}
```

これまでに学んだ戦略を利用することで、`if` 式の条件部、あるいはパターンマッチングであればその `case` 部分を完成させることができる。

* 構造的再帰を使って、考えうる条件がふたつあることを確認する
* 型に従って、これらの条件を既に書いたコードに合わせる

この手順に従えば、短時間で正しい解決策にたどり着くことができる。

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

ここには興味深いポイントがいくつかある。

まず、`map` は構造的再帰であり、同時に構造的余再帰でもあるということを認識しておく必要がある。これはいつも成り立つわけではない。たとえば、`foldLeft` と `foldRight` は、必ずしも代数的データ型を生成するとは限らないし、構造的余再帰ではない。

次に、`map` を構造的再帰として実装するプロセスを進めた際に、型に従う一環として、それと知らずに構造的余再帰のパターンを使用していたことに注目してほしい。我々は `List` を生成しようとしていることに気づき、`List` を生成する方法としてふたつのコンストラクタの存在を認識し、それぞれに適した条件を見出した。構造的余再帰をひとつの戦略として公式化することで、それを適用しているということを、より意識的に認識できるようになる。

最後に、`map` を定義していく過程で、`if` 式からパターンマッチ式に切り替えたことに留意してほしい。これはまったく問題ない。どちらの式でも同じ効果を得ることができる。ただし、パターンマッチングは網羅性チェックがあるのですこし安全である。また、もし `if` 式を使い続けるのであれば、`Empty` と `Pair` を区別するためのメソッド（たとえば `isEmpty`）を定義しなくてはならないが、そのメソッドの実装にあたってパターンマッチングを使用する必要があるため、パターンマッチングの直接的な利用を避けても、それを別の場所に押し込めてしまうだけである。

### 構造的余再帰としての展開（unfold）

構造的再帰を `fold` として抽象化できたように、任意の代数的データ型に対して、構造的余再帰を `unfold` として抽象化することができる。`unfold` は `fold` ほど一般的に使われるものではないが、有用なツールである。

今回も `MyList` を例として使いながら、`unfold` を導き出すプロセスを見ていこう。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(head: A, tail: MyList[A])
}
```

構造的余再帰の骨組みは以下のとおりだった。

```scala
if ??? then MyList.Empty()
else MyList.Pair(???, recursion(???))
```

まず `unfold` の骨格を書き出すことから始める。すこし変わっているのは、`seed` というパラメータを追加していることである。これはデータを作成するために必要なものだが、戦略から導き出せるものではないため、初期の仮定としてここに追加されている。

```scala
def unfold[A, B](seed: A): MyList[B] =
  ???
```

ここからは、いつもの戦略を使って足りないピースを埋めていく。以下のコードでは、余再帰の骨格を用いるとともに、導出のステップを省くため、再帰呼び出しのルールの適用まで行っている。

```scala
def unfold[A, B](seed: A): MyList[B] =
  if ??? then MyList.Empty()
  else MyList.Pair(???, unfold(seed))
```

条件部は `A => Boolean` 型の関数を使って抽象化できる。

```scala
def unfold[A, B](seed: A, stop: A => Boolean): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(???, unfold(seed, stop))
```

次に必要なのは `Pair` を生成するケースのハンドリングである。 `A` 型の値として `seed` がすでにあるので、`Pair` の `head` 要素を作成するには `A => B` 型の関数があればよいだろう。

```scala
def unfold[A, B](seed: A, stop: A => Boolean, f: A => B): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(f(seed), unfold(???, stop, f))
```

最後に、現在の `seed` 値から、次の再帰呼び出しで使う値を導かなくてはならない。そのために `A => A` 型の関数が必要となる。

```scala mdoc:silent
def unfold[A, B](seed: A, stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
  if stop(seed) then MyList.Empty()
  else MyList.Pair(f(seed), unfold(next(seed), stop, f, next))
```

これで `unfold` の定義は完成である。`unfold` を使って他のメソッドを定義し、その便利さを味わってみよう。すでに見たように、`map` は構造的余再帰なので、これを `unfold` で定義しなおす。さらに、リストを構築するメソッドである `fill` と `iterate` も定義することにする。これらは、Scala 標準ライブラリの `List` にある同名のメソッドに対応している。

以下に示すコードでは、作業を簡単にするため、`unfold` を `MyList` のコンパニオンオブジェクトのメソッドとして宣言している。また、型推論を改善するために定義にすこし調整を加えている。Scala では、あるメソッドパラメータについて推論された型を、同じパラメータリスト内にある他のパラメータの型推論で使うことはできない。だが、異なるパラメータリストにおいては、先行するパラメータリストで推論された型を後続のパラメータリストで利用することができる。関数パラメータを `seed` とは別のパラメータリストに分けることで、`seed` から推論された `A` の型を、関数パラメータの入力型の推論に使用できるようにしている。

さらに、代数的データ型を分解するメソッドである**デストラクタ（destructor）**もいくつか宣言した。今回の `MyList` では、`head`、`tail`、および述語 `isEmpty` がデストラクタである。これらについては後ほど詳しく説明する。

<!--
訳注: オブジェクト指向言語の文脈で使う「デストラクタ」と異なり、分解関数とでも言うべきものであること、余データの章を読めばこの用語のもつ含意がつかめるだろうことを述べる
-->

以下のコードから始めよう。

```scala mdoc:reset:silent
enum MyList[A] {
  case Empty()
  case Pair(_head: A, _tail: MyList[A])

  def isEmpty: Boolean =
    this match {
      case Empty() => true
      case _       => false
    }
    
  def head: A =
    this match {
      case Pair(head, _) => head
    }
    
  def tail: MyList[A] =
    this match {
      case Pair(_, tail) => tail
    }
}
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
}
```

まずはリストのコンストラクタである `fill` と `iterate` を、そしてその後に `map` を、`unfold` を使って定義する。コンストラクタのほうがシンプルなので、まずはそちらから取り組もうという意図である。

```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    ???
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    ???
}
```

ここでは、`List` のドキュメントからそのまま取ってきたメソッドの骨格だけを追加している。これらのメソッドを実装するにあたっては、次のふたつの戦略のいずれかが利用可能である。

- 命令型言語における方法でループについて考えること
- 自然数の構造的再帰について考えること

順に見ていこう。

`unfold` のパラメータが、Java などの言語で for ループを作る際に必要なものとほぼ一致していることに気づいた人もいるかもしれない。典型的な `for(i = 0; i < n; i++)` 形式の for ループには、次の4つの要素がある。

1. ループカウンタの初期値
2. ループの停止条件
3. カウンタを進めるためのステートメント
4. カウンタを使用するループの本体

これらはそれぞれ `unfold` のパラメータである `seed`、`stop`、`next`、`f` に対応している。

ループの変化条件と不変条件について意識することは、命令型ループが何を行っているのかを把握するための標準的な方法である。ループの処理内容を読み解く方法についてはすでに知っているものとし（ここで使っている用語には馴染みがないかもしれないが）、ここでは詳しく説明しない。代わりに、ふたつ目の推論戦略について説明する。それは、`unfold` の書き方を、我々がすでに学んだ構造的再帰と関連づける方法である。

最初のステップは、自然数（ここでは0以上の整数と定義する）が概念的には代数的データ型であるという点に着目することである。もっとも、Scala においては自然数は `Int` を使って表現されるし、代数的データ型とは言えないが。あるひとつの自然数は次のいずれかである。

- ゼロ
- ある自然数に1を足した数

これは直和型と直積型の両方からなるもっともシンプルな代数的データ型である。

これを理解すると、構造的再帰の推論ツールを使って、`unfold` に渡すパラメータを作成することができる。どういうことか `fill` の例で説明しよう。パラメータ `n` は作成するリストの要素数を示している。パラメータ `elem` はリストの要素を生成するため、各要素に対して一度ずつ呼び出される。ここでは、これを自然数に対する構造的再帰として考える。`n` を `seed` とし、`stop` には `x => x == 0` という関数を渡す。これらは自然数に対する構造的再帰における標準的な条件である。`next` はどうすればよいだろうか。自然数の定義に従い再帰的に1を引けばよいので、`next` は `x => x - 1` となる。最後に `f` だが、これは `fill` がどのように動作すべきかという定義から導かれる。`fill` は `elem` を使って値を作成するので、`f` はシンプルに `_ => elem` となる。

```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0, _ => elem, _ - 1)
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    ???
}
```

```scala mdoc:reset:invisible
enum MyList[A] {
  case Empty()
  case Pair(_head: A, _tail: MyList[A])

  def isEmpty: Boolean =
    this match {
      case Empty() => true
      case _       => false
    }
    
  def head: A =
    this match {
      case Pair(head, _) => head
    }
    
  def tail: MyList[A] =
    this match {
      case Pair(_, tail) => tail
    }
    
  def map[B](f: A => B): MyList[B] =
    MyList.unfold(this)(
      _.isEmpty,
      pair => f(pair.head),
      pair => pair.tail
    )
    
  override def toString(): String = {
    def loop(list: MyList[A]): List[A] =
      list match {
        case Empty() => List.empty
        case Pair(h, t) => h :: loop(t)
      }
    s"MyList(${loop(this).mkString(", ")})"
  }
}
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))

  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0, _ => elem, _ - 1)

  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    unfold((len, start))(
      (len, _) => len == 0,
      (_, start) => start,
      (len, start) => (len - 1, f(start))
    )
}
```

この実装が意図どおりに動作するか確認しておこう。確認には `List.fill` と比較すればよい。

```scala mdoc:to-string
List.fill(5)(1)
MyList.fill(5)(1)
```

次に示すのは、状態をもつ関数を `elem` として用いて昇順の数値リストを作成する、すこし複雑な確認例である。まず状態を定義し、それを使用する関数を定義する。

```scala mdoc:silent
var counter = 0
def getAndInc(): Int = {
  val temp = counter
  counter = counter + 1
  temp 
}
```

これを `fill` メソッドに渡せば、昇順の数値リストが得られるはずである。

```scala mdoc:to-string
List.fill(5)(getAndInc())
counter = 0
MyList.fill(5)(getAndInc())
```


#### 演習: `MyList` への `iterate` の実装 {-}

`fill` の時と同じ考え方を使って `iterate` を実装せよ。`iterate` は、カウンタの値と型 `A` の値というふたつの情報を保持する必要があるため、`fill` よりもすこし複雑になる。
 
<div class="solution">
```scala
object MyList {
  def unfold[A, B](seed: A)(stop: A => Boolean, f: A => B, next: A => A): MyList[B] =
    if stop(seed) then MyList.Empty()
    else MyList.Pair(f(seed), unfold(next(seed))(stop, f, next))
    
  def fill[A](n: Int)(elem: => A): MyList[A] =
    unfold(n)(_ == 0)(_ => elem, _ - 1)
    
  def iterate[A](start: A, len: Int)(f: A => A): MyList[A] =
    unfold((len, start)){
      (len, _) => len == 0,
      (_, start) => start,
      (len, start) => (len - 1, f(start))
    }
}
```

解答は以上である。これも動作確認をしておこう。

```scala mdoc:to-string
List.iterate(0, 5)(x => x - 1)
MyList.iterate(0, 5)(x => x - 1)
```
</div>


#### 演習: `MyList` への `map` の実装 {-}

`map` を `unfold` を用いて再実装せよ。なお、これを実装するにはデストラクタを利用する必要がある。

<div class="solution">
```scala
def map[B](f: A => B): MyList[B] =
  MyList.unfold(this)(
    _.isEmpty,
    pair => f(pair.head),
    pair => pair.tail
  )
```

```scala mdoc:to-string
List.iterate(0, 5)(x => x + 1).map(x => x * 2)
MyList.iterate(0, 5)(x => x + 1).map(x => x * 2)
```
</div>

ここで、デストラクタについて簡単に見ておこう。デストラクタはふたつのことを行う。

1. 直和型のバリアントを区別する
2. 区別したそれぞれの直積型からプロパティを取り出す

`MyList` の場合、最小限のデストラクタは、`Pair` から `Empty` を区別する `isEmpty` と、そして `head` と `tail` である。プロパティを取り出すメソッド（extractor）は部分関数になる。これは Scala とは無関係な概念上の話である。それらは特定の直積型に対してだけ定義されており、それ以外のケースで用いると例外をスローする。先ほど `fill` に渡した関数が、まさに自然数に対するデストラクタであったことに気付いた人もいるかもしれない。

デストラクタは構造的再帰と構造的余再帰の間にある双対性のひとつの側面である。構造的再帰がやることは以下のとおりだが、

- コンストラクタを区別するパターンマッチングによって定義され
- 代数的データ型を小さな部分に分解する

一方、構造的余再帰は以下のことを行う。

- デストラクタを用いて記述される、入力に対する条件によって定義され
- 小さな部分から代数的データ型の値を組み立てる

`unfold` の話を終える前にもうひとつだけ述べておきたい。一般的な `unfold` の定義を見ると、おそらく以下のような定義が見つかるだろう。

```scala
def unfold[A, B](in: A)(f: A => Option[(A, B)]): List[B]
```

これは、我々が用いた定義と等価だが、インターフェースはややコンパクトである。我々の定義は、メソッドの構造を明確にするために、より明示的なものとなっている。
