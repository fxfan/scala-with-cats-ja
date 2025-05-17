<!--

## The Reader Monad {#sec:monads:reader}

[`cats.data.Reader`][cats.data.Reader] is a monad
that allows us to sequence operations that depend on some input.
Instances of `Reader` wrap up functions of one argument,
providing us with useful methods for composing them.

One common use for `Readers` is dependency injection.
If we have a number of operations
that all depend on some external configuration,
we can chain them together using a `Reader`
to produce one large operation that
accepts the configuration as a parameter
and runs our program in the order specified.

### Creating and Unpacking Readers

We can create a `Reader[A, B]` from a function `A => B`
using the `Reader.apply` constructor:

```scala mdoc:silent:reset-object
import cats.data.Reader
```

```scala mdoc
final case class Cat(name: String, favoriteFood: String)

val catName: Reader[Cat, String] =
  Reader(cat => cat.name)
```

We can extract the function again
using the `Reader's` `run` method
and call it using `apply` as usual:

```scala mdoc
catName.run(Cat("Garfield", "lasagne"))
```

So far so simple,
but what advantage do `Readers` give us over the raw functions?

### Composing Readers

The power of `Readers` comes from their `map` and `flatMap` methods,
which represent different kinds of function composition.
We typically create a set of `Readers`
that accept the same type of configuration,
combine them with `map` and `flatMap`,
and then call `run` to inject the config at the end.

The `map` method simply extends the computation in the `Reader`
by passing its result through a function:

```scala mdoc:silent
val greetKitty: Reader[Cat, String] =
  catName.map(name => s"Hello ${name}")
```

```scala mdoc
greetKitty.run(Cat("Heathcliff", "junk food"))
```

The `flatMap` method is more interesting.
It allows us to combine readers that depend on the same input type.
To illustrate this, let's extend our greeting example
to also feed the cat:

```scala mdoc:silent
val feedKitty: Reader[Cat, String] =
  Reader(cat => s"Have a nice bowl of ${cat.favoriteFood}")

val greetAndFeed: Reader[Cat, String] =
  for {
    greet <- greetKitty
    feed  <- feedKitty
  } yield s"$greet. $feed."
```

```scala mdoc
greetAndFeed(Cat("Garfield", "lasagne"))
greetAndFeed(Cat("Heathcliff", "junk food"))
```

### Exercise: Hacking on Readers

The classic use of `Readers` is to build programs
that accept a configuration as a parameter.
Let's ground this with a complete example
of a simple login system.
Our configuration will consist of two databases:
a list of valid users and a list of their passwords:

```scala mdoc:silent
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
```

Start by creating a type alias `DbReader` for
a `Reader` that consumes a `Db` as input.
This will make the rest of our code shorter.

<div class="solution">
Our type alias fixes the `Db` type
but leaves the result type flexible:

```scala mdoc:silent
type DbReader[A] = Reader[Db, A]
```
</div>

Now create methods that generate `DbReaders` to
look up the username for an `Int` user ID, and
look up the password for a `String` username.
The type signatures should be as follows:

```scala mdoc:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  ???

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
Remember: the idea is to leave injecting the configuration until last.
This means setting up functions that accept the config as a parameter
and check it against the concrete user info we have been given:

```scala mdoc:invisible:reset-object
import cats.data.Reader
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
type DbReader[A] = Reader[Db, A]
```
```scala mdoc:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))
```

</div>

Finally create a `checkLogin` method
to check the password for a given user ID.
The type signature should be as follows:

```scala mdoc:silent
def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
As you might expect,
here we use `flatMap` to chain `findUsername` and `checkPassword`.
We use `pure` to lift a `Boolean` to a `DbReader[Boolean]`
when the username is not found:

```scala mdoc:invisible:reset-object
import cats.data.Reader
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
type DbReader[A] = Reader[Db, A]
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))
```
```scala mdoc:silent
import cats.syntax.applicative._ // for pure

def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  for {
    username   <- findUsername(userId)
    passwordOk <- username.map { username =>
                    checkPassword(username, password)
                  }.getOrElse {
                    false.pure[DbReader]
                  }
  } yield passwordOk
```
</div>

You should be able to use `checkLogin` as follows:

```scala mdoc:silent
val users = Map(
  1 -> "dade",
  2 -> "kate",
  3 -> "margo"
)

val passwords = Map(
  "dade"  -> "zerocool",
  "kate"  -> "acidburn",
  "margo" -> "secret"
)

val db = Db(users, passwords)
```

```scala mdoc
checkLogin(1, "zerocool").run(db)
checkLogin(4, "davinci").run(db)
```

### When to Use Readers?

`Readers` provide a tool for doing dependency injection.
We write steps of our program as instances of `Reader`,
chain them together with `map` and `flatMap`,
and build a function that accepts the dependency as input.

There are many ways of implementing dependency injection in Scala,
from simple techniques like methods with multiple parameter lists,
through implicit parameters and type classes,
to complex techniques like the cake pattern and DI frameworks.

`Readers` are most useful in situations where:

- we are constructing a program
  that can easily be represented by a function;

- we need to defer injection of a known parameter
  or set of parameters;

- we want to be able to test
  parts of the program in isolation.

By representing the steps of our program as `Readers`
we can test them as easily as pure functions,
plus we gain access to the `map` and `flatMap` combinators.

For more complicated problems where we have lots of dependencies,
or where a program isn't easily represented as a pure function,
other dependency injection techniques tend to be more appropriate.

<div class="callout callout-warning">
  *Kleisli Arrows*

  You may have noticed from console output
  that `Reader` is implemented in terms of another type called `Kleisli`.
  *Kleisli arrows* provide a more general form of `Reader`
  that generalise over the type constructor of the result type.
  We will encounter Kleislis again in Chapter [@sec:monad-transformers].
</div>


```scala mdoc:reset:silent
```
--->

## `Reader` モナド {#sec:monads:reader}

[`cats.data.Reader`][cats.data.Reader] は、何らかの入力に依存する操作を順に連結するためのモナドである。`Reader` のインスタンスは引数をひとつ取る関数をラップし、インスタンス同士を組み合わせるための便利なメソッドを提供する。

`Reader` の一般的な用途のひとつは、依存性注入である。複数の操作がすべて外部の設定に依存する場合、それらを `Reader` で連結してひとつの大きな操作にまとめ、設定をパラメータとして受け取り、指定された順序でプログラムを実行することができる。

### `Reader` の作成と展開

`Reader.apply` コンストラクタを使って `A => B` 型の関数から `Reader[A, B]` を作成できる。

```scala mdoc:silent:reset-object
import cats.data.Reader
```

```scala mdoc
final case class Cat(name: String, favoriteFood: String)

val catName: Reader[Cat, String] =
  Reader(cat => cat.name)
```

`Reader` の `run` で再び関数を取り出すことができ、通常どおり `apply` で呼び出すことができる。

```scala mdoc
catName.run(Cat("Garfield", "lasagne"))
```

今のところ非常にシンプルだが、関数そのままではなく `Reader` を使う利点は何だろうか。

### `Reader` の合成

`Reader` の強力さは、異なる種類の関数の合成を表す `map` および `flatMap` メソッドにある。同じ型の設定を受け取る一連の `Reader` を作成し、`map` や `flatMap` でそれらを組み合わせて最後に `run` を呼び出して設定を注入する、というのがよくある使い方である。

`map` メソッドは、`Reader` の計算結果を指定関数に渡すことで、計算をシンプルに拡張する。

```scala mdoc:silent
val greetKitty: Reader[Cat, String] =
  catName.map(name => s"Hello ${name}")
```

```scala mdoc
greetKitty.run(Cat("Heathcliff", "junk food"))
```

The `flatMap` method is more interesting.
It allows us to combine readers that depend on the same input type.
To illustrate this, let's extend our greeting example
to also feed the cat:

`flatMap` メソッドはもっとおもしろい働きをする。同じ入力型に依存する `Reader` を組み合わせることができる。これを示すために、あいさつの例を、つづけて猫の餌やりもするように拡張してみよう。

```scala mdoc:silent
val feedKitty: Reader[Cat, String] =
  Reader(cat => s"Have a nice bowl of ${cat.favoriteFood}")

val greetAndFeed: Reader[Cat, String] =
  for {
    greet <- greetKitty
    feed  <- feedKitty
  } yield s"$greet. $feed."
```

```scala mdoc
greetAndFeed(Cat("Garfield", "lasagne"))
greetAndFeed(Cat("Heathcliff", "junk food"))
```

### 演習: `Reader` をハックする

`Reader` のよくある使い方は、設定をパラメータとして受け取るプログラムを作成するというものである。ここでは、シンプルだがひととおりの機能を備えたログインシステムの例を用いて理解を深めよう。設定は、有効なユーザのリストとそのパスワードのリストというふたつのデータベースから構成される。

```scala mdoc:silent
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
```

はじめに、`Db` を入力として受け取る `Reader` の型エイリアス `DbReader` を作成せよ。こうしておけば以降のコードを短くできる。

<div class="solution">
この型エイリアスは入力を `Db` 型に固定するが、結果型は指定可能なままにしておく。

```scala mdoc:silent
type DbReader[A] = Reader[Db, A]
```
</div>

`Int` 型のユーザIDからユーザ名を検索する `DbReader` を生成するメソッドと、`String` 型のユーザ名からパスワードを検索する `DbReader` を生成するメソッドを、それぞれ作成せよ。型シグネチャは以下のとおりとする。

```scala mdoc:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  ???

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
この考え方のもとでは設定の注入を行うのは一番最後なのだということを思い起こしてほしい。つまり、設定をパラメータとして受け取り、それを与えられた具体的なユーザー情報と照らし合わせる関数を用意する必要がある。

```scala mdoc:invisible:reset-object
import cats.data.Reader
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
type DbReader[A] = Reader[Db, A]
```
```scala mdoc:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))
```

</div>

最後に、与えられたユーザIDのパスワードを確認する `checkLogin` メソッドを作成せよ。型シグネチャは以下のとおりとする。

```scala mdoc:silent
def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
すでに予想されていたかもしれないが、ここでは `flatMap` を使って `findUsername` と `checkPassword` を連結する。ユーザー名が見つからなかった場合は `pure` を使って `Boolean` を `DbReader[Boolean]` にもち上げている。

```scala mdoc:invisible:reset-object
import cats.data.Reader
final case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
type DbReader[A] = Reader[Db, A]
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))
```
```scala mdoc:silent
import cats.syntax.applicative._ // pure

def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  for {
    username   <- findUsername(userId)
    passwordOk <- username.map { username =>
                    checkPassword(username, password)
                  }.getOrElse {
                    false.pure[DbReader]
                  }
  } yield passwordOk
```
</div>

`checkLogin` は以下のように利用できる。

```scala mdoc:silent
val users = Map(
  1 -> "dade",
  2 -> "kate",
  3 -> "margo"
)

val passwords = Map(
  "dade"  -> "zerocool",
  "kate"  -> "acidburn",
  "margo" -> "secret"
)

val db = Db(users, passwords)
```

```scala mdoc
checkLogin(1, "zerocool").run(db)
checkLogin(4, "davinci").run(db)
```

### どんなときに `Reader` を使うか

`Reader` は依存性注入を行うためのツールを提供する。プログラムの各ステップを `Reader` インスタンスとして記述し、それらを `map` と `flatMap` で連結し、依存性を入力として受け取る関数を構築する。

Scala には、依存性注入を実装する方法が数多く存在する。メソッドに複数のパラメータリストを定義する単純な手法から、暗黙のパラメータや型クラス、そして Cake パターンや DI フレームワークのような複雑な技術まで様々である。

`Reader` は以下のような状況でもっともその真価を発揮する。

- 関数で簡単に表現できるプログラムを構築しているとき
- 既知のパラメータもしくは一連のパラメータの注入を後回しにしたいとき
- プログラムの一部を他と切り離してテストできるようにしたいとき

プログラムのステップを `Reader` として表現することで、純粋関数のように簡単にテストでき、さらに `map` と `flatMap` というコンビネータを利用できるようになる。

For more complicated problems where we have lots of dependencies,
or where a program isn't easily represented as a pure function,
other dependency injection techniques tend to be more appropriate.

たくさんの依存関係をもっている、もしくはプログラムを純粋関数として表現しにくい複雑な問題の場合は、他の依存性注入の手法の方が適していることが多い。

<div class="callout callout-warning">
  *クライスリ射*

  コンソール出力から気付いたかもしれないが、`Reader` は `Kleisli` という別の型を使って実装されている。*クライスリ射（Kleisli arrow）* は、結果型の型コンストラクタを抽象化した `Reader` のより一般的な形式を表す。クライスリについては[@sec:monad-transformers]章で再び取り上げる。
</div>
