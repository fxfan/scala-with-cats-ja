## 余データの構造的再帰と余再帰

In this section we'll build a library for streams, also known as lazy lists. These are the codata equivalent of lists. Whereas a list must have a finite length, streams have an infinite length. We'll use this example to explore structural recursion and structural corecursion as applied to codata.

この節ではストリームあるいは遅延リストとして知られるライブラリを作成する。これらはリストの余データに相当する。リストの長さが有限でなくてはならないのに対して、ストリームは無限の長さをもっている。この例を用いて、余データに適用される構造的再帰と構造的余再帰について考察する。

Let's start by reviewing structural recursion and corecursion. The key idea is to use the input or output type, respectively, to drive the process of writing the method. We've already seen how this works with data, where we emphasized structural recursion. With codata it's more often the case that structural corecursion is used. The steps for using structural corecursion are:

構造的再帰と構造的余再帰について復習することから始めよう。鍵となるアイデアは、それぞれ入力型または出力型を利用して、メソッドの記述プロセスを

このセクションでは、構造的再帰と構造的余再帰を復習する。鍵となるアイデアは、それぞれ入力型または出力型を利用してメソッドを考えながら書き進めることである。データにおいて構造的再帰にフォーカスした手法がどのように機能するかは既に見てきた。余データにおいては、構造的余再帰がより頻繁に使用される。構造的余再帰を使用するためのステップは以下の通りである。

1. recognize the output of the method or function is codata;
2. write down the skeleton to construct an instance of the codata type, usually using an anonymous subclass; and
3. fill in the methods, using strategies such as structural recursion or following the types to help.

1. メソッドまたは関数の出力が余データであることを認識する
2. 余データ型のインスタンスを構築する骨組みを記述する。通常は匿名サブクラスを使用する
3. 構造的再帰などの戦略を用いるか、型に従って、メソッドの本体を完成させる

It's important that any computation takes places within the methods, and so only runs when the methods are called. Once we start creating streams the importance of this will become clear.

どの計算もメソッド内で行われ、メソッドが呼び出されたときにのみ実行されることが重要である。ストリームを作成し始めると、この重要性が明確になるだろう。

For structural recursion the steps are:

構造的再帰の手順は以下の通りである。

1. recognize the input of the method or function is codata;
2. note the codata's destructors as possible sources of values in writing the method; and
3. complete the method, using strategies such as following the types or structural corecursion and the methods identified above.

1. メソッドまたは関数の入力が余データであることを認識する
2. メソッドを書く際に、余データのデストラクタが値の供給源であることに留意する
3. 型に従ったり、構造的余再帰などの戦略を用いたりして、上記で特定したメソッドを使いながらメソッドを完成させる

<!-- TODO: 上記で特定したメソッドとはデストラクタのこと？ -->

Our first step is to define our stream type. As this is codata, it is defined in terms of its destructors. The destructors that define a `Stream` of elements of type `A` are:

最初のステップはストリーム型を定義することである。ストリームは余データなので、デストラクタの集まりとして定義される。`A` 型の要素をもつ `Stream` を定義するデストラクタは以下のとおりである。

- a `head` of type `A`; and
- a `tail` of type `Stream[A]`.

- `A` 型の `head`
- `Stream[A]` 型の `tail`

Note these are almost the destructors of `List`. We haven't defined `isEmpty` as a destructor because our streams never end and thus this method would always return `false`. (A lot of real implementations, such as the `LazyList` in the Scala standard library, do define such a method which allows them to represent finite and infinite lists in the same structure. We're not doing this for simplicity and because we want to work with codata in its purest form.)

これらが `List` のデストラクタとほぼ同じである点に注目しよう。デストラクタとして `isEmpty` を定義していないのは、ここで作成しようとしているストリームに終わりがなく、`isEmpty` があったとしてもそれは常に `false` を返すことになるからである。なお、Scala標準ライブラリの `LasyList` など多くの実例において `isEmpty` のようなメソッドが定義されているが、これは、それらが有限と無限両方のリストを表現可能であるという理由による。ここでは、余データを最も端的な形で定義したいので、シンプルさを優先し、そのようなことはしない。

We can translate this to Scala, as we've previously seen, giving us

これをScalaに翻訳すると、これまでに見てきたように、次のようになる。

```scala mdoc:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]
}
```

Now we can create an instance of `Stream`. Let's create a never-ending stream of ones.
We will start with the skeleton below and apply strategies to complete the code.

これで、`Stream` インスタンスを作成できる。ここでは、永遠に1という数を返し続けるストリームを作成してみよう。以下の骨組みから始め、コードを完成させるための戦略を適用する。

```scala
val ones: Stream[Int] = ???
```

The first strategy is structural corecursion. We're returning an instance of codata, so we can insert the skeleton to construct a `Stream`.

最初の戦略は構造的余再帰である。余データのインスタンスを返したいのだから、 `Stream` インスタンスを構築する骨組みを挿入する。

```scala
val ones: Stream[Int] =
  new Stream[Int] {
    def head: Int = ???
    def tail: Stream[Int] = ???
  }
```

Here I've used the anonymous subclass approach, so I can just write all the code in one place.

ここでは、匿名サブクラスを使うことで、すべてのコードを一か所にまとめて書けるようにしている。

The next step is to fill in the method bodies. The first method, `head`, is trivial. The answer is `1` by definition.

次のステップはメソッド本体を埋めることである。最初のメソッドである `head` は簡単で、定義により答えは `1` である。

```scala
val ones: Stream[Int] =
  new Stream[Int] {
    def head: Int = 1
    def tail: Stream[Int] = ???
  }
```

It's not so obvious what to do with `tail`. We want to return a `Stream[Int]` so we could apply structural corecursion again.

`tail` をどのように実装すればよいかは自明とは言えない。`Stream[Int]` 型の値を返したいのだから、もう一度構造的余再帰を適用することが考えられる。

```scala
val ones: Stream[Int] =
  new Stream[Int] {
    def head: Int = 1
    def tail: Stream[Int] =
      new Stream[Int] {
        def head: Int = 1
        def tail: Stream[Int] = ???
      }
  }
```

This approach doesn't seem like it's going to work. We'll have to write this out an infinite number of times to correctly implement the method, which might be a problem.

だが、このアプローチがうまく行くとは思えない。この方法でメソッドを正しく実装するには同じことを無限に書き続けなくてはならない。

Instead we can follow the types. We need to return a `Stream[Int]`. We have one in scope: `ones`. This is exactly the `Stream` we need to return: the infinite stream of ones!

代わりに、型に従って考えてみよう。`Stream[Int]` 型を返す必要があるが、スコープ内にはその型の変数である `ones` がすでに存在する。これこそがまさに、返すべき無限の1のストリームである。

```scala mdoc:silent
val ones: Stream[Int] =
  new Stream[Int] {
    def head: Int = 1
    def tail: Stream[Int] = ones
  }
```

You might be alarmed to see the circular reference to `ones` in `tail`. This works because it is within a method, and so is only evaluated when that method is called. This delaying of evaluation is what allows us to represent an infinite number of elements, as we only ever evaluate a finite portion of them. This is a core difference from data, which is fully evaluated when it is constructed.

`tail` で `ones` への循環参照を見て驚くかもしれないが、これは問題ない。この参照はメソッド内にあり、メソッドが呼び出されたときにのみ評価される。この評価の遅延こそが、無限の要素を表現できる理由であり、実際には有限の部分だけが評価される。これは、データのように構築時にすべてが評価されるものとは根本的に異なる点である。

Let's check that our definition of `ones` does indeed work.
We can't extract all the elements from an infinite `Stream` (at least, not in finite time) so in general we'll have to resort to checking a finite sequence of elements. 

この `ones` の定義が確かに動作することを確認しておこう。無限のストリームからすべての要素を取り出すことは（少なくとも有限の時間内には）できない。したがって、通常は有限の要素列を確認することに頼らざるを得ない。

```scala mdoc
ones.head
ones.tail.head
ones.tail.tail.head
```

This all looks correct. We'll often want to check our implementation in this way, so let's implement a method, `take`, to make this easier.

すべて正しいようである。動作確認にはこの方法を使うことが多いので、これをもっと簡単にしてくれるメソッド `take` を実装しよう。

```scala mdoc:reset:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]
  
  def take(count: Int): List[A] =
    count match {
      case 0 => Nil
      case n => head :: tail.take(n - 1)
    }
}
```

We can use either the structural recursion or structural corecursion strategies for data to implement `take`. Since we've already covered these in detail I won't go through them here. The important point is that `take` only uses the destructors when interacting with the `Stream`. 

`take` の実装には、データに対する構造的再帰か構造的余再帰いずれかの戦略を利用できる。それらについては既に詳しく説明しているので、ここでは触れない。重要なのは、`take` がデストラクタのみを通じて `Stream` とやり取りしているということである。

Now we can more easily check our implementations are correct.

これで、実装が正しいことをより簡単に確認できるようになる。

```scala mdoc:invisible
val ones: Stream[Int] =
  new Stream {
    def head: Int = 1

    def tail: Stream[Int] = ones
  }
```
```scala mdoc
ones.take(5)
```

For our next task we'll implement `map`. Implementing a method on `Stream` allows us to see both structural recursion and corecursion for codata in action. As usual we begin by writing out the method skeleton.

次の課題として `map` を実装する。`Stream` にメソッドを実装することで、余データにおける構造的再帰と構造的余再帰の両方を実際に確認することができる。いつものようにメソッドの骨組みを書き出すところから始める。

```scala mdoc:reset:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]
  
  def map[B](f: A => B): Stream[B] = 
    ???
}
```

Now we have a choice of strategy to use. Since we haven't used structural recursion yet, let's start with that. The input is codata, a `Stream`, and the structural recursion strategy tells us we should consider using the destructors. Let's write them down to remind us of them.

利用する戦略を決めなくてはならない。まだ構造的再帰を使っていないので、まずはそれを試してみよう。入力は余データ `Stream` である。構造的再帰戦略ではデストラクタを用いることを検討すべきとされているので、リマインドの意味でデストラクタ呼び出しを書き出しておこう。

```scala
trait Stream[A] {
  def head: A
  def tail: Stream[A]
  
  def map[B](f: A => B): Stream[B] = {
    this.head ???
    this.tail ???
  }
}
```

To make progress we can follow the types or use structural corecursion. Let's choose corecursion to see another example of it in use.

さらに実装を進めるために、型に従う手法か構造的余再帰を利用できる。構造的余再帰の使用例をもうひとつ見てみたいので、そちらを選択することにしよう。

```scala
trait Stream[A] {
  def head: A
  def tail: Stream[A]
  
  def map[B](f: A => B): Stream[B] = {
    this.head ???
    this.tail ???
    
    new Stream[B] {
      def head: B = ???
      def tail: Stream[B] = ???
    }
  }
}
```

Now we've used structural recursion and structural corecursion, a bit of following the types is in order. This quickly arrives at the correct solution.

これで構造的再帰と構造的余再帰を使ったので、次は型に従う方法をすこし試してみよう。迅速に正しい解答にたどり着くことができる。

```scala mdoc:reset:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]
  
  def map[B](f: A => B): Stream[B] = {
    val self = this 
    new Stream[B] {
      def head: B = f(self.head)
      def tail: Stream[B] = self.tail.map(f)
    }
  }
}
```

There are two important points. Firstly, notice how I gave the name `self` to `this`. This is so I can access the value inside the new `Stream` we are creating, where `this` would be bound to this new `Stream`. Next, notice that we access `self.head` and `self.tail` inside the methods on the new `Stream`. This maintains the correct semantics of only performing computation when it has been asked for. If we performed the computation outside of the methods that we would do it too early, which is some cases can lead to an infinite loop.

重要な点が二つある。まず注目してほしいのは、`this` に `self` という名前を与えたことである。作成しようとしている新しい `Stream` オブジェクトの中では `this` はその新しいストリーム自体を指すので、その中で元の `Stream` オブジェクトを参照するにはこれが必要となる。次に、`self.head` と `self.tail` へのアクセスが、新しい `Stream` に定義されたメソッドの中で行われている点に注目してほしい。これによって、必要とされたときにのみ計算を行うという適切なセマンティクスが維持される。メソッド外で計算を行うと、必要以上に早く評価されてしまい、場合によっては無限ループにつながることもある。

As our final example, let's return to constructing `Stream`, and implement the universal constructor `unfold`. We start with the skeleton for `unfold`, remembering the `seed` parameter.

最後の例として、再び `Stream` の構築に戻り、汎用的なコンストラクタである `unfold` を実装しよう。`seed` パラメータについて考慮しつつ、まずは `unfold` の骨組みから始める。


```scala mdoc:reset:silent 
trait Stream[A] {
  def head: A
  def tail: Stream[A]
}
object Stream {
  def unfold[A, B](seed: A): Stream[B] =
    ???
}
```

It's natural to apply structural corecursion to make progress.

構造的余再帰を適用して実装を進めるのが自然だろう。

```scala mdoc:reset:silent 
trait Stream[A] {
  def head: A
  def tail: Stream[A]
}
object Stream {
  def unfold[A, B](seed: A): Stream[B] =
    new Stream[B]{
      def head: B = ???
      def tail: Stream[B] = ???
    }
}
```

Now we can follow the types, adding parameters as we need them. This gives us the complete method shown below.

次に、型に従って、必要に応じたパラメータを追加する。これにより、メソッドは以下のとおり完成する。

```scala mdoc:reset:silent 
trait Stream[A] {
  def head: A
  def tail: Stream[A]
}
object Stream {
  def unfold[A, B](seed: A, f: A => B, next: A => A): Stream[B] =
    new Stream[B]{
      def head: B = 
        f(seed)
      def tail: Stream[B] = 
        unfold(next(seed), f, next)
    }
}
```

We can use this to implement some interesting streams. Here's a stream that alternates between `1` and `-1`.

これを使って、おもしろいストリームを実装できる。以下に示すのは `1` と `-1` が交互に現れるストリームである。

```scala mdoc:reset:invisible
trait Stream[A] {
  def head: A
  def tail: Stream[A]

  def take(count: Int): List[A] =
    count match {
      case 0 => Nil
      case n => head :: tail.take(n - 1)
    }
}
object Stream {
  def unfold[A, B](seed: A, f: A => B, next: A => A): Stream[B] =
    new Stream[B]{
      def head: B = 
        f(seed)
      def tail: Stream[B] = 
        unfold(next(seed), f, next)
    }
}
```

```scala mdoc:silent
val alternating = Stream.unfold(
  true, 
  x => if x then 1 else -1, 
  x => !x
)
```

We can check it works.

動作を確認しておこう。

```scala mdoc
alternating.take(5)
```


#### 演習: `Stream` のコンビネータ実装 {-}

It's time for you to get some practice with structural recursion and structural corecursion using codata.
Implement `filter`, `zip`, and `scanLeft` on `Stream`. They have the same semantics as the same methods on `List`, and the signatures shown below.

そろそろ、余データを用いた構造的再帰と構造的余再帰について練習をしておこう。`Stream` に `filter`、`zip`、および `scanLeft` メソッドを実装せよ。いずれも `List` における同名のメソッドと同じ意味をもっているものとし、シグネチャは以下のとおりとする。

```scala mdoc:reset:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]

  def filter(pred: A => Boolean): Stream[A]
  def zip[B](that: Stream[B]): Stream[(A, B)]
  def scanLeft[B](zero: B)(f: (B, A) => B): Stream[B]
}
```

<div class="solution">
For all of these methods I found that structural corecursion was the most natural way to tackle them. You could start with structural recursion, though.

いずれのメソッドの実装においても、構造的余再帰がもっとも自然なアプローチだろう。構造的再帰から始めることも可能ではあるが。

You might be worried about the inefficiency of `filter`. That's something we'll discuss a bit later.

`filter` の非効率さが気になるかもしれないが、それについては後ほどすこし説明する。

```scala mdoc:reset:silent
trait Stream[A] {
  def head: A
  def tail: Stream[A]

  def filter(pred: A => Boolean): Stream[A] = {
    val self = this
    new Stream[A] {
      def head: A = {
        def loop(stream: Stream[A]): A =
          if pred(stream.head) then stream.head
          else loop(stream.tail)
          
        loop(self)
      }
      
      def tail: Stream[A] = {
        def loop(stream: Stream[A]): Stream[A] =
          if pred(stream.head) then stream.tail
          else loop(stream.tail)
          
        loop(self)
      }
    }
  }

  def zip[B](that: Stream[B]): Stream[(A, B)] = {
    val self = this 
    new Stream[(A, B)] {
      def head: (A, B) = (self.head, that.head)
      
      def tail: Stream[(A, B)] =
        self.tail.zip(that.tail)
    }
  }

  def scanLeft[B](zero: B)(f: (B, A) => B): Stream[B] = {
    val self = this
    new Stream[B] {
      def head: B = f(zero, self.head)
      
      def tail: Stream[B] =
        self.tail.scanLeft(this.head)(f)
    }
  }
}
```
</div>

We can do some neat things with the methods defined above. For example, here is the stream of natural numbers.

上記で定義したメソッドを使うと、クールなことができる。たとえば、自然数のストリームは次のようになる。

```scala mdoc:reset:invisible
trait Stream[A] {
  def head: A
  def tail: Stream[A]

  def filter(pred: A => Boolean): Stream[A] = {
    val self = this
    new Stream[A] {
      def head: A = {
        def loop(stream: Stream[A]): A =
          if pred(stream.head) then stream.head
          else loop(stream.tail)
          
        loop(self)
      }
      
      def tail: Stream[A] = {
        def loop(stream: Stream[A]): Stream[A] =
          if pred(stream.head) then stream.tail
          else loop(stream.tail)
          
        loop(self)
      }
    }
  }

  def scanLeft[B](zero: B)(f: (B, A) => B): Stream[B] = {
    val self = this
    new Stream[B] {
      def head: B = f(zero, self.head)
      
      def tail: Stream[B] =
        self.tail.scanLeft(this.head)(f)
    }
  }

  def take(count: Int): List[A] =
    count match {
      case 0 => Nil
      case n => head :: tail.take(n - 1)
    }

  def zip[B](that: Stream[B]): Stream[(A, B)] = {
    val self = this 
    new Stream[(A, B)] {
      def head: (A, B) = (self.head, that.head)
      
      def tail: Stream[(A, B)] =
        self.tail.zip(that.tail)
    }
  }

}
object Stream {
  def unfold[A, B](seed: A, f: A => B, next: A => A): Stream[B] =
    new Stream[B]{
      def head: B = 
        f(seed)
      def tail: Stream[B] = 
        unfold(next(seed), f, next)
    }
    
  val ones = unfold(1, identity, identity)
}
```

```scala mdoc:silent
val naturals = Stream.ones.scanLeft(0)((b, a) => b + a)
```

As usual, we should check it works.

いつもどおり、動作確認をしておこう。

```scala mdoc
naturals.take(5)
```

We could also define `naturals` using `unfold`. 
More interesting is defining it in terms of itself.

`unfold` を用いて `naturals` を定義することもできる。もっと興味深いのは、`naturals` を使って `naturals` を定義できることである。

```scala mdoc:reset:invisible
trait Stream[A] {
  def head: A
  def tail: Stream[A]

  def +:(elt: => A): Stream[A] = {
    val self = this
    new Stream[A] {
      def head: A = elt
      def tail: Stream[A] = self
    }
  }

  def filter(pred: A => Boolean): Stream[A] = {
    val self = this
    new Stream[A] {
      def head: A = {
        def loop(stream: Stream[A]): A =
          if pred(stream.head) then stream.head
          else loop(stream.tail)
          
        loop(self)
      }
      
      def tail: Stream[A] = {
        def loop(stream: Stream[A]): Stream[A] =
          if pred(stream.head) then stream.tail
          else loop(stream.tail)
          
        loop(self)
      }
    }
  }

  def map[B](f: A => B): Stream[B] = {
    val self = this
    new Stream[B] {
      def head: B = f(self.head)
      def tail: Stream[B] = self.tail.map(f)
    }
  }

  def scanLeft[B](zero: B)(f: (B, A) => B): Stream[B] = {
    val self = this
    new Stream[B] {
      def head: B = f(zero, self.head)
      
      def tail: Stream[B] =
        self.tail.scanLeft(this.head)(f)
    }
  }

  def take(count: Int): List[A] =
    count match {
      case 0 => Nil
      case n => head :: tail.take(n - 1)
    }

  def zip[B](that: Stream[B]): Stream[(A, B)] = {
    val self = this 
    new Stream[(A, B)] {
      def head: (A, B) = (self.head, that.head)
      
      def tail: Stream[(A, B)] =
        self.tail.zip(that.tail)
    }
  }

}
object Stream {
  def unfold[A, B](seed: A, f: A => B, next: A => A): Stream[B] =
    new Stream[B]{
      def head: B = 
        f(seed)
      def tail: Stream[B] = 
        unfold(next(seed), f, next)
    }
    
  val ones = unfold(1, identity, identity)
}
```
```scala mdoc:silent
val naturals: Stream[Int] =
  new Stream {
    def head = 1
    def tail = naturals.map(_ + 1)
  }
```

This might be confusing. If so, spend a bit of time thinking about it. It really does work!

これはややこしいかもしれないが、もしそう思ったらすこし時間をとって考えてみてほしい。この実装はちゃんと動作するのである。

```scala mdoc
naturals.take(5)
```


### Efficiency and Effects 効率と副作用

You may have noticed that our implement recomputes values, possibly many times. 
A good example is the implementation of `filter`.
This recalculates the `head` and `tail` on each call, which could be a very expensive operation.

上記の実装において、値を場合によっては何度も繰り返し再計算していることに気付いたかもしれない。`filter` の実装がそのよい例である。`filter` は呼び出すたびに `head` と `tail` を再計算するため、非常にコストの高い操作になる可能性がある。

```scala 
def filter(pred: A => Boolean): Stream[A] = {
  val self = this
  new Stream[A] {
    def head: A = {
      def loop(stream: Stream[A]): A =
        if pred(stream.head) then stream.head
        else loop(stream.tail)
        
      loop(self)
    }
    
    def tail: Stream[A] = {
      def loop(stream: Stream[A]): Stream[A] =
        if pred(stream.head) then stream.tail
        else loop(stream.tail)
        
      loop(self)
    }
  }
}
```

We know that delaying the computation until the method is called is important, because that is how we can handle infinite and self-referential data. However we don't need to redo this computation on succesive calls. We can instead cache the result from the first call and use that next time.
Scala makes this easy with `lazy val`, which is a `val` that is not computed until its first call.
Additionally, Scala's use of the *uniform access principle* means we can implement a method with no parameters using a `lazy val`.
Here's a quick example demonstrating it in use.

メソッドが呼び出されるまで計算を遅延することの重要性はすでに述べた。それによって無限の、そして自己参照的なデータを扱うことができるからである。だが、そのメソッドが連続して呼び出されるときに、同じ計算を再実行する必要はない。最初の呼び出し時に結果をキャッシュしておけば、次回からはそれを使用できる。Scalaでは `lazy val` を使うことでこれを簡単に実現できる。`lazy val` は、最初に呼び出されるまで評価されない `val` である。


```scala mdoc:silent
def always[A](elt: => A): Stream[A] =
  new Stream[A] {
    lazy val head: A = elt
    lazy val tail: Stream[A] = always(head)
  }
  
val twos = always(2)
```

As usual we should check our work.

いつもどおり動作確認をしておこう。

```scala mdoc
twos.take(5)
```

We get the same result whether we use a method or a `lazy val`, because we are assuming that we are only dealing with pure computations that have no dependency on state that might change. In this case a `lazy val` simply consumes additional space to save on time.

メソッドと `lazy val` どちらを使っても得られる結果は同じになる。可変状態に依存しない純粋な計算のみを扱っている想定だからである。そのような場合においては、`lazy val` はメモリを追加で消費し、時間を節約する役割を果たす。

<!-- TODO: ここより下、評価戦略に関する原文の記述はちょっと混乱していると思う。訳注するか？ -->

Recomputing a result every time it is needed is known as **call by name**, while caching the result the first time it is computed is known as **call by need**. These two different **evaluation strategies** can be applied to individual values, as we've done here, or across an entire programming. Haskell, for example, uses call by need. All values in Haskell are only computed the first time they are need. This is approach is sometimes known as **lazy evaluation**. Another alternative, called **call by value**, computes results when they are defined instead of waiting until they are needed. This is the default in Scala.

必要になるたびに結果を再計算する方式は**名前渡し（call by name）**と呼ばれ、最初に計算された結果をキャッシュしておく方式は**必要渡し（call by need）**と呼ばれる。この二つの異なる**評価戦略（evaluation strategy）**は、今回のように個々の値に適用することもできるし、プログラム全体にわたって適用することもできる。例として、Haskellでは必要渡しが使われており、すべての値はそれが必要となる最初のタイミングでのみ計算される。このアプローチは**遅延評価**と呼ばれることもある。**値渡し（call by value）**と呼ばれる別の方式では、結果が必要とされるのを待たず、定義された時点で計算される。この方式がScalaでのデフォルトである。

We can illustrate the difference between call by name and call by need if we use an impure computation. 
For example, we can define a stream of random numbers.
Random number generators depend on some internal state.

純粋でない計算を使って、名前渡しと必要渡しの違いを示すことができる。たとえば、乱数のストリームを定義してみよう。乱数生成器は内部状態に依存している。

Here's the call by name implementation, using the methods we have already defined.

以下は、名前渡しを用いた実装である。

```scala mdoc:silent
import scala.util.Random

val randoms: Stream[Double] = 
  Stream.unfold(Random, r => r.nextDouble(), r => r)
```

Notice that we get *different* results each time we `take` a section of the `Stream`.
We would expect these results to be the same.

`Stream` の一部を `take` するたびに、*異なる*結果が得られることに注目してほしい。これらの結果は本来同じであることが期待される。

```scala mdoc
randoms.take(5)
randoms.take(5)
```

Now let's define the same stream in a call by need style, using `lazy val`.

次に、`lazy val` を使い、必要渡しのスタイルで同じストリームを定義してみよう。

```scala mdoc:silent
val randomsByNeed: Stream[Double] =
  new Stream[Double] {
    lazy val head: Double = Random.nextDouble()
    lazy val tail: Stream[Double] = randomsByNeed
  }
```

This time we get the *same* result when we `take` a section, and each number is the same.

今回は、ストリームの一部を `take` したときに得られる結果は同じになる。また、得られる数はどれも同じである。

```scala mdoc
randomsByNeed.take(5)
randomsByNeed.take(5)
```

If we wanted a stream that had a different random number for each element but those numbers were constant, we could redefine `unfold` to use call by need.

```scala mdoc:silent
def unfoldByNeed[A, B](seed: A, f: A => B, next: A => A): Stream[B] =
  new Stream[B]{
    lazy val head: B = 
      f(seed)
    lazy val tail: Stream[B] = 
      unfoldByNeed(next(seed), f, next)
  }
```

Now redefining `randomsByNeed` using `unfoldByNeed` gives us the result we are after. First, redefine it.

```scala mdoc:silent
val randomsByNeed2 =
  unfoldByNeed(Random, r => r.nextDouble(), r => r)
```

Then check it works.

```scala mdoc
randomsByNeed2.take(5)
randomsByNeed2.take(5)
```

These subtleties are one of the reasons that functional programmers try to avoid using state as far as possible.
