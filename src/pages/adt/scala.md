## Scalaにおける代数的データ型

Now we know what algebraic data types are, we will turn to their representation in Scala. The important point here is that the translation to Scala is entirely determined by the structure of the data; no thinking is required! This means the work is in finding the structure of the data that best represents the problem at hand. Work out the structure of the data and the code directly follows from it.

代数的データ型が何であるかを理解したところで、次にそれがScalaでどのように表現されるかを見ていこう。ここで重要なのは、Scalaへの翻訳がデータの構造によって完全に決定され、特別な考察は必要ないということである。そのため、取り組んでいる課題に最適なデータの構造を見つけることが主な作業となる。データ構造を考え出し、それに従ってコードを書けばよい。

As algebraic data types are defined in terms of logical ands and logical ors, to represent algebraic data types in Scala we must know how to represent these two concepts. Scala 3 simplifies the representation of algebraic data types compared to Scala 2, so we'll look at each language version separately.

代数的データ型は論理積と論理和で定義されるため、Scalaで代数的データ型を表現するためには、これら二つの概念がどのように表現されるかを知っておく必要がある。Scala3では、Scala2に比べて代数的データ型の表現が簡素化されているため、それぞれの言語バージョンを別々に見ていく。

I'm assuming that you're familiar with the language features we use to represent algebraic data types in Scala, so I won't be going over them.

なお、代数的データ型をScalaで表現するために使用する言語機能についてはすでに知っているものとし、その説明は割愛する。

### Scala3における代数的データ型

In Scala 3 a logical and (a product type) is represented by a `final case class`. If we define a product type `A` is `B` *and* `C`, the representation in Scala 3 is

Scala3では、論理積（直積型）は `final case class` で表現される。直積型 `A` が `B` *と* `C` の集まりであると定義する場合、そのScala3での表現は次のようになる。

```scala
final case class A(b: B, c: C)
```

Not everyone makes their case classes `final`, but they should. A non-`final` case class can still be extended by a class, which breaks the closed world criteria for algebraic data types.

この case class を `final` にしない人もいるが、するべきである。`final` でない case class は他のクラスによって拡張される可能性があり、これは代数的データ型の閉じた世界を実現する条件を壊してしまう。

A logical or (a sum type) is represented by an `enum`. For the sum type `A` is `B` *or* `C`, the Scala 3 representation is

論理和（直和型）は enum で表現される。直和型 `A` が `B` *または* `C` である場合、そのScala3での表現は次のようになる。


```scala
enum A {
  case B
  case C
}
```

There are a few wrinkles to be aware of. 
If we have a sum of products, such as:

注意すべき点がいくつかある。データが積の和である場合、たとえば次のようなデータであれば、

- `A` is `B` or `C`; and
- `B` is `D` and `E`; and
- `C` is `F` and `G`

- `A` は `B` または `C`
- `B` は `D` および `E`
- `C` は `F` および `G`

コード表現は次のようになる。

```scala
enum A {
  case B(d: D, e: E)
  case C(f: F, g: G)
}
```

In other words we don't write `final case class` inside an `enum`. You also can't nest `enum` inside `enum`. Nested logical ors  can be rewritten into a single logical or containing only logical ands (known as disjunctive normal form) so this is not a limitation in practice. However the Scala 2 representation is still available in Scala 3 should you want more expressivity.

つまり、`enum` の中には `final case class` とは書かない。また、`enum` の中に `enum` をネストすることもできない。ネストされた論理和は、論理積の論理和（これを選言標準形と呼ぶ）に書き換えることができるので、実用にあたってこれが制約になることはない。ただし、ネストされた論理和を表すことのできるScala2の表現はScala3でも引き続き利用可能である。

### Scala2における代数的データ型

A logical and (product type) has the same representation in Scala 2 as in Scala 3. If we define a product type `A` is `B` *and* `C`, the representation in Scala 2 is

論理積（直積型）は、Scala2でもScala3でも同じ表現をもつ。直積型 `A` が `B` *と* `C` の集まりであると定義する場合、Scala2での表現は次のようになる。

```scala
final case class A(b: B, c: C)
```

A logical or (a sum type) is represented by a `sealed abstract class`.  For the sum type `A` is a `B` *or* `C` the Scala 2 representation is

論理和（直和型）は `sealed abstract class` で表現される。直和型 `A` が `B` *または* `C` である場合、Scala2での表現は以下のとおりである。

```scala
sealed abstract class A
final case class B() extends A
final case class C() extends A
```

Scala 2 has several little tricks to defining algebraic data types.

Scala2で代数的データ型を定義するにあたっては小さなテクニックがいくつかある。

Firstly, instead of using a `sealed abstract class` you can use a `sealed trait`. There isn't much practical difference between the two. When teaching beginners I'll often use `sealed trait` to avoid having to introduce `abstract class`. I believe `sealed abstract class` has slightly better performance and Java interoperability, but I haven't tested this. I also think `sealed abstract class` is closer, semantically, to the meaning of a sum type.

まず、`sealed abstract class` の代わりに `sealed trait` を使用することができる。この二つの間には大きな実用的差異はない。私の場合、初心者に教える際には `sealed trait` を用いることが多い。そうすることで `abstract class` を紹介する手間を避けることができる。`sealed abstract class` の方が若干パフォーマンスが良く、Javaとの互換性が高いと思われるが、テストをして確認したわけではない。また、`sealed abstract class` の方が意味的に直和型に近いとも考えられる。

For extra style points we can `extend Product with Serializable` from `sealed abstract class`. Compare the reported types below with and without this little addition.

コードをさらに洗練させるために、`sealed abstract class` を `Product` と `Serializable` を継承した型として定義することもできる。このちょっとした追加がある場合とない場合で、推論される型を比較してみてほしい。

Let's first see the code without extending `Product` and `Serializable`.

まずは `Product` と `Serializable` を拡張しないコードを見てみよう。

```scala mdoc:silent
sealed abstract class A
final case class B() extends A
final case class C() extends A
```

```scala mdoc:silent
val list = List(B(), C())
// list: List[A extends Product with Serializable] = List(B(), C())
```

Notice how the type of `list` includes `Product` and `Serializable`. 

`list` 変数の型に `Product` と `Serializable` が含まれている点に注目してほしい。

Now we have extending `Product` and `Serializable`.

そこで `A` に `Product` と `Serializable` を継承させる。

```scala mdoc:reset:silent
sealed abstract class A extends Product with Serializable
final case class B() extends A
final case class C() extends A
```
   
```scala mdoc
val list = List(B(), C())
```

Much easier to read!

これで推論される型がシンプルに読みやすくなる。

You'll only see this in Scala 2. Scala 3 has the concept of **transparent traits**, which aren't reported in inferred types, so you'll see the same output in Scala 3 no matter whether you add `Product` and `Serializable` or not.

この事情はScala2でのみ確認できる。Scala3には**transparentトレイト**という概念があり、このトレイトは型推論で報告される型には反映されない。そのため、Scala3では `Product` と `Serializable` を追加してもしなくても同じ結果が得られる。

Finally, if a logical and holds no data we can use a `case object` instead of a `case class`. For example, if we're defining some type `A` that holds no data we can just write

最後に、論理積がデータをひとつも保持しない場合、`case class` の代わりに `case object` を使用することができる。たとえば、データを保持しない型 `A` は次のように定義できる。

```scala mdoc:silent
case object A
```

There is no need to mark the `case object` as `final`, as objects cannot be extended.

オブジェクトは拡張できないので、`case object` を `final` としてマークする必要はない。

### 例

Let's make the discussion above more concrete with some examples.

いくつか例を挙げて、ここまでの議論をもっと具体的に考えてみよう。

#### RoleとUser

In the discussion forum example, we said a role is normal, moderator, or administrator. This is a logical or, so we can directly translate it to Scala using the appropriate pattern. In Scala 3 we write

ディスカッションフォーラムの例では、ロールは一般ユーザ・モデレータ・管理者のいずれかであるとした。これは論理和であり、適切なパターンを用いて機械的にScalaに翻訳することができる。Scala3であれば次のようになる。

```scala mdoc:silent
enum Role {
  case Normal
  case Moderator
  case Administrator
}
```

Scala2の場合は以下のように書ける。

```scala mdoc:reset:silent
sealed abstract class Role extends Product with Serializable
case object Normal extends Role
case object Moderator extends Role
case object Administrator extends Role
```

The cases within a role don't hold any data, so we used a `case object` in the Scala 2 code.

各ロールはデータをひとつも保持しないため、Scala2のコードでは `case class` ではなく `case object` を使っている。

We defined a user as a screen name, an email address, a password, and a role. In both Scala 3 and Scala 2 this becomes

ユーザは、表示名とメールアドレスとパスワードとロールから構成されると定義した。これはScala3とScala2のどちらにおいても次のようになる。

```scala mdoc:silent
final case class User(
  screenName: String,
  emailAddress: String,
  password: String,
  role: Role
)
```

I've used `String` to represent most of the data within a `User`, but in real code we might want to define distinct types for each field.

`User` が保持するデータを主に `String` 型で表現しているが、実際のコードでは各フィールドに対して個別の型を定義したほうがよいかもしれない。

#### Path

We defined a path as a sequence of actions of a virtual pen. The possible actions are straight lines, Bezier curves, or movement that doesn't result in visible output. A straight line has an end point (the starting point is implicit), a Bezier curve has two control points and an end point, and a move has an end point. 

前述のデータ例で、パスを仮想ペンの一連の動作として定義した。それによると、可能な動作には、直線、ベジエ曲線、または目に見える出力を伴わない移動がある。また、直線には終点があり（始点は自動的に決まる）、ベジエ曲線には二つの制御点と終点があり、移動には終点がある。

This has a straightforward translation to Scala. We can represent paths as the following in both Scala 3 and Scala 2.

これらはストレートにScalaでの表現に置き換えることができる。Scala3とScala2、どちらにおいてもパスは次のように表現できる。

```scala mdoc:invisible
type Action = Int
```
```scala mdoc:silent
final case class Path(actions: Seq[Action])
```

An action is a logical or, so we have different representations in Scala 3 and Scala 2. In Scala 3 we'd write

ペンの動作は論理和なので、Scala3とScala2で表現の仕方が異なる。Scala3では次のように記述できる。

```scala mdoc:reset:invisible
type Point = Int
```
```scala mdoc:silent
enum Action {
  case Line(end: Point)
  case Curve(cp1: Point, cp2: Point, end: Point)
  case Move(end: Point)
}
```

where `Point` is a suitable representation of a two-dimensional point.

なお、ここで用いている `Point` は二次元座標を表現したクラスであるとする。

In Scala 2 we have to go with the more verbose

Scala2ではもっと冗長な表現方法を用いる必要がある。

```scala mdoc:reset:invisible
type Point = Int
```
```scala mdoc:silent
sealed abstract class Action extends Product with Serializable 
final case class Line(end: Point) extends Action
final case class Curve(cp1: Point, cp2: Point, end: Point)
  extends Action
final case class Move(end: Point) extends Action
```


### Scala3における代数的データ型の表現

We've seen that the Scala 3 representation of algebraic data types, using `enum`, is more compact than the Scala 2 representation. However the Scala 2 representation is still available. Should you ever use the Scala 2 representation in Scala 3? There are a few cases where you may want to:

Scala3の `enum` を使用した代数的データ型の表現が、Scala2のそれよりもコンパクトであることを見てきたが、Scala2の表現は依然として使用可能である。では、Scala3であえてScala2の表現を使うべき場面はあるのかというと、いくつか考慮すべきケースがある。

- Scala 3's doesn't currently support nested `enums` (`enums` within `enums`). This may change in the future, but right now it can be more convenient to use the Scala 2 representation to express this without having to convert to disjunctive normal form.

- 現在のところ、Scala3ではネストされた `enum`（`enum` 内の `enum`）をサポートしていない。これは今後変更されるかもしれないが、現時点では、これを選言標準形に変換することなく表現するために、Scala2の方法を使用する方が便利な場合がある。

- Scala 2's representation can express things that are almost, but not quite, algebraic data types. For example, if you define a method on an `enum` you must be able to define it for all the members of the `enum`. Sometimes you want a case of an `enum` to have methods that are only defined for that case. To implement this you'll need to use the Scala 2 representation instead. 

Scala2の表現方法では、代数的データ型に近いが完全にそうとは言えないものを表現することができる。たとえば、`enum` にメソッドを定義する場合、そのメソッドは `enum` のすべてのメンバーに対して意味をなさなくてはならないが、特定のメンバーのみにメソッドを定義したいことがある。このような場合、Scala2の表現方法を使用する必要がある。


#### 演習: 木構造 {-}

To gain a bit of practice defining algebraic data types, code the following description in Scala (your choice of version, or do both.)

代数的データ型の定義に慣れるため、以下の記述をScalaのコードで表せ。なお、Scalaのバージョンは自由に選んでよい。両方を試してもかまわない。

A `Tree` with elements of type `A` is:

`A` 型の要素をもつ `Tree` は以下のいずれかとして定義される。

- a `Leaf` with a value of type `A`; or
- a `Node` with a left and right child, which are both `Trees` with elements of type `A`.

- `A` 型の値を保持する `Leaf`、もしくは
- `A` 型の要素をもつ `Tree` を左右の子として保持する `Node`

<div class="solution">
We can directly translate this binary tree into Scala. Here's the Scala 3 version.

このような二分木は機械的にScalaのコードに落とし込むことができる。以下はScala3で書いたコードである。

```scala mdoc:silent
enum Tree[A] {
  case Leaf(value: A)
  case Node(left: Tree[A], right: Tree[A])
}
```

In the Scala 2 encoding we write

Scala2なら次のようになる。

```scala mdoc:reset:silent
sealed abstract class Tree[A] extends Product with Serializable
final case class Leaf[A](value: A) extends Tree[A]
final case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
```
</div>
