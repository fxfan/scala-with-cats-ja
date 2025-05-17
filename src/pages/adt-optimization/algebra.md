<!--

## Algebraic Manipulation

Reifying a program represents it as a data structure. We can **rewrite** this data structure to several ends: as a way to simplify and therefore optimize the program being interpreted, but also as a general form of computation implementing the interpreter. In this section we're going to return to our regular expression example, and show how rewriting can be used perform both of these tasks.

We will use a technique known as regular expression derivatives. Regular expression derivatives provide a simple way to match a regular expression against input (with the correct semantics for union, which you may recall we didn't deal with in the previous chapter). The derivative of a regular expression, with respect to a character, is the regular expression that remains after matching that character. Say we have the regular expression that matches the string `"osprey"`. In our library this would be `Regexp("osprey")`. The derivative with respect to the character `o` is `Regexp("sprey")`. In other words it's the regular expression that is looking for the string `"sprey"`. The derivative with respect to the character `a` is the regular expression that matches nothing, which is written `Regexp.empty` in our library. To take a more complicated example, the derivative with respect to `c` of `Regexp("cats").repeat` is `Regexp("ats") ++ Regexp("cats").repeat`. This indicates we're looking for the string `"ats"` followed by zero or more repeats of `"cats"`

All we need to do to determine if a regular expression matches some input is to calculate successive derivatives with respect to the characters in the input in the order in which they occur. If the resulting regular expression matches the empty string then we have a successful match. Otherwise it has failed to match.

To implement this algorithm we need three things:

1. an explicit representation of the regular expression that matches the empty string;
2. a method that tests if a regular expression matches the empty string; and
3. a method that computes the derivative of a regular expression with respect to a given character.

Our starting point is the basic reified interpreter we developed in the previous chapter. 
This is the simplest code and therefore the easiest to work with.

```scala mdoc:silent
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

    // Check we matched the entire input
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

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

We want to explicitly represent the regular expression that matches the empty string, as it plays an important part in the algorithms that follow. 
This is simple to do: we just reify it and adjust the constructors as necessary.
I've called this case "epsilon", which matches the terminology used in the literature.

```scala
enum Regexp {
  // ...
  case Epsilon
}
object Regexp {
  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

Next up we will create a predicate that tells us if a regular expression matches the empty string. Such a regular expression is called "nullable". The code is so simple it's easier to read it than try to explain it in English.

```scala
def nullable: Boolean =
  this match {
    case Append(left, right) => left.nullable && right.nullable
    case OrElse(first, second) => first.nullable || second.nullable
    case Repeat(source) => true
    case Apply(string) => false
    case Epsilon => true
    case Empty => false
  }
```

Now we can implement the actual regular expression derivative.
It consists of two parts: the method to calculate the derivative which in turn depends on a method that handles a nullable regular expression. Both parts are quite simple so I'll give the code first and then explain the more complicated parts.

```scala
def delta: Regexp =
  if nullable then Epsilon else Empty

def derivative(ch: Char): Regexp =
  this match {
    case Append(left, right) =>
      (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
    case OrElse(first, second) =>
      first.derivative(ch).orElse(second.derivative(ch))
    case Repeat(source) =>
      source.derivative(ch) ++ this
    case Apply(string) =>
      if string.size == 1 then
        if string.charAt(0) == ch then Epsilon
        else Empty
      else if string.charAt(0) == ch then Apply(string.tail)
      else Empty
    case Epsilon => Empty
    case Empty => Empty
  }
```

I think this code is reasonably straightforward, except perhaps for the cases for `OrElse` and `Append`. The case for `OrElse` is trying to match both regular expressions simultaneously, which gets around the problem in our earlier implementation. The definition of `nullable` ensures we match if either side matches. The case for `Append` is attempting to match the `left` side if it is still looking for characters; otherwise it is attempting to match the `right` side.

With this we redefine `matches` as follows.

```scala
def matches(input: String): Boolean = {
  val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
  r.nullable
}
```

```scala mdoc:reset:invisible
enum Regexp {
  def ++(that: Regexp): Regexp = {
    Append(this, that)
  }

  def orElse(that: Regexp): Regexp = {
    OrElse(this, that)
  }

  def repeat: Regexp = {
    Repeat(this)
  }

  def `*` : Regexp = this.repeat

  /** True if this regular expression accepts the empty string */
  def nullable: Boolean =
    this match {
      case Append(left, right)   => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source)        => true
      case Apply(string)         => false
      case Epsilon               => true
      case Empty                 => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right)
          .orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty   => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this) { (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

We can show the code works as expected.

```scala mdoc:silent
val regexp = Regexp("Sca") ++ Regexp("la") ++ Regexp("la").repeat
```
```scala mdoc
regexp.matches("Scala")
regexp.matches("Scalalalala")
regexp.matches("Sca")
regexp.matches("Scalal")
```

It also solves the problem with the earlier implementation.

```scala mdoc
Regexp("cat").orElse(Regexp("cats")).matches("cats")
```

This is a nice result for a very simple algorithm.
However there is a problem.
You might notice that regular expression matching can become very slow. 
In fact we can run out of heap space trying a simple match like

```scala
Regexp("cats").repeat.matches("catscatscatscats")
// java.lang.OutOfMemoryError: Java heap space
```

This happens because the derivative of the regular expression can grow very large.
Look at this example, after only a few derivatives.

```scala mdoc:to-string
Regexp("cats").repeat.derivative('c').derivative('a').derivative('t')
```

The root cause is that the derivative rules for `Append`, `OrElse`, and `Repeat` can produce a regular expression that is larger than the input. However this output often contains redundant information. In the example above there are multiple occurrences of `Append(Empty, ...)`, which is equivalent to just `Empty`. This is similar to adding zero or multiplying by one in arithmetic, and we can use similar algebraic simplification rules to get rid of these unnecessary elements.

We can implement this simplification in one of two ways: we can make simplification a separate method that we apply to an existing `Regexp`, or we can do the simplification as we construct the `Regexp`. I've chosen to do the latter, modifying `++`, `orElse`, and `repeat` as follows:

```scala
def ++(that: Regexp): Regexp = {
  (this, that) match {
    case (Epsilon, re2) => re2
    case (re1, Epsilon) => re1
    case (Empty, _) => Empty
    case (_, Empty) => Empty
    case _ => Append(this, that)
  }
}

def orElse(that: Regexp): Regexp = {
  (this, that) match {
    case (Empty, re) => re
    case (re, Empty) => re
    case _ => OrElse(this, that)
  }
}

def repeat: Regexp = {
  this match {
    case Repeat(source) => this
    case Epsilon => Epsilon
    case Empty => Empty
    case _ => Repeat(this)
  }
}
```

With this small change in-place, our regular expressions stay at a reasonable size for any input.

```scala mdoc:reset:invisible
enum Regexp {
  def ++(that: Regexp): Regexp = {
    (this, that) match {
      case (Epsilon, re2) => re2
      case (re1, Epsilon) => re1
      case (Empty, _) => Empty
      case (_, Empty) => Empty
      case _ => Append(this, that)
    }
  }

  def orElse(that: Regexp): Regexp = {
    (this, that) match {
      case (Empty, re) => re
      case (re, Empty) => re
      case _ => OrElse(this, that)
    }
  }

  def repeat: Regexp = {
    this match {
      case Repeat(source) => this
      case Epsilon => Epsilon
      case Empty => Empty
      case _ => Repeat(this)
    }
  }

  def `*` : Regexp = this.repeat

  /** True if this regular expression accepts the empty string */
  def nullable: Boolean =
    this match {
      case Append(left, right) => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source) => true
      case Apply(string) => false
      case Epsilon => true
      case Empty => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

```scala mdoc:to-string
Regexp("cats").repeat.derivative('c').derivative('a').derivative('t')
```

Here's the final code.

```scala mdoc:reset:silent
enum Regexp {
  def ++(that: Regexp): Regexp = {
    (this, that) match {
      case (Epsilon, re2) => re2
      case (re1, Epsilon) => re1
      case (Empty, _) => Empty
      case (_, Empty) => Empty
      case _ => Append(this, that)
    }
  }

  def orElse(that: Regexp): Regexp = {
    (this, that) match {
      case (Empty, re) => re
      case (re, Empty) => re
      case _ => OrElse(this, that)
    }
  }

  def repeat: Regexp = {
    this match {
      case Repeat(source) => this
      case Epsilon => Epsilon
      case Empty => Empty
      case _ => Repeat(this)
    }
  }

  def `*` : Regexp = this.repeat

  /** True if this regular expression accepts the empty string */
  def nullable: Boolean =
    this match {
      case Append(left, right) => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source) => true
      case Apply(string) => false
      case Epsilon => true
      case Empty => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

Notice that our implementation is tail recursive. The only "looping" is the call to the tail recursive `foldLeft` in `matches`. No continuation-passing style transform is necessary here! (Calculating the derivatives is not tail recursive but it very unlikely this would overflow the stack.) This may not be surprising if you've studied theory of computation. A key result from that field is the equivalence between regular expressions and finite state machines. If you know this you may have found it a bit surprising we had to use a stack at all in our prior implementations. But hold on a minute. If we think carefully about regular expression derivatives we'll see that they actually are continuations! A continuation means "what comes next", which is exactly what a regular expression derviative defines for a regular expression and a particular character. So our interpreter does use CPS, but reified as a regular expression not a function, and derived through a different route.

Continuations reify control-flow. That is, they give us an explicit representation of how control moves through our program. This means we can change the control flow by applying continuations in a different order. Let's make this concrete. A regular expression derivative represents a continuation. So imagine we're running a regular expression on data that arrives asynchronously; we want to match as much data as we have available, and then suspend the regular expression and continue matching when more data arrives. This is trival. When we run out of data we just store the current derivative. When more data arrives we continue processing using the derivative we stored. Here's an example.

Start by defining the regular expression.

```scala mdoc:silent
val cats = Regexp("cats").repeat
```

Process the first piece of data and store the continuation.

```scala mdoc:silent
val next = "catsca".foldLeft(cats){ (regexp, ch) => regexp.derivative(ch) }
```

Continue processing when more data arrives.

```scala mdoc:silent
"tscats".foldLeft(next){ (regexp, ch) => regexp.derivative(ch) }
```

Notice that we could just as easily go back to a previous regular expression if we wanted to. This would give us backtracking. We don't need backtracking for regular expressions, but for more general parsers we do. In fact with continuations we can define any control flow we like, including backtracking search, exceptions, cooperative threading, and much much more.

In this section we've also seen the power of rewrites. Regular expression matching using derivatives works solely by rewriting the regular expression. We also used rewriting to simplify the regular expressions, avoiding the explosion in size that derivatives can cause.
The abstract type of these methods is `Program => Program` so we might think they are combinators. However the implementation uses structural recursion and they serve the role of interpreters. Rewrites are the one place where the types alone can lead us astray.

I hope you find regular expression derivatives interesting and a bit surprising. I certainly did when I first read about them. There is a deeper point here, which runs throughout the book: most problems have already been solved and we can save a lot of time if we can just find those solutions. I elevate this idea of the status of a strategy, which I call **read the literature** for reasons that will soon be clear. Most developers read the occasional blog post and might attend a conference from time to time. Many fewer, I think, read academic papers. This is unfortunate. Part of the fault is with the academics: they write in a style that is hard to read without some practice. However I think many developers think the academic literature is irrelevant. One of the goals of this book is to show the relevance of academic work, which is why each chapter conclusion sketches the development of its main ideas with links to relevant papers.


```scala mdoc:reset:silent
```
--->

## 代数的操作

プログラムのレイフィケーションとは、プログラムをデータ構造として表現することである。このデータ構造は**書き換え**が可能である。この書き換えとは、解釈されるプログラムを簡略化し最適化する方法であり、またインタープリタを実装する計算の一般的な形式でもある。この節では正規表現の例に戻り、これらふたつのタスクを実行するにあたって書き換えがどのように利用されるかを示す。

ここでは正規表現の微分と呼ばれる技法を使用する。正規表現の微分は、入力に対する正規表現のマッチングを簡単に行う方法を提供してくれる。この手法では、以前の章では扱わなかった、和集合に対する正しいセマンティクスも考慮される。

正規表現をある文字で微分した結果は、その文字にマッチした後に残る正規表現である。たとえば、文字列 `"osprey"` にマッチする正規表現があるとしよう。以前の章で作成したライブラリを使えば、これは `Regexp("osprey")` と表せる。この正規表現を文字 `o` で微分した結果は `Regexp("sprey")`、つまり文字列 `"sprey"` を探す正規表現である。一方、文字 `a` で微分した結果はどんな文字にもマッチしない正規表現で、`Regexp.empty` で表される。さらに複雑な例として、`Regexp("cats").repeat` を `c` で微分した結果は `Regexp("ats") ++ Regexp("cats").repeat` となる。これは、文字列 `"ats"` とその後に続く `"cats"` の0回以上の繰り返しを探す正規表現である。

正規表現がある入力にマッチするかを判定するには、入力に含まれる一つひとつの文字で順に微分するだけでよい。最終的に得られた正規表現が空文字列にマッチすれば成功、それ以外の場合はマッチに失敗したことになる。

このアルゴリズムを実装するために必要なのは以下の三つである。

1. 空文字列にマッチする正規表現の明示的な表現
2. 正規表現が空文字列にマッチするかをテストするメソッド
3. 正規表現を特定の文字で微分するメソッド

まずは以前の章で開発した基本的なインタープリタから始めよう。これがもっともシンプルなコードであり、ベースとして扱うのに都合がよい。

```scala mdoc:silent
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

    // 入力全体にマッチしたかどうかをチェックする
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

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

空文字列にマッチする正規表現を明示的に表現したい。これは、この後のアルゴリズムで重要な役割を果たす。これを実現するのは簡単である。レイフィケーションを行い、必要に応じてコンストラクタを調整するだけでよい。このケースのことを、文献で用いられている用語に合わせて「epsilon」と呼ぶことにする。

```scala
enum Regexp {
  // ...
  case Epsilon
}
object Regexp {
  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

次に、正規表現が空文字列にマッチするかどうかを判定する述語関数を作成する。そのような正規表現は「 nullable である」と表現される。コードは非常にシンプルなので、言葉で説明するよりもコードを読んだほうが理解しやすいだろう。

```scala
def nullable: Boolean =
  this match {
    case Append(left, right) => left.nullable && right.nullable
    case OrElse(first, second) => first.nullable || second.nullable
    case Repeat(source) => true
    case Apply(string) => false
    case Epsilon => true
    case Empty => false
  }
```

これで正規表現の微分の本体ロジックを実装できる。これはふたつの部分からなる。微分を行うメソッドと、nullable な正規表現を処理するメソッドで、前者は後者に依存している。どちらも比較的シンプルなので、まずコードを示し、その後、複雑な部分について説明することにする。

```scala
def delta: Regexp =
  if nullable then Epsilon else Empty

def derivative(ch: Char): Regexp =
  this match {
    case Append(left, right) =>
      (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
    case OrElse(first, second) =>
      first.derivative(ch).orElse(second.derivative(ch))
    case Repeat(source) =>
      source.derivative(ch) ++ this
    case Apply(string) =>
      if string.size == 1 then
        if string.charAt(0) == ch then Epsilon
        else Empty
      else if string.charAt(0) == ch then Apply(string.tail)
      else Empty
    case Epsilon => Empty
    case Empty => Empty
  }
```

このコードは概ねわかりやすいが、`OrElse` と `Append` のケースはすこしわかりにくいかもしれない。`OrElse` のケースでは、両方の正規表現に同時にマッチを試みており、これにより前回の実装での問題が解消される[^tn-adt-optimization-algebra-01]。`nullable` の定義では、`OrElse` のどちらか一方が空文字列にマッチすれば `OrElse` 自体もマッチすることが保証されている。`Append` のケースでは、`left` 側がまだ文字を探している場合には `left` とのマッチを試み、そうでなければ `right` 側とのマッチを試みる。

[^tn-adt-optimization-algebra-01]: 【訳注】 前回の実装では、和集合においては最初にマッチしたパターンが常に採用され、それ以外のパターンを選んだほうが後続の入力についてより長くマッチするケースを考慮できていなかった。そのため、たとえば `(z|zxy)ab` という正規表現に対して `zxyab` という入力がマッチしない、という問題があった。

これを用いて `matches` を次のように再定義する。

```scala
def matches(input: String): Boolean = {
  val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
  r.nullable
}
```

```scala mdoc:reset:invisible
enum Regexp {
  def ++(that: Regexp): Regexp = {
    Append(this, that)
  }

  def orElse(that: Regexp): Regexp = {
    OrElse(this, that)
  }

  def repeat: Regexp = {
    Repeat(this)
  }

  def `*` : Regexp = this.repeat

  /** 正規表現が空文字にマッチするなら true */
  def nullable: Boolean =
    this match {
      case Append(left, right)   => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source)        => true
      case Apply(string)         => false
      case Epsilon               => true
      case Empty                 => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right)
          .orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty   => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this) { (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

このコードが期待どおり動作することを以下に示しておく。

```scala mdoc:silent
val regexp = Regexp("Sca") ++ Regexp("la") ++ Regexp("la").repeat
```
```scala mdoc
regexp.matches("Scala")
regexp.matches("Scalalalala")
regexp.matches("Sca")
regexp.matches("Scalal")
```

今回の実装は、前回の実装が抱えていた課題も解決している。

```scala mdoc
Regexp("cat").orElse(Regexp("cats")).matches("cats")
```

これは非常にシンプルなアルゴリズムとしてはよい結果だが、ひとつ問題がある。すでに気付いているかもしれないが、正規表現のマッチングが非常に遅くなる可能性がある。実際、以下のような簡単なマッチングを試みるだけでもヒープ領域が不足してしまうことがある。

```scala
Regexp("cats").repeat.matches("catscatscatscats")
// java.lang.OutOfMemoryError: Java heap space
```

これは、正規表現の微分結果が非常に大きくなることによって起こる。以下の例を見てほしい。ほんの数回微分しただけでこうなる。

```scala mdoc:to-string
Regexp("cats").repeat.derivative('c').derivative('a').derivative('t')
```

OrElse(
  OrElse(
    Append(Apply(s),Repeat(Apply(cats))),
    Append(Empty,Append(Empty,Repeat(Apply(cats))))
  ),OrElse(
    Append(Empty,Append(Empty,Repeat(Apply(cats)))),
    Append(
      Empty,
      OrElse(
        Append(Empty,Repeat(Apply(cats))),
        Append(Empty,Append(Empty,Repeat(Apply(cats))))
      )
    )
  )
)

根本的な原因は、`Append`、`OrElse`、および `Repeat` の微分ルールが、入力よりも大きな正規表現を生成する可能性がある点にある。しかし、この出力には冗長な情報が含まれている場合が多い。上記の例では `Append(Empty, ...)` が複数回出現しているが、これはただの `Empty` と等価である。これは算術演算でゼロを加えたり1を掛けたりするのと似ている。算術演算と同じような代数的簡略化を用いることで、こういった不要な要素を取り除くことができる。

この簡略化を実装する方法はふたつある。ひとつは既存の `Regexp` に適用する独立した簡略化メソッドを作成する方法、もうひとつは `Regexp` を構築する際に簡略化を行う方法である。ここでは後者を選び、`++`、`orElse` および `repeat` を次のように修正する。

```scala
def ++(that: Regexp): Regexp = {
  (this, that) match {
    case (Epsilon, re2) => re2
    case (re1, Epsilon) => re1
    case (Empty, _) => Empty
    case (_, Empty) => Empty
    case _ => Append(this, that)
  }
}

def orElse(that: Regexp): Regexp = {
  (this, that) match {
    case (Empty, re) => re
    case (re, Empty) => re
    case _ => OrElse(this, that)
  }
}

def repeat: Regexp = {
  this match {
    case Repeat(source) => this
    case Epsilon => Epsilon
    case Empty => Empty
    case _ => Repeat(this)
  }
}
```

この小さな変更を加えることで、正規表現はどのような入力に対しても適切なサイズに保たれる。

```scala mdoc:reset:invisible
enum Regexp {
  def ++(that: Regexp): Regexp = {
    (this, that) match {
      case (Epsilon, re2) => re2
      case (re1, Epsilon) => re1
      case (Empty, _) => Empty
      case (_, Empty) => Empty
      case _ => Append(this, that)
    }
  }

  def orElse(that: Regexp): Regexp = {
    (this, that) match {
      case (Empty, re) => re
      case (re, Empty) => re
      case _ => OrElse(this, that)
    }
  }

  def repeat: Regexp = {
    this match {
      case Repeat(source) => this
      case Epsilon => Epsilon
      case Empty => Empty
      case _ => Repeat(this)
    }
  }

  def `*` : Regexp = this.repeat

  /** 正規表現が空文字にマッチするなら true */
  def nullable: Boolean =
    this match {
      case Append(left, right) => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source) => true
      case Apply(string) => false
      case Epsilon => true
      case Empty => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

```scala mdoc:to-string
Regexp("cats").repeat.derivative('c').derivative('a').derivative('t')
```

最終的なコードを以下に示す。

```scala mdoc:reset:silent
enum Regexp {
  def ++(that: Regexp): Regexp = {
    (this, that) match {
      case (Epsilon, re2) => re2
      case (re1, Epsilon) => re1
      case (Empty, _) => Empty
      case (_, Empty) => Empty
      case _ => Append(this, that)
    }
  }

  def orElse(that: Regexp): Regexp = {
    (this, that) match {
      case (Empty, re) => re
      case (re, Empty) => re
      case _ => OrElse(this, that)
    }
  }

  def repeat: Regexp = {
    this match {
      case Repeat(source) => this
      case Epsilon => Epsilon
      case Empty => Empty
      case _ => Repeat(this)
    }
  }

  def `*` : Regexp = this.repeat

  /** 正規表現が空文字にマッチするなら true */
  def nullable: Boolean =
    this match {
      case Append(left, right) => left.nullable && right.nullable
      case OrElse(first, second) => first.nullable || second.nullable
      case Repeat(source) => true
      case Apply(string) => false
      case Epsilon => true
      case Empty => false
    }

  def delta: Regexp =
    if nullable then Epsilon else Empty

  def derivative(ch: Char): Regexp =
    this match {
      case Append(left, right) =>
        (left.derivative(ch) ++ right).orElse(left.delta ++ right.derivative(ch))
      case OrElse(first, second) =>
        first.derivative(ch).orElse(second.derivative(ch))
      case Repeat(source) =>
        source.derivative(ch) ++ this
      case Apply(string) =>
        if string.size == 1 then
          if string.charAt(0) == ch then Epsilon
          else Empty
        else if string.charAt(0) == ch then Apply(string.tail)
        else Empty
      case Epsilon => Empty
      case Empty => Empty
    }

  def matches(input: String): Boolean = {
    val r = input.foldLeft(this){ (regexp, ch) => regexp.derivative(ch) }
    r.nullable
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Epsilon
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  val epsilon: Regexp = Epsilon

  def apply(string: String): Regexp =
    if string.isEmpty() then Epsilon
    else Apply(string)
}
```

この実装が末尾再帰である点に注目してほしい。唯一の「ループ」は、`matches` 内での末尾再帰的な `foldLeft` の呼び出しだが、ここでは継続渡しスタイルへの変換は不要である（正規表現を微分する部分は末尾再帰ではないが、これによってスタックオーバーフローが起きる可能性は非常に低い）。もし計算理論を学んだことがあれば、このことは驚きではないかもしれない。計算理論の重要な成果のひとつに正規表現と有限状態機械の等価性がある。それについて知っていれば、以前の実装でそもそもスタックを使う必要があったことにすこし驚いたかもしれない。

だがすこし待ってほしい。正規表現の微分についてじっくり考えれば、その結果が実は継続であることがわかる。継続とは「次に何をするか」を表しており、正規表現の微分結果が、ある正規表現とひと文字の入力に対して「次に何とマッチするか」を定義しているのとまさに一致している。つまり、我々のインタープリタは実際には継続渡しスタイルを使っているが、関数ではなく正規表現として具現化され、異なる手法によって導かれているのである。

継続はプログラムの制御フローをレイフィケーションしたものである。つまり、プログラム内で制御がどのように移っていくのかを明示的に表現する。これは、継続を異なる順序で適用することで制御フローを変更できることを意味する。具体的な例を考えてみよう。正規表現の微分結果は継続を表している。たとえば、非同期に到着するデータに対して正規表現とのマッチングを実行するとしよう。利用可能なデータに対してできるかぎりマッチングを行ったのち処理を中断し、新たなデータが到着したらマッチングを続行したい。この処理は簡単である。データがなくなったら、その時点での微分結果を記憶しておくだけでよい。新たなデータが到着したら、記憶しておいた微分結果を使って処理を再開する。以下にその例を示す。

まずは正規表現を定義しよう。

```scala mdoc:silent
val cats = Regexp("cats").repeat
```

データの最初の断片を処理し、継続を保存する。

```scala mdoc:silent
val next = "catsca".foldLeft(cats){ (regexp, ch) => regexp.derivative(ch) }
```

次のデータが届いたら処理を続行する。

```scala mdoc:silent
"tscats".foldLeft(next){ (regexp, ch) => regexp.derivative(ch) }
```

正規表現を以前の状態に戻すことも簡単にできる点に注目してほしい。これによりバックトラッキングを実現できる。正規表現にバックトラッキングは必要ないが、より一般的なパーサには必要である。実際、継続を使用すれば、バックトラッキング検索、例外処理、協調スレッディングなど、任意の制御フローを定義することができる。

また、この節では書き換えの力を目の当たりにした。微分を用いた正規表現のマッチングは、正規表現の書き換えのみによって実現される。また、微分が起こすサイズの爆発を回避するために正規表現を簡略化する際も、書き換えを用いた。これらのメソッドの抽象型は `Program => Program` であり、一見するとコンビネータのように思えるかもしれない。しかし、実装は構造的再帰を使用しており、インタープリタの役割を果たしている。書き換えは、型情報だけでは誤解を招く可能性がある唯一のケースである。

正規表現の微分に面白さとちょっとした驚きを感じてもらえたなら嬉しい。私自身も初めて読んだときに驚いたことを覚えている。ここには本書全体を貫く重要なポイントがある。それは、ほとんどの問題には既に解決策が存在しており、それらを見つけることができれば大幅な時間の節約につながるということである。このアイデアを私は「文献を読む（read the literature）」というひとつの戦略として位置づけている。その理由はすぐに明らかになるだろう。多くの開発者は、たまにブログ記事を読んだり、カンファレンスに参加したりする。しかし、学術論文を読む人はかなり少ないのではないかと思う。これは残念なことである。原因の一部は学術サイドにある。学術論文は、ある程度の訓練なしでは読みづらいスタイルで書かれている。しかし、多くの開発者が学術的な文献を自分と無関係だと考えていることも原因だと思う。本書の目的のひとつは、学術研究と我々が向き合っている課題との関連性を示すことである。そのため各章のまとめでは主要なアイデアの発展について概説し、関連する論文へのリンクを提示している。
