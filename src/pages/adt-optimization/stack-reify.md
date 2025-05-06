<!--

## From Continuations to Stacks

In the previous section we explored regular expression derivatives. We saw that they are continuations, but reified as data structures rather than the functions we used when we first worked with continuation-passing style. In this section we'll reify continuations-as-functions as data. In doing so we'll find continuations implicitly encode a stack structure. Explicitly reifying this structure is a step towards implementing a stack machine.

We'll start with the CPSed regular expression interpreter (not using derivatives), shown below.

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
    // Define a type alias so we can easily write continuations
    type Continuation = Option[Int] => Option[Int]

    def loop(regexp: Regexp, idx: Int, cont: Continuation): Option[Int] =
      regexp match {
        case Append(left, right) =>
          val k: Continuation = _ match {
            case None    => cont(None)
            case Some(i) => loop(right, i, cont)
          }
          loop(left, idx, k)

        case OrElse(first, second) =>
          val k: Continuation = _ match {
            case None => loop(second, idx, cont)
            case some => cont(some)
          }
          loop(first, idx, k)

        case Repeat(source) =>
          val k: Continuation =
            _ match {
              case None    => cont(Some(idx))
              case Some(i) => loop(regexp, i, cont)
            }
          loop(source, idx, k)

        case Apply(string) =>
          cont(Option.when(input.startsWith(string, idx))(idx + string.size))

        case Empty =>
          cont(None)
      }

    // Check we matched the entire input
    loop(this, 0, identity).map(idx => idx == input.size).getOrElse(false)
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

To reify the continuations we can apply the same recipe as before: we create a case for each place in which we construct a continuation.
In our interpreter loop this is for `Append`, `OrElse`, and `Repeat`. We also construct a continuation using the identity function when we first call `loop`, which represents the continuation to call when the loop has finished. This gives us four cases.

```scala
enum Continuation {
  case AppendK
  case OrElseK
  case RepeatK
  case DoneK
}
```

What data does each case next to hold?
Let's let look at the structure of the cases within the CPS interpreter.
The case for `Append` is typical.

```scala
case Append(left, right) =>
  val k: Cont = _ match {
    case None    => cont(None)
    case Some(i) => loop(right, i, cont)
  }
  loop(left, idx, k)
```

The continuation `k` refers to the `Regexp` `right`, the method `loop`, and the continuation `cont`.
Our reification should reflect this by holding the same data.
If we consider all the cases we end up with the following definition.
Notice that I implemented an `apply` method so we can still call these continuations like a function.

```scala
type Loop = (Regexp, Int, Continuation) => Option[Int]
enum Continuation {
  case AppendK(right: Regexp, loop: Loop, next: Continuation)
  case OrElseK(second: Regexp, index: Int, loop: Loop, next: Continuation)
  case RepeatK(regexp: Regexp, index: Int, loop: Loop, next: Continuation)
  case DoneK

  def apply(idx: Option[Int]): Option[Int] =
    this match {
      case AppendK(right, loop, next) =>
        idx match {
          case None    => next(None)
          case Some(i) => loop(right, i, next)
        }

      case OrElseK(second, index, loop, next) =>
        idx match {
          case None => loop(second, index, next)
          case some => next(some)
        }

      case RepeatK(regexp, index, loop, next) =>
        idx match {
          case None    => next(Some(index))
          case Some(i) => loop(regexp, i, next)
        }

      case DoneK =>
        idx
    }
}
```

Now we can rewrite the interpreter loop using the `Continuation` type.

```scala
def matches(input: String): Boolean = {
  def loop(
      regexp: Regexp,
      idx: Int,
      cont: Continuation
  ): Option[Int] =
    regexp match {
      case Append(left, right) =>
        val k: Continuation = AppendK(right, loop, cont)
        loop(left, idx, k)

      case OrElse(first, second) =>
        val k: Continuation = OrElseK(second, idx, loop, cont)
        loop(first, idx, k)

      case Repeat(source) =>
        val k: Continuation = RepeatK(regexp, idx, loop, cont)
        loop(source, idx, k)

      case Apply(string) =>
        cont(Option.when(input.startsWith(string, idx))(idx + string.size))

      case Empty =>
        cont(None)
    }

  // Check we matched the entire input
  loop(this, 0, DoneK)
    .map(idx => idx == input.size)
    .getOrElse(false)
}
```

The point of this construction is that we've reified the stack: it's now explicitly represented as the `next` field in each `Continuation`. The stack is a last-in first-out (LIFO) data structure: the last element we add to the stack is the first element we use. (This is exactly the same as efficient use of a `List`.) We construct continuations by adding elements to the front of the existing continuation, which is exactly how we construct lists or stacks. We use continuations from front-to-back; in other words in last-in first-out (LIFO) order. This is the correct access pattern to use a list efficiently, and also the access pattern that defines a stack. Reifying the continuations as data has reified the stack. In the next section we'll use this fact to build a compiler that targets a stack machine.


```scala mdoc:reset:silent
```
--->

## 継続からスタックへ

前節では正規表現の微分について探求し、微分の結果が継続であることを見てきた。ただし、最初に継続渡しスタイルを扱ったときのように関数としてではなく、データ構造としてレイフィケーションされている点が特徴的であった。本節では、関数としての継続をデータとしてレイフィケーションする。この過程で、継続が暗黙的にスタック構造をエンコードしていることがわかるだろう。この構造を明示的にレイフィケーションすることは、スタックマシンの実装に向けたステップとなる。

まずは、以下に示すように、微分を使わない継続渡しスタイルによる正規表現インタープリタから始めよう。

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
    // 継続を書きやすくするために型エイリアスを定義
    type Continuation = Option[Int] => Option[Int]

    def loop(regexp: Regexp, idx: Int, cont: Continuation): Option[Int] =
      regexp match {
        case Append(left, right) =>
          val k: Continuation = _ match {
            case None    => cont(None)
            case Some(i) => loop(right, i, cont)
          }
          loop(left, idx, k)

        case OrElse(first, second) =>
          val k: Continuation = _ match {
            case None => loop(second, idx, cont)
            case some => cont(some)
          }
          loop(first, idx, k)

        case Repeat(source) =>
          val k: Continuation =
            _ match {
              case None    => cont(Some(idx))
              case Some(i) => loop(regexp, i, cont)
            }
          loop(source, idx, k)

        case Apply(string) =>
          cont(Option.when(input.startsWith(string, idx))(idx + string.size))

        case Empty =>
          cont(None)
      }

    // 入力全体にマッチするかどうかチェック
    loop(this, 0, identity).map(idx => idx == input.size).getOrElse(false)
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

継続のレイフィケーションにあたってはこれまでと同じ手順を適用することができる。すなわち、継続を構築するそれぞれの場面に対応するケースを作成する。これはインタープリタのループにおける `Append`、`OrElse`、`Repeat` に該当する。また、最初に `loop` を呼び出す際には恒等関数を使用して継続を構築する。これはループが終了したときに呼び出す継続を表す。以上を踏まえると、次の4つのケースが得られる。

```scala
enum Continuation {
  case AppendK
  case OrElseK
  case RepeatK
  case DoneK
}
```

各ケースが次に保持するデータは何だろうか。継続渡しスタイルインタープリタ内のケース構造を見てみよう。`Append` の構造が典型的である。

```scala
case Append(left, right) =>
  val k: Cont = _ match {
    case None    => cont(None)
    case Some(i) => loop(right, i, cont)
  }
  loop(left, idx, k)
```

継続 `k` は、`Regex` の `right`、`loop` 関数、および継続 `cont` を参照している。今やろうとしているレイフィケーションでは、これらと同じデータを保持することでこの構造を反映させる必要がある。すべてのケースについて同じように考えると、以下のような定義にたどり着く。`apply` メソッドを実装したことで、引き続き関数のように継続を呼び出せる点にも注目してほしい。

```scala
type Loop = (Regexp, Int, Continuation) => Option[Int]
enum Continuation {
  case AppendK(right: Regexp, loop: Loop, next: Continuation)
  case OrElseK(second: Regexp, index: Int, loop: Loop, next: Continuation)
  case RepeatK(regexp: Regexp, index: Int, loop: Loop, next: Continuation)
  case DoneK

  def apply(idx: Option[Int]): Option[Int] =
    this match {
      case AppendK(right, loop, next) =>
        idx match {
          case None    => next(None)
          case Some(i) => loop(right, i, next)
        }

      case OrElseK(second, index, loop, next) =>
        idx match {
          case None => loop(second, index, next)
          case some => next(some)
        }

      case RepeatK(regexp, index, loop, next) =>
        idx match {
          case None    => next(Some(index))
          case Some(i) => loop(regexp, i, next)
        }

      case DoneK =>
        idx
    }
}
```

これで、インタープリタのループを `Continuation` 型を用いて書き換えることが可能となる。

```scala
def matches(input: String): Boolean = {
  def loop(
      regexp: Regexp,
      idx: Int,
      cont: Continuation
  ): Option[Int] =
    regexp match {
      case Append(left, right) =>
        val k: Continuation = AppendK(right, loop, cont)
        loop(left, idx, k)

      case OrElse(first, second) =>
        val k: Continuation = OrElseK(second, idx, loop, cont)
        loop(first, idx, k)

      case Repeat(source) =>
        val k: Continuation = RepeatK(regexp, idx, loop, cont)
        loop(source, idx, k)

      case Apply(string) =>
        cont(Option.when(input.startsWith(string, idx))(idx + string.size))

      case Empty =>
        cont(None)
    }

  // 入力全体にマッチするかどうかチェック
  loop(this, 0, DoneK)
    .map(idx => idx == input.size)
    .getOrElse(false)
}
```

この構造の要点は、スタックをレイフィケーションしたことにある。スタックは各 `Continuation` の `next` フィールドとして明示的に表現されている。スタックは後入れ先出し（LIFO）のデータ構造であり、スタックに最後に追加した要素が最初に使用される（これは `List` の効率的な使用法とまったく同じである）。継続は既存の継続の先頭に要素を追加することで構築されるが、これはリストやスタックを構築する方法とまったく同じである。そして継続は先頭から順に使用される。つまり、継続は後入れ先出し（LIFO）である。このアクセスパターンはリストを効率的に使用するための正しい方法であり、同時にスタックを定義するアクセスパターンでもある。継続をデータとしてレイフィケーションすることで、スタックもレイフィケーションされたことになる。次節では、この事実を利用してスタックマシンをターゲットとするコンパイラを構築する。
