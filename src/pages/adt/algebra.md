<!--

## The Algebra of Algebraic Data Types

A question that sometimes comes up is where the "algebra" in algebraic data types comes from. I want to talk about this a little bit and show some of the algebraic manipulations that can be done on algebraic data types.

The term algebra is used in the sense of abstract algebra, an area of mathematics.
Abstract algebra deals with algebraic structures. 
An algebraic structure consists of a set of values, operations on that set, and properties that those operations must maintain.
An example is the set of integers, the operations addition and multiplication, and the familiar properties of these operations such as associativity, which says that $a + (b + c) = (a + b) + c$.
The abstract in abstract algebra means that it doesn't deal with concrete values like integers---that would be far too easy to understand---and instead with abstractions with wacky names like semigroup, monoid, and ring.
The example of integers above is an instance of a ring.
We'll see a lot more of these soon enough!

Algebraic data types also correspond to the algebraic structure called a ring.
A ring has two operations, which are conventionally written $+$ and $\times$.
You'll perhaps guess that these correspond to sum and product types respectively, and you'd be absolutely correct.
What about the properties of these operations?
We'll they are similar to what we know from basic algebra:

- $+$ and $\times$ are associative, so $a + (b + c) = (a + b) + c$ and likewise for $\times$;
- $a + b = b + a$, known as commutivitiy;
- there is an identity $0$ such that $a + 0 = a$;
- there is an identity $1$ such that $a \times 1 = a$;
- there is distribution, so that $a \times (b + c) = (a \times b) + (a \times c)$

So far, so abstract. 
Let's make it concrete by looking at actual examples in Scala.

Remember the algebraic data types work with types, so the operations $+$ and $\times$ take types as parameters.
So $Int \times String$ is equivalent to

```scala mdoc:silent
final case class IntAndString(int: Int, string: String)
```

We can use tuples to avoid creating lots of names.

```scala mdoc:reset:silent
type IntAndString = (Int, String)
```

We can do the same thing for $+$. $Int + String$ is

```scala mdoc:silent
enum IntOrString {
  case IsInt(int: Int)
  case IsString(string: String)
}
```

or just

```scala mdoc:reset:silent
type IntOrString = Either[Int, String]
```


#### Exercise: Identities {-}

Can you work out which Scala type corresponds to the identity $1$ for product types?

<div class="solution">
It's `Unit`, because adding `Unit` to any product doesn't add any more information.
So, `Int` contains exactly as much information as $Int \times Unit$ (written as the tuple `(Int, Unit)` in Scala).
</div>

What about the Scala type corresponding to the identity $0$ for sum types?

<div class="solution">
It's `Nothing`, following the same reasoning as products: a case of `Nothing` adds no further information (and we cannot even create a value with this type.)
</div>


What about the distribution law? This allows us to manipulate algebraic data types to form equivalent, but perhaps more useful, representations.
Consider this example of a user data type.

```scala mdoc:silent
final case class Person(name: String, permissions: Permissions)
enum Permissions {
  case User
  case Moderator
}
```

Written in mathematical notation, this is

$$
Person = String \times Permissions
$$
$$
Permissions = User + Moderator
$$

Performing substitution gets us

$$
Person = String \times (User + Moderator)
$$

Applying distribution results in

$$
Person = (String \times User) + (String \times Moderator)
$$

which in Scala we can represent as

```scala mdoc:reset:silent
enum Person {
  case User(name: String)
  case Moderator(name: String)
}
```

Is this representation more useful? I can't say without the context of where the data is being used. However I can say that knowing this manipulation is possible, and correct, is useful.

There is a lot more that could be said about algebraic data types, but at this point I feel we're really getting into the weeds.
I'll finish up with a few pointers to other interesting facts:

- Exponential types exist. They are functions! A function `A => B` is equivalent to $b^a$.
- Quotient types also exist, but they are a bit weird. Read up about them if you're interested.
- Another interesting algebraic manipulation is taking the derivative of an algebraic data type. This gives us a kind of iterator, known as a zipper, for that type.


```scala mdoc:reset:silent
```
--->

## 代数的データ型における代数

ときどき挙がる質問に、代数的データ型はなぜ「代数」と名付けられているのか、というものがある。この点についてすこし触れ、代数的データ型で行える代数的な操作をいくつか紹介したい。

この「代数」という用語は、数学の一分野である抽象代数学の意味で使われている。抽象代数学は、代数的構造を扱う分野である。代数的構造は、ある値の集合、その集合上の操作、およびそれらの操作が満たすべき性質から構成される。

具体例として、整数の集合と、加法や乗法という操作、そして結合律 $a + (b + c) = (a + b) + c$ などといった操作の性質が挙げられる。抽象代数学の「抽象」とは、整数のような具体的な集合を扱うのではなく、代わりに半群、モノイド、環といった奇妙な名前のついた抽象的な概念を扱うことを意味している（具体的であれば理解するのは極めて簡単なのだが）。上述の整数の話は環の一例である。これから、そのような概念をたくさん見ていくことになるだろう。

代数的データ型も、環と呼ばれる代数的構造に対応している。環はふたつの演算をもち、通常それらは $+$ および $\times$ と記述される。これらがそれぞれ直和型と直積型に対応していると推測したなら、まさにそのとおりである。では、環におけるこれらの演算の性質はどのようなものだろうか。それらは、基本的な代数について我々が知っているものと似ている。

- $+$ と $\times$ は結合律を満たす。つまり $a + (b + c) = (a + b) + c$ であり、$\times$ についても同様である
- $+$ は交換律を満たす。つまり $a + b = b + a$ である
- $a + 0 = a$ となる単位元 $0$ が存在する
- $a \times 1 = a$ となる単位元 $1$ が存在する
- 分配律を満たす。つまり $a \times (b + c) = (a \times b) + (a \times c)$ である

かなり抽象的な話になってしまったので、ここからは Scala による実例を使って具体化してみよう。

Remember the algebraic data types work with types, so the operations $+$ and $\times$ take types as parameters.
So $Int \times String$ is equivalent to

代数的データ型は型を取り扱うものなので、演算 $+$ と $\times$ も型を引数として受け取る。たとえば、$Int \times String$ は次のように表される。

```scala mdoc:silent
final case class IntAndString(int: Int, string: String)
```

いちいち命名せずにタプルを使うという手もある。

```scala mdoc:reset:silent
type IntAndString = (Int, String)
```

$+$ という演算に対しても同じような表現が可能である。 $Int + String$ は次のように表される。

```scala mdoc:silent
enum IntOrString {
  case IsInt(int: Int)
  case IsString(string: String)
}
```

もしくは単に以下のように表現してもかまわない。

```scala mdoc:reset:silent
type IntOrString = Either[Int, String]
```

#### 演習: 単位元 {-}

直積型における単位元 $1$ に対応する Scala の型は何か考察せよ。

<div class="solution">
直積型の単位元は `Unit` 型である。ある直積型に `Unit` 型のフィールドを追加しても、何も情報を追加したことにはならない。 `Int` 型がもつ情報量は、 $Int \times Unit$ 型、Scala のタプルとして書くなら `(Int, Unit)` 型と同じである。
</div>

また、直和型における単位元 $0$ に対応する Scala の型は何か考察せよ。

<div class="solution">
直和型の単位元は `Nothing` 型である。直積型のときと同様に `Nothing` 型のバリアントは何も情報を追加しない。この型の値を生成することすらできない。
</div>

代数的データ型における分配律はどのようなものだろうか。この法則を使うと、代数的データ型を操作して、等価だがより使いやすい表現を作成できることがある。その例として、ユーザを表すデータ型を考えてみよう。

```scala mdoc:silent
final case class Person(name: String, permissions: Permissions)
enum Permissions {
  case User
  case Moderator
}
```

これを数学的に表記すると次のようになる。

$$
Person = String \times Permissions
$$
$$
Permissions = User + Moderator
$$

`Permissions` をその定義で置き換えると次のようになり、

$$
Person = String \times (User + Moderator)
$$

さらに分配法則を適用すると結果は次のようになる。

$$
Person = (String \times User) + (String \times Moderator)
$$

これは Scala では次のように表現される。

```scala mdoc:reset:silent
enum Person {
  case User(name: String)
  case Moderator(name: String)
}
```

これが元の定義よりも有用かどうかは、データがどういうコンテキストで使われるのかわからないかぎり何とも言えないが、このような操作が可能かつ正しいものであり、役に立ちうることは間違いないだろう。

代数的データ型について言えることは他にもたくさんあるが、やや深入りしすぎてしまったようである。最後にいくつか興味深いトピックを簡単に紹介して締めくくりたい。

- 指数型というものが存在する。それは関数である。関数 `A => B` は、数学的に $b^a$ に相当する
- 商型も存在するが、すこし変わっている。興味があれば調べてみてほしい
- もうひとつの興味深い代数的操作として、代数的データ型の微分がある。これにより、その型に対するイテレータの一種である zipper が得られる
