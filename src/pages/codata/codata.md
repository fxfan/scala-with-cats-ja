## データと余データ

Data describes what things are, while codata describes what things can do. 

データはモノが何であるのかを記述し、余データはモノが何をすることができるかを記述する。

We have seen that data is defined in terms of constructors producing elements of the data type. Let's take a very simple example: a `Bool` is either `True` or `False`. We know we can represent this in Scala as

これまで、データは、そのデータ型のバリアントを生成するコンストラクタによって定義されることを見てきた。非常に単純な例として、`Bool` は `True` または `False` のいずれかで、Scalaでは次のように表現できる。

<!-- 
TODO: 代数的データ型の「要素」
要素という表現は多くの意味（コレクションの要素など）で使われるのでわかりにくいが、
この書籍全体を通じて「element」と呼ばれるので、素直に「要素」という訳語で統一したほうがいいのかもしれない。訳注で回避するか？
-->

```scala mdoc:silent
enum Bool {
  case True
  case False
}
```

The definition tells us there are two ways to construct an element of type `Bool`.
Furthermore, if we have such an element we can tell exactly which case it is, by using a pattern match for example. Similarly, if the instances themselves hold data, as in `List` for example, we can always extract all the data within them. Again, we can use pattern matching to achieve this.

この定義は、`Bool` 型の値を構築する方法が二つあることを示している。さらに、そのようなバリアントがあるときに、たとえばパターンマッチを使用することで、正確にどちらのケースであるかを判断することができる。同様に、たとえば `List` のようにインスタンス自体がデータを保持している場合、その中のすべてのデータを取り出すことが常に可能である。ここでも、パターンマッチを使用してこれを実現することができる。

Codata, in contrast, is defined in terms of operations we can perform on the elements of the type. These operations are sometimes called **destructors** (which we've already encountered), **observations**, or **eliminators**. A common example of codata is a data structures such as a set. We might define the operations on a `Set` with elements of type `A` as:

対照的に、余データは、その型の値に対して行える操作によって定義される。これらの操作は、**デストラクタ（destructor）**、**観察（observation）**、または**エリミネータ（eliminator）**と呼ばれることがある。デストラクタは前章にも登場した。余データの一般的な例として、集合のようなデータ構造がある。たとえば、 `A` 型の要素をもつ `Set` の操作は次のように定義できる。

- `contains` which takes a `Set[A]` and an element `A` and returns a `Boolean` indicating if the set contains the element;
- `insert` which takes a `Set[A]` and an element `A` and returns a `Set[A]` containing all the elements from the original set and the new element; and
- `union` which takes a `Set[A]` and a set `Set[A]` and returns a `Set[A]` containing all the elements of both sets.

- `contains`: `Set[A]` と要素 `A` を受け取り、集合にその要素が含まれているかどうかを示す `Boolean` を返す
- `insert`: `Set[A]` と要素 `A` を受け取り、元の集合のすべての要素と新しい要素を含む `Set[A]` を返す
- `union`: `Set[A]` ともうひとつの集合 `Set[A]` を受け取り、両方の集合のすべての要素を含む `Set[A]` を返す

In Scala we could implement this definition as

Scalaではこの定義を次のように記述できる。

```scala mdoc:silent
trait Set[A] {
  
  /** 与えられた要素がこの集合に含まれていればTrue */
  def contains(elt: A): Boolean
  
  /** この集合のすべての要素と与えられた要素を含む新しい集合を構築する */
  def insert(elt: A): Set[A]
  
  /** Construct the union of this and that set */
  /** この集合と指定された集合の和集合を構築する */
  def union(that: Set[A]): Set[A]
}
```

This definition does not tell us anything about the internal representation of the elements in the set. It could use a hash table, a tree, or something more exotic. It does, however, tell us what we can do with the set. We know we can take the union but not the intersection, for example. 

この定義は、集合内の要素が内部的にどのように表現されているかついて何も述べていない。ハッシュテーブルが使われるかもしれないし、木構造やあるいはもっと特殊な何かが使われるかもしれない。だが、集合に対してどういう操作が可能であるかをこの定義は教えてくれる。たとえば、和集合を取ることはできても、積集合をとることはできない、ということは分かるのである。

If you come from the object-oriented world you might recognize the description of codata above as programming to an interface. In some ways codata is just taking concepts from the object-oriented world and presenting them in a way that is consistent with the rest of the functional programming paradigm. However, this does not mean adopting all the features of object-oriented programming. We won't use state, which is difficult to reason about. We also won't use implementation inheritance either, for the same reason. In our subset of object-oriented programming we'll either be defining interfaces (which may have default implementations of some methods) or final classes that implement those interfaces. Interestingly, this subset of object-oriented programming is often recommended by advocates of object-oriented programming.

オブジェクト指向の世界から来た人であれば、前述の余データの説明をインターフェースを利用したプログラミングと認識するかもしれない。ある意味、余データとはオブジェクト指向の世界から概念を取り入れ、それを関数型プログラミングのパラダイムと一貫した形で提示するものと言える。しかし、これはオブジェクト指向プログラミングのすべての機能を採用するという意味ではない。我々は、推論の難しい状態を使用しないし、同じ理由で実装の継承も使わない。我々のオブジェクト指向プログラミングのサブセットでは、インターフェースを定義し（いくつかのメソッドにデフォルト実装を持たせてもよい）、そのインターフェースを実装するfinalクラスを定義する。興味深いことに、このようなオブジェクト指向プログラミングのサブセットは、オブジェクト指向プログラミングの支持者によってもしばしば勧められている。

Let's now be a little more precise in our definition of codata, which will make the duality between data and codata clearer. Remember the definition of data: it is defined in terms of sums (logical ors) and products (logical ands). We can transform any data into a sum of products. Each product in the sum is a constructor, and the product itself is the parameters that the constructor accepts. Finally, we can think of constructors as functions which take some arbitrary input and produce an element of data. Our end point is a sum of functions from arbitrary input to data.

それでは、余データをもうすこし正確に定義していこう。そうすることで、データと余データの双対性がより明確になるはずである。データの定義を思い出してほしい。データは論理和と論理積を使って定義される。我々は、任意のデータを「積の和」として表現することができる。この和の中の各積がコンストラクタであり、積を構成する各要素がコンストラクタの受け取るパラメータとなる。コンストラクタは任意の入力を受け取りデータのバリアントを生成する関数と考えることができ、最終的な形は、任意の入力からデータを生成する関数の直和となる。

More abstractly, if we are constructing an element of some data type `A` we call one of the constructors

より抽象的に言うと、あるデータ型 `A` の値を構築するときは、複数あるコンストラクタのうちのひとつを呼び出す。

- `A1: (B, C, ...) => A`; or
- `A2: (D, E, ...) => A`; or
- `A3: (F, G, ...) => A`; and so on.

- `A1: (B, C, ...) => A` または
- `A2: (D, E, ...) => A` または
- `A3: (F, G, ...) => A` など


Now we'll turn to codata. Codata is defined as a product of functions, these functions being the destructors. The input to a destructor is always an element of the codata type and possibly some other parameters. The output is usually something that is not of the codata type. Thus constructing an element of some codata type `A` means defining

一方、余データは関数の積として定義される。これらの関数はデストラクタである。デストラクタへの入力には常に余データ型が含まれ、場合によって他のパラメータも存在する。出力は通常、余データ型ではない何かになる。したがって、余データ型 `A` の要素を構築するということは、以下のような定義を意味する。

- `A1: (A, B, ...) => C` および
- `A2: (A, D, ...) => E` および
- `A3: (A, F, ...) => G` など

This hopefully makes the duality between the two clearer.
Now we understand what codata is, we will turn to representing codata in Scala.

これで、両者の双対性がより明確になることを期待したい。余データが何であるかは理解できたので、次にScalaにおける余データの表現に移ろう。
