<!--

## Data and Codata Extensibility {#sec:codata:extensibility}

We have seen that codata can represent types with an infinite number of elements, such as `Stream`. This is one expressive difference from data, which must always be finite. We'll now look at another, which is the type of extensibility we get from data and from codata. Together these gives use guidelines to choose between the two.

Firstly, let's define extensibility. It means the ability to add new features without modifying existing code. (If we allow modification of existing code then any extension becomes trivial.) In particular there are two dimensions along which we can extend code: adding new functions or adding new elements. We will see that data and codata have orthogonal extensibility: it's easy to add new functions to data but adding new elements is impossible without modifying existing code, while adding new elements to codata is straight-forward but adding new functions is not.

Let's start with a concrete example of both data and codata. For data we'll use the familiar `List` type.

```scala mdoc:silent
enum List[A] {
  case Empty()
  case Pair(head: A, tail: List[A])
}
```

For codata, we'll use `Set` as our exemplar.

```scala mdoc:silent
trait Set[A] {
  def contains(elt: A): Boolean
  def insert(elt: A): Set[A]
  def union(that: Set[A]): Set[A]
}
```

We know there are lots of methods we can define on `List`. The standard library is full of them! We also know that any method we care to write can be written using structural recursion. Finally, we can write these methods without modifying existing code.

Imagine `filter` was not defined on `List`. We can easily implement it as

```scala mdoc:silent
import List.*

def filter[A](list: List[A], pred: A => Boolean): List[A] = 
  list match {
    case Empty() => Empty()
    case Pair(head, tail) => 
      if pred(head) then Pair(head, filter(tail, pred))
      else filter(tail, pred)
  }
```

We could even use an extension method to make it appear as a normal method.

```scala mdoc:reset:invisible
enum List[A] {
  case Empty()
  case Pair(head: A, tail: List[A])
}
import List.*
```
```scala mdoc:silent
extension [A](list: List[A]) {
  def filter(pred: A => Boolean): List[A] = 
    list match {
      case Empty() => Empty()
      case Pair(head, tail) => 
        if pred(head) then Pair(head, tail.filter(pred))
        else tail.filter(pred)
    }
}
```

This shows we can add new functions to data without issue.

What about adding new elements to data? Perhaps we want to add a special case to optimize single-element lists. This is impossible without changing existing code. By definition, we cannot add a new element to an `enum` without changing the `enum`. Adding such a new element would break all existing pattern matches, and so require they all change. So in summary we can add new functions to data, but not new elements.

Now let's look at codata. This has the opposite extensibility; duality strikes again! In the codata case we can easily add new elements. We simply implement the `trait` that defines the codata interface. We saw this when we defined, for example, `ListSet`.

```scala mdoc:reset:invisible
trait Set[A] {
  def contains(elt: A): Boolean
  def insert(elt: A): Set[A]
  def union(that: Set[A]): Set[A]
}
```
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

What about adding new functionality? If the functionality can be defined in terms of existing functionality then we're ok. We can easily define this functionality, and we can use the extension method trick to make it appear like a built-in. However, if we want to define a function that cannot be expressed in terms of existing functions we are out of luck. Let's saw we want to define some kind of iterator over the elements of a `Set`. We might use a `LazyList`, the standard library's equivalent of `Stream` we defined earlier, because we know some sets have an infinite number of elements. Well, we can't do this without changing the definition of `Set`, which in turn breaks all existing implementations. We cannot define it in a different way because we don't know all the possible implementations of `Set`.

So in summary we can add new elements to codata, but not new functions.

If we tabulate this we clearly see that data and codata have orthogonal extensibility.

+---------------+------+--------+
| Extension     | Data | Codata |
+===============+======+========+
| Add elements  | No   | Yes    |
+---------------+------+--------+
| Add functions | Yes  | No     |
+---------------+------+--------+

This difference in extensibility gives us another rule for choosing between data and codata as an implementation strategy, in addition to the finite vs infinite distinction we saw earlier. If we want extensibilty of functions but not elements we should use data. If we have a fixed interface but an unknown number of possible implementations we should use codata.

You might wonder if we can have both forms of extensibility. Achieving this is called the **expression problem**. There are various ways to solve the expression problem, and we'll see one that works particularly well in Scala in a later chapter.


```scala mdoc:reset:silent
```
--->

## データと余データの拡張性 {#sec:codata:extensibility}

余データが `Stream` のような無限個の要素をもつ型を表せることについてはすでに見た。これは、常に有限個の要素しかもたないデータとの大きな違いのひとつである。ここでは、もうひとつの違い、データと余データから得られる拡張性の特徴について見ていこう。これらの違いが、両者のうちいずれかを選択するにあたての指針となる。

最初に、拡張性とは何かを定義しておきたい。拡張性とは、既存コードを変更することなく新しい機能を追加できる能力のことを指す（もし既存コードの変更を認めるのであれば、どんな拡張だってできてしまう）。コードは、機能の追加とバリアントの追加というふたつの軸で拡張される。データでは機能を追加するのは簡単だが、バリアントを追加するには既存コードの変更が必要となる。一方、余データではバリアントの追加は簡単だが機能の追加は難しい。このように、データと余データは直交する拡張性をもっている。

それでは、データと余データそれぞれの具体例を見てみよう。データとしてはおなじみの `List` 型を使う。

```scala mdoc:silent
enum List[A] {
  case Empty()
  case Pair(head: A, tail: List[A])
}
```

余データの例としては `Set` を使う。

```scala mdoc:silent
trait Set[A] {
  def contains(elt: A): Boolean
  def insert(elt: A): Set[A]
  def union(that: Set[A]): Set[A]
}
```

標準ライブラリがそうであるように、この `List` にもたくさんのメソッドを定義することができる。どんなメソッドも構造的再帰を使って記述できることはすでに学んだとおりで、それらは既存コードの変更なしに書くことができる。

`filter` が `List` に定義されていなかったと想像してほしい。これを実装するのは簡単である。

```scala mdoc:silent
import List.*

def filter[A](list: List[A], pred: A => Boolean): List[A] = 
  list match {
    case Empty() => Empty()
    case Pair(head, tail) => 
      if pred(head) then Pair(head, filter(tail, pred))
      else filter(tail, pred)
  }
```

拡張メソッドとして定義し、それを普通のメソッドのように見せることだってできる。

```scala mdoc:reset:invisible
enum List[A] {
  case Empty()
  case Pair(head: A, tail: List[A])
}
import List.*
```
```scala mdoc:silent
extension [A](list: List[A]) {
  def filter(pred: A => Boolean): List[A] = 
    list match {
      case Empty() => Empty()
      case Pair(head, tail) => 
        if pred(head) then Pair(head, tail.filter(pred))
        else tail.filter(pred)
    }
}
```

これにより、データには新しい関数を問題なく追加できることがわかる。

データに新しいバリアントを追加する場合はどうだろうか。たとえば、単一要素のリストを最適化する特別なケースを追加したいとしよう。これを既存コードの変更なしに行うことはできない。`enum` に新しいバリアントを追加するには `enum` の定義を変更する必要がある。さらに、そのような新しいバリアントを追加すれば、既存のパターンマッチはすべて壊れてしまい修正が必要となる。したがって、データには新しい関数を追加することはできるが、新しいバリアントを追加することはできない。

次に余データを見てみよう。こちらは拡張性が逆である。ここにも両者の双対性が見てとれる。余データの場合、新しいバリアントの追加は簡単である。余データのインターフェースを定義している `trait` を実装するだけでよい。本書では `ListSet` を定義したときなどにこれを行っている。

```scala mdoc:reset:invisible
trait Set[A] {
  def contains(elt: A): Boolean
  def insert(elt: A): Set[A]
  def union(that: Set[A]): Set[A]
}
```
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

新しい機能の追加についてはどうだろうか。もしその機能が既存の機能を組み合わせて定義可能であれば問題はない。そのような機能は簡単に定義できるし、拡張メソッドとして定義し、ビルトインメソッドのように見せることもできる。だが、既存の機能を使って定義できない機能を追加するのは困難である。`Set` の要素を走査するある種のイテレータを定義したいとしよう。集合は無限個の要素を持つ可能性があるため、以前定義した `Stream` に相当する標準ライブラリの `LazyList` を使うかもしれない。だが、このような機能を実現するには `Set` の定義を変更しなければならず、すべての既存の実装を壊してしまう。`Set` のあらゆる実装について知ることは不可能なので、別の方法でこれを定義することもできない。

まとめると、余データには新しいバリアントを追加することは可能だが、新しい機能を追加することはできない。

これを表にまとめると、データと余データの拡張性が直交していることがはっきりとわかる。

| 拡張           | データ | 余データ |
|---------------|-------|---------|
| バリアントの追加 | 不可   | 可     |
| 機能の追加      | 可    | 不可     |

この拡張性の違いは、データと余データのいずれかを実装戦略として選択するにあたって、以前に述べた「有限か無限か」という区別に加えて、もうひとつの指針を与えてくれる。もし機能の拡張は必要でもバリアントの拡張が不要なのであれば、データを選ぶべきである。一方、固定されたインターフェースをもつが、実装がいくつ必要となるか分からない場合は余データを選ぶべきである。

両方の拡張性を同時にもつことはできるのか気になる人もいるかもしれない。これは **式の問題（expression problem）** と呼ばれる。式の問題を解決する方法はいくつかある。後ほど、Scala において特に有効な解法を学ぶ。
