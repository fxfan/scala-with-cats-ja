## 正規表現 Regular Expressions

We'll start this case study by briefly describing the usual task for regular expressions---matching text---and then take a more theoretical view. We'll then move on to implementation.

このケーススタディでは、まず正規表現の一般的なタスクである「テキストのマッチング」について簡単に紹介し、次に、より理論的な観点から説明を加える。その後、実装へと進む。

We most commonly use regular expressions to determine if a string matches a particular pattern.
The simplest regular expression is one that matches only one string.
In Scala we can create a regular expression by calling the `r` method on `String`.
Here's a regular expression that matches exactly the string `"Scala"`.

正規表現は、ある文字列が特定のパターンに一致するかどうかを判定する技術として、もっとも広く用いられている。もっとも単純な正規表現は、ただひとつの文字列にのみマッチするものである。Scalaでは、`String` に対して `r` メソッドを呼び出すことで正規表現を作成できる。次の正規表現は、文字列 `"Scala"` に正確に一致する。

```scala mdoc:silent 
val regexp = "Scala".r
```

We can see that it matches only `"Scala"` and fails if we give it a shorter or longer input.

これが `"Scala"` にだけマッチし、文字列がそれより短くても長くてもマッチしないということを見ておこう。

```scala mdoc
regexp.matches("Scala")
regexp.matches("Sca")
regexp.matches("Scalaland")
```

Notice we already have a separation between description and action. 
The description is the regular expression itself, created by calling the `r` method, and the action is calling the `matches` method on the regular expression.

すでに指示と実行が分かれていることに注目してほしい。指示は `r` メソッド呼び出しによって作成される正規表現そのものであり、正規表現に対して `matches` メソッドを呼び出すことが実行にあたる。

There are some characters that have a special meaning within the `String` describing a regular expression.
For example, the character `*` matches the preceding character zero or more times.

正規表現を記述する文字列においては、特別な意味を持つ文字がいくつか存在する。たとえば、`*` という文字は、直前の文字の0回以上の繰り返しにマッチする。

```scala mdoc:reset:silent
val regexp = "Scala*".r
```
```scala mdoc
regexp.matches("Scal")
regexp.matches("Scala")
regexp.matches("Scalaaaa")
```

We can also use parentheses to group sequences of characters.
For example, if we wanted to match all the strings like `"Scala"`, `"Scalala"`, `"Scalalala"` and so on, we could use the following regular expression.

括弧を使って文字の並びをグループ化することもできる。たとえば、`"Scala"`、`"Scalala"`、`"Scalalala"` といった文字列にマッチさせたければ、次のような正規表現を使用できる。

```scala mdoc:reset:silent
val regexp = "Scala(la)*".r
```

Let's check it matches what we're looking for.

期待どおりにマッチするか確認しておこう。

```scala mdoc
regexp.matches("Scala")
regexp.matches("Scalalalala")
```

We should also check it fails to match as expected.

マッチしないケースも期待どおりであるか確認しておくほうがよいだろう。

```scala mdoc
regexp.matches("Sca")
regexp.matches("Scalal")
regexp.matches("Scalaland")
```

That's all I'm going to say about Scala's built-in regular expressions. If you'd like to learn more there are many resources online. The [JDK documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/regex/Pattern.html) is one example, which describes all the features available in the JVM implementation of regular expressions.

Scala組み込みの正規表現について述べるのは、ここまでにしておく。もっと詳しく知りたい場合は、オンラインにある多くのリソースを参照してほしい。たとえば、[JDKのドキュメント](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/regex/Pattern.html)では、正規表現のJVM実装で利用できるすべての機能について記述されている。

Let's turn to the theoretical description, such as we might find in a textbook. A regular expression is:

1. the empty regular expression that matches nothing;
2. a string, which matches exactly that string (including the empty string); 
3. the concatenation of two regular expressions, which matches the first regular expression and then the second;
4. the union of two regular expressions, which matches if either expression matches; and
5. the repetition of a regular expression (often known as the Kleene star), which matches zero or more repetitions of the underlying expression.

次に、教科書に載っているような理論的な説明に移ろう。正規表現は以下のような構成要素から成り立っている。

1. 何にもマッチしない空の正規表現
2. ある文字列（空文字列も含む）に正確に一致する文字列
3. 二つの正規表現を連結したもの。最初の正規表現にマッチした後に二番目の正規表現にマッチする
4. 二つの正規表現の和集合。いずれかの正規表現にマッチする
5. 正規表現の繰り返し。基礎となる表現の0回以上の繰り返しにマッチする

This kind of description may seem very abstract if you're not used to it. It is very useful for our purposes because it defines a minimal API that we can easily implement. Let's walk through the description and see how each part relates to code.

この種の記述は、慣れていないと抽象的に思えるかもしれないが、簡単に実装できる最小限のAPIを定義しており、今回の目的にとって非常に役に立つ。この記述を順に追いながら、各項目がどのようにコードで表現されるかを見ていこう。

The empty regular expression is defining a constructor with type `() => Regexp`, which we can simplify to a value of type `Regexp`.
In Scala we put constructors on the companion object, so this tells us we need

空の正規表現は `() => Regexp` 型のコンストラクタを定義しており、これは `Regexp` 型の値に簡略化することができる。Scalaではコンストラクタをコンパニオンオブジェクトに配置するので、まず書くべきなのは次のようなコードである。

<!-- 
TODO: あらためて、ここでいうコンストラクタとはオブジェクト指向における一般的な意味と異なり、もっと広くオブジェクトを生成する関数、別の言葉でいえばファクトリメソッドのようなものを表していることを訳注する。かどうか考える。
-->

```scala
object Regexp {
  val empty: Regexp =
    ???
}
```

The second part tells us we need another constructor, this one with type `String => Regexp`.

第2項からは、`String => Regexp` 型の別のコンストラクタが必要であることがわかる。

```scala
object Regexp {
  val empty: Regexp =
    ???

  def apply(string: String): Regexp =
    ???
}
```

The other three components all take a regular expression and produce a regular expression.
In Scala these will become methods on the `Regexp` type.
Let's model this as a `trait` for now, and define these methods.

残りの3項はすべて、正規表現から別の正規表現を生成する。Scalaではこれらは `Regexp` 型の上に定義されるメソッドとして記述できる。ひとまずは `Regexp` を `trait` で表現し、メソッドを定義することにしよう。

The first method, the concatenation of two regular expressions, is conventionally called `++` in Scala.

ひとつ目は二つの正規表現を連結するメソッドである。Scalaでは二つのオブジェクトの連結メソッドは慣例的に `++` という名前をもつ。

```scala
trait Regexp {
  def ++(that: Regexp): Regexp
}
```

Union is conventionally called `orElse`.

和集合を生成するメソッドは慣例に従って `orElse` と名付ける。

```scala
trait Regexp {
  def ++(that: Regexp): Regexp
  def orElse(that: Regexp): Regexp
}
```

Repetition we'll call `repeat`, and define an alias `*` that matches how this operation is written in conventional regular expressions.

繰り返しによって新しい `Regexp` を生成するメソッドは `repeat` と呼ぶことにし、標準的な正規表現での記法に倣って、`*` をそのエイリアスとして定義する。

```scala
trait Regexp {
  def ++(that: Regexp): Regexp
  def orElse(that: Regexp): Regexp
  def repeat: Regexp
  def `*`: Regexp = this.repeat
}
```

We're missing one thing: a method to actually match our regular expression against some input. Let's call this method `matches`.

正規表現を実際に入力とマッチさせるメソッドも忘れてはならない。このメソッドは `matches` と呼ぶことにする。

```scala
trait Regexp {
  def ++(that: Regexp): Regexp
  def orElse(that: Regexp): Regexp
  def repeat: Regexp
  def `*`: Regexp = this.repeat
  
  def matches(input: String): Boolean
}
```

This completes our API.
Now we can turn to implementation.
We're going to represent `Regexp` as an algebraic data type, and each method that returns a `Regexp` will return an instance of this algebraic data type.
What should be the elements that make up the algebraic data type?
There will be one element for each method, and the constructor arguments will be exactly the parameters passed to the method *including the hidden `this` parameter for methods on the trait*.

以上でAPIは完成である。次は実装に取り掛かる。`Regexp` を代数的データ型として表現し、`Regexp` 型の戻り値をもつ各メソッドにはこの代数的データ型のインスタンスを返却させる。代数的データ型を構成するバリアントはどのようなものにすべきだろうか。それぞれのメソッドに対してひとつバリアントが必要であり、メソッドに渡されるパラメータがそのままコンストラクタ引数となる。なお、*暗黙的に渡される `this` もパラメータのひとつと考える*。

それらを反映したコードは以下のとおり。

```scala mdoc:silent
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*`: Regexp = this.repeat
  
  def matches(input: String): Boolean =
    ???
  
  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  val empty: Regexp = Empty
  
  def apply(string: String): Regexp =
    Apply(string)
}
```

A quick note about `this`. We can think of every method on an object as accepting a hidden parameter that is the object itself. This is `this`. (If you have used Python, it makes this explicit as the `self` parameter.) As we consider `this` to be a parameter to a method call, and our implementation strategy is to capture all the method parameters in a data structure, we must make sure we capture `this` when it is available. The only case where we don't capture `this` is when we are defining a constructor on a companion object.

`this` についての簡単に説明しておく。オブジェクトのすべてのメソッドは、そのオブジェクト自体を隠されたパラメータとして受け取っていると考えることができる。これが `this` である（Pythonを使用したことがあれば、これが `self` パラメータとして明示されるのを見たことがあるだろう）。`this` をメソッド呼び出しのパラメータと見なし、また実装戦略としてはすべてのメソッドパラメータをデータ構造に取り込むのだから、`this` が利用可能なのであればそれも取り込む必要がある。ただし、コンパニオンオブジェクトに定義されるコンストラクタについては、`this` を取り込む必要はない。

Notice that we haven't implemented `matches`. It doesn't return a `Regexp` so we cannot return an element of our algebraic data type. What should we do here? `Regexp` is an algebraic data type and `matches` transforms an algebraic data type into a `Boolean`. Therefore we can use structural recursion! Let's write out the skeleton, including the recursion rule.

`matches` をまだ実装していないことに注意してほしい。このメソッドの戻り値は `Regexp` 型ではないし、代数的データ型のバリアントを返すことはできない。どうすべきだろうか。`Regexp` は代数的データ型であり、`matches` は代数的データ型を `Boolean` に変換するのだから、構造的再帰を使用できる。骨組みを書き出し、データ構造が再帰的であるところに再帰呼び出しを配置してみよう。

```scala
def matches(input: String): Boolean =
  this match {
    case Append(left, right)   => left.matches(???) ??? right.matches(???)
    case OrElse(first, second) => first.matches(???) ??? second.matches(???)
    case Repeat(source)        => source.matches(???) ???
    case Apply(string)         => ???
    case Empty                 => ???
  }
```

Now we can apply the usual strategies to complete the implementation. Let's reason independently by case, starting with the case for `Empty`. This case is trivial as it always fails to match, so we just return `false`.

これで、実装の完成に向けて通常の戦略を適用することができる。ケースごとに考えていこう。まずは `Empty` だが、このケースは常にマッチに失敗するので難しいことは何もない。ただ `false` を返すだけでよい。

```scala
def matches(input: String): Boolean =
  this match {
    case Append(left, right)   => left.matches(???) ??? right.matches(???)
    case OrElse(first, second) => first.matches(???) ??? second.matches(???)
    case Repeat(source)        => source.matches(???) ???
    case Apply(string)         => ???
    case Empty                 => false
  }
```

Let's move on to the `Append` case. This should match if the `left` regular expression matches the start of the `input`, and the `right` regular expression matches starting where the `left` regular expression stopped. This has uncovered a hidden requirement: we need to keep an index into the `input` that tells us where we should start matching from. Using a nested method is the easiest way to keep around additional information that we need. Here I've created a nested method that returns an `Option[Int]`. The `Int` is the new index to use, and we return an `Option` to indicate if the regular expression matched or not.

`Append` のケースに進もう。このケースにマッチするとは、正規表現 `left` が `input` の先頭にマッチし、そのマッチが終了した位置から正規表現 `right` がマッチを開始するということである。ここで新たな要件が明らかになった。どこからマッチを始めるべきかを示すインデックスを `input` と合わせて保持する必要がある。この追加情報を手元に置いておくには、ネストしたメソッドを使うのがもっとも簡単である。ここでは `Option[Int]` を返すネストメソッドを作成した。`Int` は新しいインデックスを表し、それが `Option` であることは正規表現にマッチしたかどうかを示す。


```scala
def matches(input: String): Boolean = {
  def loop(regexp: Regexp, idx: Int): Option[Int] =
    regexp match {
      case Append(left, right) =>
        loop(left, idx).flatMap(idx => loop(right, idx))
      case OrElse(first, second) => 
        loop(first, idx) ??? loop(second, ???)
      case Repeat(source) => 
        loop(source, idx) ???
      case Apply(string) => 
        ???
      case Empty =>
        None
    }

  // 入力全体がマッチしたかどうかを確認する
  loop(this, 0).map(idx => idx == input.size).getOrElse(false)
}
```

Now we can go ahead and complete the implementation.

さらに先へ進み、実装を完了させよう。

```scala
def matches(input: String): Boolean = {
  def loop(regexp: Regexp, idx: Int): Option[Int] =
    regexp match {
      case Append(left, right) =>
        loop(left, idx).flatMap(i => loop(right, i))
      case OrElse(first, second) => 
        loop(first, idx).orElse(loop(second, idx))
      case Repeat(source) =>
        loop(source, idx)
          .flatMap(i => loop(regexp, i))
          .orElse(Some(idx))
      case Apply(string) =>
        Option.when(input.startsWith(string, idx))(idx + string.size)
    }

  // 入力全体がマッチしたかどうかを確認する
  loop(this, 0).map(idx => idx == input.size).getOrElse(false)
}
```

The implementation for `Repeat` is a little tricky, so I'll walk through the code.

`Repeat` の実装はややトリッキーなので、コードを詳しく見ておこう。

```scala
case Repeat(source) =>
  loop(source, idx)
    .flatMap(i => loop(regexp, i))
    .orElse(Some(idx))
```

The first line (`loop(source, index)`) is seeing if the `source` regular expression matches.
If it does we loop again, but on `regexp` (which is `Repeat(source)`), not `source`. 
This is because we want to repeat an indefinite number of times. 
If we looped on `source` we would only try twice.
Remember that failing to match is still a success; repeat matches zero or more times.
This condition is handled by the `orElse` clause.

最初の行となる `loop(source, index)` では、入力が正規表現 `source` にマッチするかを確認している。 マッチした場合、再度ループを行うが、次は `regexp`（`Repeat(source)`）にマッチするかを調べる。これは、試行を無限に繰り返すためである。もしここで `source` に対して `loop` を呼び出したら、マッチするかどうかのチェックは二回だけで終わってしまう。`Repeat` が表すのは0回以上のマッチなので、`loop` によるマッチに失敗しても全体としては成功と見なされる。この条件は `orElse` 句によって実現されている。

We should test that our implementation works.

最後に、実装が正しく動作するかをテストしておこう。

```scala mdoc:reset:invisible
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int): Option[Int] =
      regexp match {
        case Append(left, right) =>
          loop(left, idx).flatMap(i => loop(right, i))
        case OrElse(first, second) =>
          loop(first, idx).orElse(loop(second, idx))
        case Repeat(source) =>
          loop(source, idx)
            .flatMap(i => loop(regexp, i))
            .orElse(Some(idx))
        case Apply(string) =>
          Option.when(input.startsWith(string, idx))(idx + string.size)
        case Empty =>
          None
      }

    // 入力全体がマッチしたかどうかを確認する
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  def apply(string: String): Regexp =
    Apply(string)
}
```

Here's the example regular expression we started the chapter with.

以下は、この章の冒頭に登場した正規表現の例を `Regexp` オブジェクトで表したものである。

```scala mdoc:silent
val regexp = Regexp("Sca") ++ Regexp("la") ++ Regexp("la").repeat
```

Here are cases that should succeed.

以下はマッチに成功するケース。

```scala mdoc
regexp.matches("Scala")
regexp.matches("Scalalalala")
```

Here are cases that should fail.

そして、以下は失敗すべきケースである。

```scala mdoc
regexp.matches("Sca")
regexp.matches("Scalal")
regexp.matches("Scalaland")
```

Success! At this point we could add many extensions to our library. For example, regular expressions usually have a method (by convention denoted `+`) that matches one or more times, and one that matches zero or once (usually denoted `?`). These are both conveniences we can build on our existing API. However, our goal at the moment is to fully understand interpreters and the implementation technique we've used here. So in the next section we'll discuss these in detail.

これで実装は完了である。このライブラリに更に多くの拡張機能を追加することもできる。たとえば、正規表現には通常、1回以上の繰り返しにマッチするメソッド（通常 `+` で表される）や、0回もしくは1回の出現にマッチするメソッド（通常 `?` で表される）がある。これらは便利だし、現実装の仕組みの延長で追加できるが、今は行わない。ここでの目標はインタープリタとその実装手法を完全に理解することにある。次節ではこれらについて詳しく見ていく。

<!-- TODO: these が指すのは何か -->

<div class="callout callout-info">
#### 正規表現の意味付けについて Regular Expression Semantics {-}

Our regular expression implementation handles union differently to Scala's built-in regular expressions. Look at the following example comparing the two.

今回の正規表現実装では、和集合の扱いがScalaの組み込み正規表現とは異なる。以下の例で両者の違いを比較してみよう。

```scala mdoc:silent
val r1 = "(z|zxy)ab".r
val r2 = Regexp("z").orElse(Regexp("zxy")) ++ Regexp("ab")
```
```scala mdoc
r1.matches("zxyab")
r2.matches("zxyab")
```

The reason for this difference is that our implementation commits to the first branch in a union that successfully matches some of the input, regardless of how that affects later matching. We should instead try both branches, but doing so makes the implementation more complex. The semantics of regular expressions are not essential to what we're trying to do here; we're just using them as an example to motivate the programming strategies we're learning. I decided the extra complexity of implementing union in the usual way outweighed the benefits, and so kept the simpler implementation. Don't worry, we'll see how to do it properly in the next chapter!

今回の実装では、和集合の中でも最初に入力の一部にマッチしたパターンが採用され、後続の入力に対してより長くマッチするパターンの存在を無視してしまうことが、この違いの理由である。本来は両方のパターンを試すべきだが、それだと実装が複雑化してしまう。ここでは、学ぼうとしているプログラミング戦略の意義を知るための例として正規表現を用いているだけであり、その目的にとって正規表現の意味付けは本質的ではない。一般的な挙動の和集合を実装するための複雑さは利点を上回ると判断し、シンプルな実装にとどめた。だが、次の章では正しい実装方法についても説明することになるだろう。
</div>
