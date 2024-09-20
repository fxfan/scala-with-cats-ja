## 代数的データ型における代数

A question that sometimes comes up is where the "algebra" in algebraic data types comes from. I want to talk about this a little bit and show some of the algebraic manipulations that can be done on algebraic data types.

ときどき出てくる質問に、代数的データ型の「代数」とは何なのか、というものがある。この点について少し触れ、代数的データ型で行える代数的な操作をいくつか紹介したい。

The term algebra is used in the sense of abstract algebra, an area of mathematics.
Abstract algebra deals with algebraic structures. 
An algebraic structure consists of a set of values, operations on that set, and properties that those operations must maintain.

ここでの「代数」という用語は、数学の一分野である抽象代数学の意味で使われている。抽象代数学は、代数的構造を扱う分野である。代数的構造とは、ある値の集合、その集合に対する操作、およびそれらの操作が満たすべき性質を指している。

An example is the set of integers, the operations addition and multiplication, and the familiar properties of these operations such as associativity, which says that $a + (b + c) = (a + b) + c$.
The abstract in abstract algebra means that it doesn't deal with concrete values like integers---that would be far too easy to understand---and instead with abstractions with wacky names like semigroup, monoid, and ring.
The example of integers above is an instance of a ring.
We'll see a lot more of these soon enough!

具体例としては、整数の集合と、加法や乗法といった操作、そして結合法則 $a + (b + c) = (a + b) + c$ などといった操作の性質が挙げられる。抽象代数学の「抽象」とは、整数のような具体的な集合（そうであれば理解するのは極めて簡単なのだが）を扱うのではなく、代わりに半群、モノイド、環といった奇妙な名前のついた抽象的な概念を扱うことを意味している。上記の整数の例は、環の一例である。これから、そのような概念をたくさん見ていくことになるだろう。

Algebraic data types also correspond to the algebraic structure called a ring.
A ring has two operations, which are conventionally written $+$ and $\times$.
You'll perhaps guess that these correspond to sum and product types respectively, and you'd be absolutely correct.
What about the properties of these operations?
We'll they are similar to what we know from basic algebra:

代数的データ型も、環と呼ばれる代数的構造に対応している。環には、二つの演算があり、通常それらは $+$ および $\times$ と記述される。これらがそれぞれ直和型と直積型に対応していると推測したなら、それはまさにその通りである。では、環におけるこれらの演算の性質はどのようなものだろうか。それらは、基本的な代数について我々が知っているものと似ている。

- $+$ and $\times$ are associative, so $a + (b + c) = (a + b) + c$ and likewise for $\times$;
- $a + b = b + a$, known as commutivitiy;
- there is an identity $0$ such that $a + 0 = a$;
- there is an identity $1$ such that $a \times 1 = a$;
- there is distribution, so that $a \times (b + c) = (a \times b) + (a \times c)$

- $+$ と $\times$ は結合法則を満たす。つまり $a + (b + c) = (a + b) + c$ であり、$\times$ についても同様
- $a + b = b + a$ です。これは交換法則として知られている
- $a + 0 = a$ となる単位元 $0$ が存在する
- $a \times 1 = a$ となる単位元 $1$ が存在する
- 分配法則が成り立つ。つまり $a \times (b + c) = (a \times b) + (a \times c)$ である

So far, so abstract. 
Let's make it concrete by looking at actual examples in Scala.

かなり抽象的な話になってしまったので、ここからは実際の例をScalaで見て具体化してみよう。

Remember the algebraic data types work with types, so the operations $+$ and $\times$ take types as parameters.
So $Int \times String$ is equivalent to

代数的データ型は型を扱うので、演算 $+$ と $\times$ も型を引数として受け取る。たとえば、$Int \times String$ は次のように表される。

```scala mdoc:silent
final case class IntAndString(int: Int, string: String)
```

We can use tuples to avoid creating lots of names.

いちいち命名しなくてもよいようにタプルを使うという手もある。

```scala mdoc:reset:silent
type IntAndString = (Int, String)
```

We can do the same thing for $+$. $Int + String$ is

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

Can you work out which Scala type corresponds to the identity $1$ for product types?

直積型における単位元 $1$ に対応する Scala の型は何か考察せよ。

<div class="solution">
It's `Unit`, because adding `Unit` to any product doesn't add any more information.
So, `Int` contains exactly as much information as $Int \times Unit$ (written as the tuple `(Int, Unit)` in Scala).

直積型の単位元は `Unit` 型である。ある直積型に `Unit` 型のフィールドを追加しても、何も情報を追加したことにはならない。 `Int` 型がもつ情報量は、 $Int \times Unit$ （Scalaでタプルとして書くなら `(Int, Unit)`）型と同じである。
</div>

What about the Scala type corresponding to the identity $0$ for sum types?

また、直和型における単位元 $0$ に対応するScalaの型は何か、考察せよ。

<div class="solution">
It's `Nothing`, following the same reasoning as products: a case of `Nothing` adds no further information (and we cannot even create a value with this type.)

直和型の単位元は `Nothing` 型である。直積型のときと同様に、 `Nothing` というケースは何の情報ももたない。この型の値を生成することすらできない。
</div>


What about the distribution law? This allows us to manipulate algebraic data types to form equivalent, but perhaps more useful, representations.
Consider this example of a user data type.

代数的データ型における分配法則はどのようなものだろうか。この法則を使うと、代数的データ型を操作して、等価だがより使いやすい表現を作成することができる。その例として、ユーザを表すデータ型を考えてみよう。

```scala mdoc:silent
final case class Person(name: String, permissions: Permissions)
enum Permissions {
  case User
  case Moderator
}
```

Written in mathematical notation, this is

これを数学的に表記すると次のようになる。

$$
Person = String \times Permissions
$$
$$
Permissions = User + Moderator
$$

Performing substitution gets us

`Permissions` をその定義に置き換えると次のようになり、

$$
Person = String \times (User + Moderator)
$$

Applying distribution results in

さらに分配法則を適用すると結果は次のようになる。

$$
Person = (String \times User) + (String \times Moderator)
$$

which in Scala we can represent as

これはScalaでは次のように表現される。

```scala mdoc:reset:silent
enum Person {
  case User(name: String)
  case Moderator(name: String)
}
```

Is this representation more useful? I can't say without the context of where the data is being used. However I can say that knowing this manipulation is possible, and correct, is useful.

これが元の定義よりも有用かどうかは、データがどういうコンテキストで使われるのか分からないかぎり何とも言えないが、このような操作が可能であり、正しいものであり、役に立つことは間違いないだろう。

There is a lot more that could be said about algebraic data types, but at this point I feel we're really getting into the weeds.
I'll finish up with a few pointers to other interesting facts:

代数的データ型について言えることは他にもたくさんあるが、やや深入りしすぎてしまったようである。最後にいくつか興味深いトピックを簡単に紹介して締めくくりたい。

- Exponential types exist. They are functions! A function `A => B` is equivalent to $b^a$.
- Quotient types also exist, but they are a bit weird. Read up about them if you're interested.
- Another interesting algebraic manipulation is taking the derivative of an algebraic data type. This gives us a kind of iterator, known as a zipper, for that type.

- 指数型というものが存在する。それは関数である。関数 `A => B` は、数学的に $b^a$ に相当する
- 商型も存在するが、少し変わっている。興味があれば調べてみてほしい
- もうひとつの興味深い代数的操作として、代数的データ型の微分がある。これにより、その型に対するイテレータの一種である zipper が得られる
