<!--

## The Mechanics of Contextual Abstraction

In section we'll go through the main Scala language features for contextual abstraction. Once we have a firm understanding of the mechanics of contextual abstraction we'll move on to their use.

The language features for contextual abstraction have changed name from Scala 2 to Scala 3, but they work in largely the same way. In the table below I show the Scala 3 features, and their Scala 2 equivalents. If you use Scala 2 you'll find that most of the code works simply by replacing `given` with `implicit val` and `using` with `implicit`.

+------------------+---------------------+
| Scala 3          | Scala 2             |
+==================+=====================+
| given instance   | implicit value      |
+------------------+---------------------+
| using clause     | implicit parameter  |
+------------------+---------------------+

Let's now explain how these language features work.


### Using Clauses

We'll start with **using clauses**. A using clause is a method parameter list that starts with the `using` keyword. We use the term **context parameters** for the parameters in a using clause.

```scala mdoc:silent
def double(using x: Int) = x + x
```

The `using` keyword applies to all parameters in the list, so in `add` below both `x` and `y` are context parameters.

```scala mdoc:silent
def add(using x: Int, y: Int) = x + y
```

We can have normal parameter lists, and multiple using clauses, in the same method.

```scala mdoc:silent
def addAll(x: Int)(using y: Int)(using z: Int): Int =
  x + y + z
```

We cannot pass parameters to a using clause in the normal way. We must proceed the parameters with the `using` keyword as shown below.

```scala mdoc
double(using 1)
add(using 1, 2)
addAll(1)(using 2)(using 3)
```

However this is not the typical way to pass parameters. In fact we don't usually explicit pass parameters to using clause at all. We usually use given instances instead, so let's turn to them.


### Given Instances

A given instance is a value that is defined with the `given` keyword. Here's a simple example.

```scala mdoc:silent
given theMagicNumber: Int = 3
```

We can use a given instance like a normal value.

```scala mdoc:silent
theMagicNumber * 2
```

However, it's more common to use them with a using clause. When we call a method that has a using clause, and we do not explicitly supply values for the context parameters, the compiler will look for given instances of the required type. If it finds a given instance it will automatically use it to complete the method call.

For example, we defined `double` above with a single `Int` context parameter. The given instance we just defined, `theMagicNumber`, also has type `Int`. So if we call `double` without providing any value for the context parameter the compiler will provide the value `theMagicNumber` for us.

```scala mdoc
double
```

The same given instance will be used for multiple parameters in a using clause with the same type, as in `add` defined above.

```scala mdoc
add
```

The above are the most important points for using clauses and given instances. We'll now turn to some of the details of their semantics.


### Given Scope and Imports

Given instances are usually not explicitly passed to using clauses.
Their whole reason for existence is to get the compiler to do this for us.
This could make code hard to understand, so we need to be very clear about which given instances are candidates to be supplied to a using clause.
In this section we'll look at the **given scope**, which is all the places that the compiler will look for given instances, and the special syntax for importing given instances.

The first rule we should know about the given scope is that it starts at the **call site**, where the method with a using clause is called, not at the **definition site** where the method is defined.
This means the following code does not compile, because the given instance is not in scope at the call site, even though it is in scope at the definition site.

```scala mdoc:reset:fail
object A {
  given a: Int = 1
  def whichInt(using int: Int): Int = int
}

A.whichInt
```

The second rule, which we have been relying on in all our examples so far, is that the given scope includes the **lexical scope** at the call site.
The lexical scope is where we usually look up the values associated with names (like the names of method parameters or `val` declarations).
This means the following code works, as `a` is defined in a scope that includes the call site.

```scala mdoc:silent
object A {
  given a: Int = 1
  
  object B {
    C.whichInt 
  }
  
  object C {
    def whichInt(using int: Int): Int = int
  }
}
```

However, if there are multiple given instances in the same scope the compiler will not arbitrarily choose one.
Instead it fails with an error telling us the choice is ambiguous.

```scala
object A {
  given a: Int = 1
  given b: Int = 2
    
  def whichInt(using int: Int): Int = int
    
  whichInt
}
// error:
// Ambiguous given instances: both given instance a in object A and
// given instance b in object A match type Int of parameter int of 
// method whichInt in object A
```

We can import given instances from other scopes, just like we can import normal declarations, but we must explicitly say we want to import given instances. The following code does not work because we have not explicitly imported the given instances.

```scala mdoc:reset:fail
object A {
  given a: Int = 1

  def whichInt(using int: Int): Int = int
}
object B {
  import A.*
    
  whichInt
}
```

It works when we do explicitly import them using `import A.given`.

```scala mdoc:reset:silent
object A {
  given a: Int = 1

  def whichInt(using int: Int): Int = int
}
object B {
  import A.{given, *}
    
  whichInt
}
```

One final wrinkle: the given scope includes the companion objects of any type involved in the type of the using clause. 
This is best illustrated with an example.
We'll start by defining a type `Sound` that represents the sound made by its type variable `A`, and a method `soundOf` to access that sound.

```scala mdoc:reset
trait Sound[A] {
  def sound: String
}

def soundOf[A](using s: Sound[A]): String =
  s.sound
```

Now we'll define some given instances. Notice that they are defined on the relevant companion objects.

```scala mdoc:silent
trait Cat
object Cat {
  given catSound: Sound[Cat] =
    new Sound[Cat]{
      def sound: String = "meow"
    }
}

trait Dog
object Dog {
  given dogSound: Sound[Dog] = 
    new Sound[Dog]{
      def sound: String = "woof"
    }
}
```

When we call `soundOf` we don't have to explicitly bring the instances into scope. They are automatically in the given scope by virtue of being defined on the companion objects of the types we use (`Cat` and `Dog`). If we had defined these instances on the `Sound` companion object they would also be in the given scope; when looking for a `Sound[A]` both the companion objects of `Sound` and `A` are in scope.

```scala mdoc
soundOf[Cat]
soundOf[Dog]
```

We should almost always be defining given instances on companion objects. This simple organization scheme means that users do not have to explicitly import them but can easily find the implementations if they wish to inspect them.


#### Given Instance Priority

Notice that given instance selection is based entirely on types. We don't even pass any values to `soundOf`! This means given instances are easiest to use when there is only one instance for each type. In this case we can just put the instances on a relevant companion object and everything works out.

However, this is not always possible (though it's often an indication of a bad design if it is not). For cases where we need multiple instances for a type, we can use the instance priority rules to select between them. We'll look at the three most important rules below.

The first rule is that explicitly passing an instance takes priority over everything else.

```scala mdoc:reset:silent
given a: Int = 1
def whichInt(using int: Int): Int = int
```
```scala mdoc
whichInt(using 2)
```
   
The second rule is that instances in the lexical scope take priority over instances in a companion object

```scala mdoc:reset:silent
trait Sound[A] {
  def sound: String
}
trait Cat
object Cat {
  given catSound: Sound[Cat] =
    new Sound[Cat]{
      def sound: String = "meow"
    }
}

def soundOf[A](using s: Sound[A]): String =
  s.sound
```
```scala mdoc
given purr: Sound[Cat]  =
  new Sound[Cat]{
    def sound: String = "purr"
  }

soundOf[Cat]
```
   
The final rule is that instances in a closer lexical scope take preference over those further away.

```scala mdoc
{
  given growl: Sound[Cat] =
   new Sound[Cat]{
     def sound: String = "growl"
   }
   
  {
    given mew: Sound[Cat] =
     new Sound[Cat]{
       def sound: String = "mew"
     }
     
    soundOf[Cat]
  }
}
```
   
We're now seen most of the details of how given instances and using clauses work. This is a craft level explanation, and it naturally leads to the question: where would use these tools? This is what we'll address next, where we look at type classes and their implementation in Scala.


```scala mdoc:reset:silent
```
--->

## コンテキスト抽象化のメカニズム

この節では、コンテキスト抽象化に関する Scala の主要な言語機能について説明する。コンテキスト抽象化のメカニズムをしっかり理解した上で、その具体的な使用方法へと進む。

コンテキスト抽象化に関する言語機能の名称は Scala2 から Scala3 で変更されているが、動作はほぼ同じである。以下の表に Scala3 の機能と、それに相当する Scala2 の機能を示した。Scala2 を使用している場合、`given` を `implicit val` に、`using` を `implicit` に置き換えるだけで多くのコードが動作するだろう。

| Scala3           | Scala2              |
|------------------|---------------------|
| given インスタンス | implicit value      |
| using 句         | implicit パラメータ   |

次にこれらの言語機能がどのように動作するか説明しよう。

### using 句

まずは **using 句**から始める。using 句とは `using` キーワードで始まるメソッドパラメータリストのことをいう。using 句内のパラメータのことは**コンテキストパラメータ**という用語で呼ぶ。

```scala mdoc:silent
def double(using x: Int) = x + x
```

`using` キーワードは、そのパラメータリスト内にあるすべてのパラメータに効果を及ぼす。次の `add` 関数では、`x` と `y` の両方がコンテキストパラメータである。

```scala mdoc:silent
def add(using x: Int, y: Int) = x + y
```

同じメソッドに普通のパラメータリストが並存していてもよいし、using 句が複数あってもかまわない。

```scala mdoc:silent
def addAll(x: Int)(using y: Int)(using z: Int): Int =
  x + y + z
```

通常の方法で using 句にパラメータを渡すことはできない。以下に示すように、パラメータの前に `using` キーワードを付けなくてはならない。

```scala mdoc
double(using 1)
add(using 1, 2)
addAll(1)(using 2)(using 3)
```

だが、これは using 句にパラメータを渡す標準的な方法ではない。using 句にパラメータを明示的に渡すことは、実際にはほとんどない。通常は given インスタンスを使用する。次にそれについて説明する。

### given インスタンス

given インスタンスは `given` キーワードを使って定義される値である。次に簡単な例を示す。

```scala mdoc:silent
given theMagicNumber: Int = 3
```

given インスタンスは普通の値のように使うことができる。

```scala mdoc:silent
theMagicNumber * 2
```

だが、using 句と組み合わせる使い方のほうがより一般的である。using 句をもつメソッドを、コンテキストパラメータを明示的に与えずに呼び出した場合、要求される型の given インスタンスをコンパイラが探してくれる。given インスタンスが見つかれば、それが自動的にパラメータとして使用され、メソッド呼び出しは完成する。

たとえば、すこし上で定義した、ひとつの `Int` 型コンテキストパラメータをもつ `double` 関数について考えよう。今しがた定義した `theMagicNumber` という given インスタンスも `Int` という型をもっている。この場合、コンテキストパラメータを与えずに `double` を呼び出すと、コンパイラは `theMagicNumber` をパラメータとして使用する。

```scala mdoc
double
```

using 句に同じ型のパラメータが複数ある場合でも、同じ given インスタンスが使用される。上述の `add` 関数がちょうどそのような定義をもっている。

```scala mdoc
add
```

以上が using 句と given インスタンスに関するもっとも重要なポイントである。次に、これらがどういう意味をもっているのかについて詳しく見ていこう。

### given スコープとインポート

given インスタンスは通常、暗黙的に using 句へと渡される。これらの構文の存在意義は、この暗黙的な処理をコンパイラに指示することにある。これはコードをわかりづらいものにする可能性があるため、どの given インスタンスが using 句に供給される候補となるのかは、明確にしておく必要がある。この節では、コンパイラが given インスタンスを探す場所である **given スコープ**と、given インスタンスをインポートするための特別な構文について見ていく。

given スコープについて知っておくべき最初のルールは、スコープの基準地点となるのが using 句をもつメソッドの**定義場所**ではなく、そのメソッドの**呼び出し場所**であるということである。次のコードでは、given インスタンス `a` は、メソッドの定義場所と同じスコープ内にはあるものの、呼び出し場所と同じスコープには存在しない。したがってこれはコンパイルできない。

```scala mdoc:reset:fail
object A {
  given a: Int = 1
  def whichInt(using int: Int): Int = int
}

A.whichInt
```

ふたつ目のルールは、これまでの例でも常に使用してきたものであるが、given スコープには呼び出し場所の**レキシカルスコープ**が含まれる、ということである。レキシカルスコープは通常、名前に関連付けられた値（メソッドパラメータ名や `val` 宣言など）を見つけるために使用される。次のコードは、`a` が呼び出し場所と同じスコープ内に定義されているので、正常にコンパイルされる。

```scala mdoc:silent
object A {
  given a: Int = 1
  
  object B {
    C.whichInt 
  }
  
  object C {
    def whichInt(using int: Int): Int = int
  }
}
```

一方、同じ型をもつ複数の given インスタンスが同じスコープ内に存在する場合、コンパイラはどれかひとつを適当に選択したりはしない。代わりに、その選択が曖昧であることを知らせるエラーが発生する。

```scala
object A {
  given a: Int = 1
  given b: Int = 2
    
  def whichInt(using int: Int): Int = int
    
  whichInt
}
// error:
// Ambiguous given instances: both given instance a in object A and
// given instance b in object A match type Int of parameter int of 
// method whichInt in object A
```

given インスタンスを他のスコープからインポートすることもできる。通常のインポートと似ているが、given インスタンスをインポートする場合には、それを明示しなくてはならない。以下のコードは given インスタンスを明示的にインポートしていないのでコンパイルできない。

```scala mdoc:reset:fail
object A {
  given a: Int = 1

  def whichInt(using int: Int): Int = int
}
object B {
  import A.*
    
  whichInt
}
```

`import A.given` という表現を使って明示的にインポートすれば、このコードはコンパイルできる。

```scala mdoc:reset:silent
object A {
  given a: Int = 1

  def whichInt(using int: Int): Int = int
}
object B {
  import A.{given, *}
    
  whichInt
}
```

最後のルールとして、given スコープには、using 句の型に関わるすべての型のコンパニオンオブジェクトが含まれる。例を見るのが一番わかりやすいだろう。まず、型変数 `A` が発する音を表す型 `Sound` を定義し、その音にアクセスするためのメソッド `soundOf` を作成する。

```scala mdoc:reset
trait Sound[A] {
  def sound: String
}

def soundOf[A](using s: Sound[A]): String =
  s.sound
```

次に given インスタンスをいくつか定義する。それらのインスタンスが、それぞれに対応するコンパニオンオブジェクト上で定義されていることに注目してほしい。

```scala mdoc:silent
trait Cat
object Cat {
  given catSound: Sound[Cat] =
    new Sound[Cat]{
      def sound: String = "meow"
    }
}

trait Dog
object Dog {
  given dogSound: Sound[Dog] = 
    new Sound[Dog]{
      def sound: String = "woof"
    }
}
```

`soundOf` の呼び出しにあたって、インスタンスを明示的にスコープにもちこむ必要はない。それらは、使用する型（`Cat` や `Dog`）のコンパニオンオブジェクト上に定義されていることにより、自動的に given スコープに含まれる。これらのインスタンスが `Sound` のコンパニオンオブジェクトに定義されていた場合もやはり、given スコープに含まれる。`Sound[A]` を探す際には、`Sound` と `A` 両方のコンパニオンオブジェクトがスコープに含まれるためである。

```scala mdoc
soundOf[Cat]
soundOf[Dog]
```

ほとんどの場合、given インスタンスはコンパニオンオブジェクト上に定義されるべきである。このシンプルな整理方法により、メソッドの利用者は given インスタンスを明示的にインポートする必要がなくなるし、実装を確認したいときには簡単に見つけることが可能となる。

#### given インスタンスの優先順位

given インスタンスの選択はもっぱら型に基づいて行われる。`soundOf` には、値を何ひとつ渡していない。つまり、各型に対してインスタンスがそれぞれひとつしかない場合、given インスタンスは特に使いやすくなる。この場合、インスタンスを関連するコンパニオンオブジェクト上に置くだけで、すべてがうまく機能する。

しかし、これは常に可能とは限らない（もっとも、可能でないのは設計がよくないことを示唆しているケースが多い）。ある型に対して複数のインスタンスが必要な場合、その中からひとつを選択するためにインスタンスの優先順位ルールが使用される。以下では三つの重要なルールを見ていく。

第一のルールとして、明示的に渡されたインスタンスは他のどれよりも優先される。

```scala mdoc:reset:silent
given a: Int = 1
def whichInt(using int: Int): Int = int
```
```scala mdoc
whichInt(using a)
```

第二に、レキシカルスコープにあるインスタンスがコンパニオンオブジェクトに置かれたインスタンスよりも優先される。

```scala mdoc:reset:silent
trait Sound[A] {
  def sound: String
}
trait Cat
object Cat {
  given catSound: Sound[Cat] =
    new Sound[Cat]{
      def sound: String = "meow"
    }
}

def soundOf[A](using s: Sound[A]): String =
  s.sound
```
```scala mdoc
given purr: Sound[Cat]  =
  new Sound[Cat]{
    def sound: String = "purr"
  }

soundOf[Cat]
```

最後に、より近くのレキシカルスコープ内にあるインスタンスが、遠くのものよりも優先される。

```scala mdoc
{
  given growl: Sound[Cat] =
   new Sound[Cat]{
     def sound: String = "growl"
   }
   
  {
    given mew: Sound[Cat] =
     new Sound[Cat]{
       def sound: String = "mew"
     }
     
    soundOf[Cat]
  }
}
```

これで、given インスタンスと using 句の仕組みについての詳細をほぼ網羅した。だが、これらは技法レベルの説明にすぎない。そこから「どういう時にこれらの機能を使うのか？」という疑問が生じるのは当然のことだろう。次は、この疑問に応えるため、型クラスおよび Scala におけるその実装について見ていく。
