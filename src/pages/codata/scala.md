## Codata in Scala

We have already seen an example of codata, which I have repeated below.

前節で見た余データの例を以下に再掲する。

```scala mdoc:silent
trait Set[A] {
  
  def contains(elt: A): Boolean
  
  def insert(elt: A): Set[A]
  
  def union(that: Set[A]): Set[A]
}
```

The abstract definition of this, which is a product of functions, defines a `Set` with elements of type `A` as:

関数の関数の集まりであるこの抽象的な定義は、`A` 型の要素をもつ `Set` を次のような操作をもつものであると示している。

- `Set[A]` と `A` 型の要素を受け取り、`Boolean` を返す関数 `contains`
- `Set[A]` と `A` 型の要素を受け取り、`Set[A]` を返す関数 `insert`
- `Set[A]` ともうひとつの `Set[A]` を受け取り、`Set[A]` を返す関数 `union`

Notice that the first parameter of each function is the type we are defining, `Set[A]`.

どの関数も最初のパラメータの型は `Set[A]` である点に注目してほしい。

The translation to Scala is:

Scalaに翻訳する際は以下のようになり、

- the overall type becomes a `trait`; and
- each function becomes a method on that `trait`. The first parameter is the hidden `this` parameter, and other parameters become normal parameters to the method.

- 全体の型を `trait` とする
- 各関数をそのトレイトのメソッドとして、最初のパラメータの代わりに `this` を用い、残りをそのメソッドの普通のパラメータとする

This gives us the Scala representation we started with.

これにより、上に再掲したScalaの表現が得られる。

This is only half the story for codata. We also need to actually implement the interface we've just defined. There are three approaches we can use:

以上は余データに関する話の半分に過ぎない。先ほど定義したインターフェースを実際に実装する必要がある。実装のアプローチは三つある。

1. a `final` subclass, in the case where we want to name the implementation;
2. an anonymous subclass; or
3. more rarely, an `object`.

1. `final` サブクラス。これは実装に名前を付けたい場合に用いる
2. 匿名のサブクラス
3. 稀なケースだが、`object`

Neither `final` nor anonymous subclasses can be further extended, meaning we cannot create deep inheritance hierarchies. This in turn avoids the difficulties that come from reasoning about deep hierarchies. Using a `class` rather than a `case class` means we don't expose implementation details like constructor arguments.

`final` サブクラスも匿名サブクラスもそれ以上拡張できないため、深い継承階層を作成することはできない。これにより、深い階層について推論する際に生じる困難を回避できる。また、`case class` ではなく `class` を使用することで、コンストラクタ引数などの実装の詳細を公開せずにすむ。

Some examples are in order. Here's a simple example of `Set`, which uses a `List` to hold the elements in the set.

いくつか例を挙げるのがよいだろう。ここでは、集合の要素を保持する実装として `List` を使用する `Set` の単純な例を示す。

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

この例ではひとつ目の実装アプローチとして紹介した `final` サブクラスを用いている。余データ型を返すメソッドを実装するときに最も便利なのが匿名サブクラスである。`union` メソッドを例に考えてみよう。このメソッドは余データ自体の型である `Set` を返す。これは以下のように実装することができる。


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

この例では、匿名サブクラスを使用し、`Set` トレイト上で `union` メソッドを実装している。この定義はすべてのサブクラスで利用可能である。メソッドを `final` にしていないのは、サブクラスがより効率的な実装でそれをオーバーライドできるようにするためであるが、これにより、実装の継承に伴う危険性が生じる。これは理論と実践が乖離する一例である。理論上、実装継承は好まれないが、実際には最適化として有用である場合がある。

It can also be useful to implement utility methods defined purely in terms of the destructors. Let's say we wanted to implement a method `containsAll` that checks if a `Set` `contains` all the elements in an `Iterable` collection.

デストラクタのみを用いて定義されたユーティリティメソッドを実装することも有用である。たとえば、`Set` が `Iterable` コレクション内のすべての要素を `contains` しているかを確認する `containsAll` メソッドを実装したいとする。

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

ここでも、メソッドを `final` にするという選択肢がある。このケースでは、より効率的な実装を想像するのは難しいため、`final` にすることがより正当化される。

Data and codata are both realized in Scala as variations of the same language features of classes and objects. This means we can define types that have properties of both data and codata. We have actually already done this. When we define data we must define names for the fields within the data, thus defining destructors. This is the same in most languages, which don't make a hard distinction between data and codata. 

データと余データは、Scalaにおいてクラスとオブジェクトという同じ言語機能の変種として実現されている。これは、データと余データ両方の特性をもつ型を定義できることを意味する。実際、我々はすでにこれを行っている。データを定義する際には、データ内のフィールドの名前を定義しなければならず、それによってデストラクタが定義される。これは、データと余データの間に明確な区別を設けていないほとんどの言語と同じである。

<!-- TODO:
「それによってデストラクタが定義される」はおそらくcase classの仕様を前提としている
そのことについて補足的な訳文にするか、訳注をつけるか、したほうがよいかもしれない
-->
Part of the appeal, I think, of classes and objects is that they can express so many conceptually different abstractions with the same language constructs. This gives them a surface appearance of simplicity; it seems we need to learn only one abstraction to solve a huge of number of coding problems. However this apparent simplicity hides real complexity, as this variety of uses forces us to reverse engineer the conceptual intention from the code. 

クラスとオブジェクトの魅力の一部は、同じ言語構造を用いて非常に多くの概念的に異なる抽象を表現できる点にあると考えている。このことは、クラスやオブジェクトを表面的には単純に見せ、ひとつの抽象さえ学べば多くのコーディングに関する問題を解決できるかのように感じさせる。だが、この見かけ上の単純さは実際の複雑さを隠している。同じ言語構造がさまざまな用途に使われるせいで、コードから概念的な意図を逆解析する必要が生じるのである。
