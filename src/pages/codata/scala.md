<!--

## Codata in Scala

We have already seen an example of codata, which I have repeated below.

```scala mdoc:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A]
}
```

The abstract definition of this, which is a product of functions, defines a `Set` with elements of type `A` as:

- a function `contains` taking a `Set[A]` and an element `A` and returning a `Boolean`,
- a function `insert` taking a `Set[A]` and an element `A` and returning a `Set[A]`, and
- a function `union` taking a `Set[A]` and a set `Set[A]` and returning a `Set[A]`.

Notice that the first parameter of each function is the type we are defining, `Set[A]`.

The translation to Scala is:

- the overall type becomes a `trait`; and
- each function becomes a method on that `trait`. The first parameter is the hidden `this` parameter, and other parameters become normal parameters to the method.

This gives us the Scala representation we started with.

This is only half the story for codata. We also need to actually implement the interface we've just defined. There are three approaches we can use:

1. a `final` subclass, in the case where we want to name the implementation;
2. an anonymous subclass; or
3. more rarely, an `object`.

Neither `final` nor anonymous subclasses can be further extended, meaning we cannot create deep inheritance hierarchies. This in turn avoids the difficulties that come from reasoning about deep hierarchies. Using a `class` rather than a `case class` means we don't expose implementation details like constructor arguments.

Some examples are in order. Here's a simple example of `Set`, which uses a `List` to hold the elements in the set.

```scala mdoc:silent
final class ListSet[A](elements: List[A]) extends Set[A] {

  def contains(elt: A): Boolean =
    elements.contains(elt)

  def insert(elt: A): Set[A] =
    ListSet(elt :: elements)

  def union(that: Set[A]): Set[A] =
    elements.foldLeft(that) { (set, elt) => set.insert(elt) }
}
object ListSet {
  def empty[A]: Set[A] = ListSet(List.empty)
}
```

This uses the first implementation approach, a `final` subclass. Where would we use an anonymous subclass? They are most useful when implementing methods that return our codata type. Let's take `union` as an example. It returns our codata type, `Set`, and we could implement it as shown below.

```scala mdoc:reset:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A] = {
    val self = this
    new Set[A] {
      def contains(elt: A): Boolean =
        self.contains(elt) || that.contains(elt)
        
      def insert(elt: A): Set[A] =
        // Arbitrary choice to insert into self
        self.insert(elt).union(that)
    }
  }
}
```

This uses an anonymous subclass to implement `union` on the `Set` trait, and hence defines the method for all subclasses. I haven't made the method `final` so that subclasses can override it with a more efficient implementation. This does open up the danger of implementation inheritance. This is an example of where theory and craft diverge. In theory we never want implementation inheritance, but in practice it can be useful as an optimization.

It can also be useful to implement utility methods defined purely in terms of the destructors. Let's say we wanted to implement a method `containsAll` that checks if a `Set` `contains` all the elements in an `Iterable` collection.

```scala
def containsAll(elements: Iterable[A]): Boolean
```

We can implement this purely in terms of `contains` on `Set` and `forall` on `Iterable`.

```scala mdoc:reset:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A]
  
  def containsAll(elements: Iterable[A]): Boolean =
    elements.forall(elt => this.contains(elt))
}
```

Once again we could make this a `final` method. In this case it's probably more justified as it's difficult to imagine a more efficient implementation.

Data and codata are both realized in Scala as variations of the same language features of classes and objects. This means we can define types that have properties of both data and codata. We have actually already done this. When we define data we must define names for the fields within the data, thus defining destructors. This is the same in most languages, which don't make a hard distinction between data and codata. 

Part of the appeal, I think, of classes and objects is that they can express so many conceptually different abstractions with the same language constructs. This gives them a surface appearance of simplicity; it seems we need to learn only one abstraction to solve a huge of number of coding problems. However this apparent simplicity hides real complexity, as this variety of uses forces us to reverse engineer the conceptual intention from the code. 


```scala mdoc:reset:silent
```
--->

## Scala における余データ

前節で見た余データの例を以下に再掲する。

```scala mdoc:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A]
}
```

関数の積として表されるこの抽象的な定義は、`A` 型の要素をもつ `Set` を次のような操作をもつものであると定義している。

- `Set[A]` と `A` 型の要素を受け取り、`Boolean` を返す関数 `contains`
- `Set[A]` と `A` 型の要素を受け取り、`Set[A]` を返す関数 `insert`
- `Set[A]` ともうひとつの `Set[A]` を受け取り、`Set[A]` を返す関数 `union`

どの関数も最初のパラメータの型は `Set[A]` である点に注目してほしい。

Scala に翻訳する方法は以下のとおりで、

- 全体の型を `trait` とする
- 各関数をそのトレイトのメソッドとして、最初のパラメータの代わりに `this` を用い、残りをそのメソッドの普通のパラメータとする

これにより、本節の最初に見せた Scala の表現が得られる。

以上は余データに関する話の半分に過ぎない。続いて、先ほど定義したインターフェースを実際に実装する必要がある。実装のアプローチは三つある。

1. `final` サブクラス。これは実装に名前を付けたい場合に用いる
2. 匿名のサブクラス
3. 稀なケースだが、`object`

`final` なサブクラスも匿名サブクラスもそれ以上拡張できないため、深い継承階層を作成することはできない。これにより、深い継承階層をもったコードを理解する際に生じる困難を回避できる。また、`case class` ではなく `class` を使用することで、コンストラクタ引数などの実装の詳細を公開せずにすむ。

いくつか例を挙げよう。ここでは、`List` を使って集合の要素を保持する `Set` 実装の単純な例を示す。

```scala mdoc:silent
final class ListSet[A](elements: List[A]) extends Set[A] {

  def contains(elt: A): Boolean =
    elements.contains(elt)

  def insert(elt: A): Set[A] =
    ListSet(elt :: elements)

  def union(that: Set[A]): Set[A] =
    elements.foldLeft(that) { (set, elt) => set.insert(elt) }
}
object ListSet {
  def empty[A]: Set[A] = ListSet(List.empty)
}
```

この例ではひとつ目の実装アプローチとして紹介した `final` サブクラスを用いている。匿名サブクラスは、余データ型を返すメソッドを実装するときにもっとも便利である。`union` メソッドを例に考えてみよう。このメソッドは余データ自体の型である `Set` を返す。これは以下のように実装することができる。


```scala mdoc:reset:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A] = {
    val self = this
    new Set[A] {
      def contains(elt: A): Boolean =
        self.contains(elt) || that.contains(elt)
        
      def insert(elt: A): Set[A] =
        // self と that どちらに insert してもよい
        self.insert(elt).union(that)
    }
  }
}
```

この例では、匿名サブクラスを使用して `Set` トレイトの `union` を実装している。この定義はすべてのサブクラスから利用できる。メソッドを `final` にしていないのは、サブクラスがもっと効率的な実装でそれをオーバーライドできるようにするためである。これにより、実装継承に伴う危険性が生じる。これは、実践がいつも原則どおりに行われるわけではないことを示す例である。原則的には実装継承は好まれないが、実際には最適化として有用である場合がある。

デストラクタのみを用いて定義されたユーティリティメソッドを実装することも有用である。たとえば、`Iterable` コレクション内のすべての要素を `contains` しているかを確認する `containsAll` メソッドを `Set` に実装したいとする。

```scala
def containsAll(elements: Iterable[A]): Boolean
```

このメソッドは `Set` の `containts` と `Iterable` の `forall` だけを使って実装できる。

```scala mdoc:reset:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A]
  
  def containsAll(elements: Iterable[A]): Boolean =
    elements.forall(elt => this.contains(elt))
}
```

ここでも、メソッドを `final` にするという選択肢がある。このケースでは、より効率的な実装を想像するのは難しいため、`final` にすることは先ほどの例より正当であると言えるだろう。

データと余データは、Scala においては、クラスとオブジェクトというひとつの言語機能の別の側面として実現されている。これは、データと余データ両方の特性をもつ型を定義できることを意味する。実際、我々はすでにこれを行っている。データを定義する際には、データ内のフィールドの名前を定義しなければならず、それによってデストラクタが定義される。これは、データと余データの間に明確な区別を設けていないほとんどの言語と同じである。

クラスとオブジェクトの魅力の一端は、ひとつの言語構造を用いて非常に多くの概念的に異なる抽象を表現できる点にあると考えている。このことは、クラスやオブジェクトを表面的には単純に見せ、ひとつの抽象さえ学べば多くのコーディングに関する問題を解決できるかのように感じさせる。だが、この見かけ上の単純さは実際の複雑さを隠している。同じ言語構造がさまざまな用途に使われるせいで、コードから概念的な意図を逆解析する必要が生じるのである。
