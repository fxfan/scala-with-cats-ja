<!--

## Conclusions

In this chapter we looked at codata interpreters, and their extension to tagless final. Tagless final is particularly interesting because it solves the expression problem, allowing us to extend both the operations a program can perform and the interpretations of that program.

Our exploration of tagless final nicely illustrates the distinction between theory and craft introduced in Section [@sec:intro:three-levels]. We saw two different encoding of tagless final in Scala (three, if we count using context bounds as a different encoding). They are both tagless final at the theory level, but are very different to implement or use as a programmer. The "standard" encoding is relatively easy to implement for the library author, but tedious and potentially confusing for the user. The improved encoding places more work on the library author, but the user writes code in a natural style.

Tagless final is very powerful and it can be tempting to use it everywhere. I want to caution against this urge. Tagless final can cause problems, both for the author and the user. From the user's point of view everything works fine until they make a mistake. Then the errors can be confusing. Consider this code, where we have missed a parameter to `and`.

```scala mdoc:invisible
type Validation[A] = A => Either[String, A]

// The validation rule that always succeeds
def succeed[A](value: A): Either[String, A] = Right(value)
trait Algebra {
  type Ui[_]
}
trait Controls extends Algebra {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}
trait Layout extends Algebra {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
trait Program[-Alg <: Algebra, A] {
  def apply(alg: Alg): alg.Ui[A]
}
object Controls {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Program[Controls, String] =
    alg => alg.textInput(label, placeholder, validation)

  def choice[A](
    label: String, 
    options: Seq[(String, A)]
  ): Program[Controls, A] =
    alg => alg.choice(label, options)
}
extension [Alg <: Algebra, A](p: Program[Alg, A]) {
  def and[Alg2 <: Algebra, B](
    second: Program[Alg2, B]
  ): Program[Alg & Alg2 & Layout, (A, B)] =
    alg => alg.and(p(alg), second(alg))
}
```


```scala mdoc:fail
Controls.textInput("Name", "John Doe").and()
```

The error message *does* tell us the problem, but it exposes a lot of the internal machinery that the user is not normally exposed to, and hence they'll probably have difficult understanding. A straightforward data or codata interpreter does not have this problem.

From the library author's point of view, it is a lot more work to create tagless final code. It can also be difficult to onboard new developers to this code, as the techniques are not familiar to most.

As always, the applicability of tagless final comes down to the context in which it is used. In cases where the extensibility is truly justified it is a powerful tool. In other cases it just introduces unwarranted complexity.

The term "expression problem" was first introduced in an email by Phil Wadler [@wadler98:ep], but there are much earlier sources that discuss the same issue. One example is [@cook90:oo-adt]. Tagless final was first introduced in [@jacques09:finally-tagless] and expanded on in [@kiselyov12:tagless-final]. It is just one of many solutions that have been proposed to the expression problem. I'm no expert on the wider field of solutions to the expression problem, but of the papers I've read the ones I'd like to highlight object algebras [@oliveira12:object-algebras] and data types à la carte [@swierstra08:data-types]. Object algebras are, in all essentials, the same as tagless final. They were developed in object-oriented languages rather than functional programming languages, making an interesting case of convergent evolution in two distinct, but connected, fields of research. The object algebras paper is also a good read for a more formal, if brief, discussion of the theory behind the concepts we've been dealing with. Data types à la carte is a data, rather than codata, approach to the expression problem, and so makes an interesting contrast to tagless final. I find tagless final much simpler, so we have not explored data types à la carte in this book. Another noteworthy paper is [@10.1145/2692915.2628138], which discuss the duality between data and codata and its implication for embedded domain specific languages.

Tagless final was introduced using Haskell as the implementation language. The standard encoding in Scala is a direct translation of the Haskell implementation. The improved Scala encoding is my own creation. The use of the single abstract method shortcut was suggested by Jakub Kozłowski.


```scala mdoc:reset:silent
```
--->

## まとめ

本章では、余データ的インタープリタと、そこから Tagless Final へと至る道程について見てきた。Tagless Final が特に興味深いのは、それが式の問題を解決するからである。これにより、プログラムが実行できる操作と、そのプログラムの解釈の両方を拡張できるようになる。

Tagless Final の探求は[@sec:intro:three-levels]節で述べた理論と技法の違いをよく示している。Scala における Tagless Final のエンコーディングをふたつ（コンテキスト境界を別のエンコーディングと数えれば三つ）見てきた。いずれも理論的には Tagless Final であるものの、プログラマにとっての実装や使用の観点からは大きく異なる。標準的なエンコーディングは、ライブラリ作者にとって実装が比較的簡単だが、利用者にとっては記法が煩雑で混乱を招きやすい。一方で、改良されたエンコーディングを用いると、ライブラリ作者にとっての負担は大きいが、利用者は自然なスタイルでコードを書くことができる。

Tagless Final は非常に強力であり、あらゆるところで使いたくなるかもしれない。しかし、ここでひとつその衝動に釘を差しておきたい。Tagless Final は、ライブラリ作者にとっても利用者にとっても問題の原因になりうる。利用者の視点では、正しいコードを書いているうちはすべてがうまく動くが、ひとたび間違いがあると、エラーが非常にわかりにくくなる。次のコードを考えてみよう。このコードには `and` の引数が足りていない。

```scala mdoc:invisible
type Validation[A] = A => Either[String, A]

// 常に成功する検証ルール
def succeed[A](value: A): Either[String, A] = Right(value)
trait Algebra {
  type Ui[_]
}
trait Controls extends Algebra {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}
trait Layout extends Algebra {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
trait Program[-Alg <: Algebra, A] {
  def apply(alg: Alg): alg.Ui[A]
}
object Controls {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Program[Controls, String] =
    alg => alg.textInput(label, placeholder, validation)

  def choice[A](
    label: String, 
    options: Seq[(String, A)]
  ): Program[Controls, A] =
    alg => alg.choice(label, options)
}
extension [Alg <: Algebra, A](p: Program[Alg, A]) {
  def and[Alg2 <: Algebra, B](
    second: Program[Alg2, B]
  ): Program[Alg & Alg2 & Layout, (A, B)] =
    alg => alg.and(p(alg), second(alg))
}
```

```scala mdoc:fail
Controls.textInput("Name", "John Doe").and()
```

エラーメッセージはたしかに問題の所在を教えてくれる。だがそこには、利用者が普段は目にすることのない内部の仕組みが多く露出しており、おそらく理解に苦しむだろう。素直な実装をもつデータ的あるいは余データ的インタープリタではこのような問題は起こらない。

ライブラリ作者の視点からすると、Tagless Final のコードを書くのはかなりの労力を要する。また、使われるテクニックはほとんどの人にとって馴染みのないものであるため、新しい開発者を参加させるのも難しい。

いつものことだが、Tagless Final がうまく当てはまるかどうかは、それが使われる文脈による。拡張性が本当に必要とされる状況では強力な道具となるが、そうでない場合は不要な複雑さをもちこむだけである。

「式の問題」という用語は、Phil Wadler によるメール [@wadler98:ep] の中ではじめて用いられた。しかし、同じ問題を論じたもっと古い文献も存在する。その一例が [@cook90:oo-adt] である。Tagless Final というアプローチは [@jacques09:finally-tagless] においてはじめて提案され、[@kiselyov12:tagless-final] がそれを発展させた。これは、式の問題に対して提案されてきた多くの解法のひとつにすぎない。私は式の問題に対する解決法全般に詳しいわけではないが、これまでに読んだ論文の中では、オブジェクト代数 [@oliveira12:object-algebras] とデータ型アラカルト [@swierstra08:data-types] を特に紹介したい。オブジェクト代数は本質的には Tagless Final と同じものだが、関数型ではなくオブジェクト指向言語において発展してきたもので、ふたつの異なる（だが密接に関係した）研究分野における収斂進化の興味深い例と言える。また、このオブジェクト代数の論文は、本書で扱ってきた概念の背後にある理論について簡潔ながら形式的に論じており、読みごたえがある。データ型アラカルトは余データではなくデータ的なアプローチによって式の問題に取り組むもので、Tagless Final とは興味深い対比をなしている。ただ、私には Tagless Final のほうがずっと簡潔であるように思われるため、本書ではデータ型アラカルトのことは掘り下げなかった。注目すべきもうひとつの論文として [@10.1145/2692915.2628138] がある。これはデータと余データの双対性および、その双対性が組み込み DSL においてどのような意味をもつのかについて論じている。

Tagless Final は Haskell を実装言語として導入された。Scala における標準的なエンコーディングは、Haskell における実装を直接的に翻訳したものである。一方、改善された Scala 向けエンコーディングは私自身が考案した。また、単一抽象メソッドを用いた簡略化手法は Jakub Kozłowski によって提案された。
