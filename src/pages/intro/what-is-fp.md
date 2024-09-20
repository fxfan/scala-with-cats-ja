## 関数型プログラミング

This is a book about the techniques and practices of functional programming (FP). This naturally leads to the question: what is FP and what does it mean to write code in a functional style? It's common to view functional programming as a collection of language features, such as first class functions, or to define it as a programming style using immutable data and pure functions. (Pure functions always return the same output given the same input.) This was my view when I started down the FP route, but I now believe the true goals of FP are enabling local reasoning and composition. Language features and programming style are in service of these goals. Let me attempt to explain the meaning and value of local reasoning and composition.

この本は、関数型プログラミングの技法と実践について書かれている。ここで、関数型プログラミングとは何か、そして関数型の流儀でコードを書くとはどういうことか、という疑問が当然に生じる。関数型プログラミングを、ファーストクラス関数のような言語機能の集合として捉えたり、不変データと純粋関数を使うプログラミングスタイルとして定義したりするのは一般的で（純粋関数とは、同じ入力に対して常に同じ出力を返す関数のこと）、私も当初はそのように考えていた。だが、今では、関数型プログラミングの核心は局所推論（local reasoning）と合成（composition）であると考えている。言語機能やプログラミングスタイルは、これらの目標を達成するための手段に過ぎない。次に、局所推論と合成の意味と価値について説明しよう。

### 関数型プログラミングとは何か

I believe that functional programming is a hypothesis about software quality: that it is easier to write and maintain software that can be understood before it is run, and is built of small reusable components. The first property is known as local reasoning, and the second as composition. Let's address each in turn.

関数型プログラミングはソフトウェア品質に関する仮説であると私は考えている。実行する前に理解できるソフトウェア、そして小さく再利用可能なコンポーネントで構築されたソフトウェアの方が、書くことも保守することも容易だという考え方である。前者の特性は「局所推論」として知られており、後者は「合成」と呼ばれている。

Local reasoning means we can understand pieces of code in isolation. When we see the expression `1 + 1` we know what it means regardless of the weather, the database, or the current status of our Kubernetes cluster. None of these external events can change it. This is a trivial and slightly silly example, but it illustrates the point. A goal of functional programming is to extend this ability across our code base. 

局所推論とは、コードの一部を他から切り離して理解できるということを意味する。たとえば、`1 + 1` という式は、天気やデータベースの状態、現在のKubernetesクラスタの状況に関係なく、その意味を理解可能である。外部要因が式の意味を変えることはない。この例は単純で少し滑稽だが、要点をよく表している。関数型プログラミングの目標のひとつは、この特性をコード全体に拡げることである。

It can help to understand local reasoning by looking at what it is not. Shared mutable state is out because relying on shared state means that other code can change what our code does without our knowledge. It means no global mutable configuration, as found in many web frameworks and graphics libraries for example, as any random code can change that configuration. Metaprogramming has to be carefully controlled. No [monkey patching][monkey-patching], for example, as again it allows other code to change our code in non-obvious ways. As we can see, adapting code to enable local reasoning can mean quite some sweeping changes. However if we work in a language that embraces functional programming this style of programming is the default.

局所推論について理解しようとするなら、それが何でないかを考えることが役立つかもしれない。可変状態を共有していると局所推論はできなくなる。なぜなら、共有された状態に依存すると、そのコードの動作を他のコードが知らないうちに変えてしまう可能性があるからである。敷衍すると、多くのウェブフレームワークやグラフィックスライブラリに見られるような、グローバルな可変の設定オブジェクトも使うことができない。使えば、任意のコードがその設定を変更できてしまう。また、メタプログラミングも慎重に管理する必要がある。[モンキーパッチ][monkey-patching]も他のコードが明示的でない方法でコードを変更できてしまうので、避けるべきである。このように、局所推論が可能となるようコードを適応させるには、かなり大きな変更が必要になることがある。しかし、関数型プログラミングを取り入れた言語で作業するのであれば、このプログラミングスタイルが基本となる。

Composition means building big things out of smaller things. Numbers are compositional. We can take any number and add one, giving us a new number. Lego is also compositional. We compose Lego by sticking it together. In the particular sense we're using composition we also require the original elements we combine don't change in any way when they are composed. When we create by `2` by adding `1` and `1` we get a new result that doesn't change what `1` means.

合成とは、小さなものを組み合わせて大きなものを作ることを指す。たとえば、数は合成可能である。どんな数でも、1を足せば新しい数が得られる。レゴもまた合成可能である。ブロック同士をくっつけることで組み立てることができる。ここで我々が指している合成という概念では、組み合わせる元となる要素が、合成をしたあとも変化しないことが求められる。たとえば、`1` と `1` を足して `2` を作るとき、その結果として新しい数が得られるが、この操作は1そのものの意味を変えるわけではない。

We can find compositional ways to model common programming tasks once we start looking for them. React components are one example familiar to many front-end developers: a component can consist of many components. HTTP routes can be modelled in a compositional way. A route is a function from an HTTP request to either a handler function or a value indicating the route did not match. We can combine routes as a logical or: try this route or, if it doesn't match, try this other route. Processing pipelines are another example that often use sequential composition: perform this pipeline stage and then this other pipeline stage.

よくあるプログラミングのタスクをモデル化する合成的手法は、探してみると結構見つかる。多くのフロントエンド開発者に馴染みのある例としては、Reactコンポーネントがある。Reactコンポーネントは、複数のコンポーネントを組み合わせて作成することができる。また、HTTPルートも合成的手法でモデル化される。ひとつのルートとは、HTTPリクエストからハンドラ関数、もしくはルートが一致しなかったことを示す値へとマッピングする関数で、そして、ルートを論理和として組み合わせることで、特定のルートが一致しなければ別のルートを試す、という挙動を実現する。パイプライン処理にも順次的な合成がよく利用される。パイプラインのあるステージを最初に実行し、その後に別のステージを実行する、というものである。

#### 型

Types are not strictly part of functional programming but statically typed FP is the most popular form of FP and sufficiently important to warrant a mention. Types help compilers generate efficient code but types in FP are as much for the programmer as they are the compiler. Types express properties of programs, and the type checker automatically ensures that these properties hold. They can tell us, for example, what a function accepts and what it returns, or that a value is optional. We can also use types to express our beliefs about a program and the type checker will tell us if those beliefs are correct. For example, we can use types to tell the compiler we do not expect an error at a particular point in our code and the type checker will let us know if this is the case. In this way types are another tool for reasoning about code.

型は厳密には関数型プログラミングの一部ではないが、関数型言語の中でも静的型付けされたものが最もよく用いられているので、言及しておく価値がある。型はコンパイラが効率的なコードを生成するのを助けるが、関数型プログラミングにおける型はコンパイラだけでなくプログラマのためのものでもある。型はプログラムの特性を表現し、型チェッカーはそれらの特性が守られていることを自動的に保証してくれる。たとえば、関数が何を受け取り、何を返すかや、ある値がオプションであるといったことを型が示してくれる。さらに、型を使ってプログラムに対する自分の仮定を表現し、型チェッカーにそれが正しいかどうかを確認してもらうこともできる。たとえば、コードのある場所において、そこでエラーは発生しないと想定していることを型でコンパイラに伝えることができ、型チェッカーがそれが正しいかどうかを教えてくれる。このように、型はコードに対する推論を行うためのもうひとつのツールとなる。

Type systems push programs towards particular designs, as to work effectively with the type checker requires designing code in a way the type checker can understand. As modern type systems come to more languages they naturally tend to shift programmers in those languages towards a FP style of coding.

型システムはプログラムを特定の設計へと導く。型チェッカーと効果的に連携するためには、型チェッカーが理解できる作法でコードを設計することが要求されるからである。現代の型システムが多くの言語に導入されるにつれて、その言語のプログラマは自然と関数型プログラミングのスタイルへと移行する傾向がある。

### 関数型プログラミングは何でないか

In my view functional programming is not about immutability, or keeping to "the substitution model of evaluation", and so on. These are tools in service of the goals of enabling local reasoning and composition, but they are not the goals themselves. Code that is immutable always allows local reasoning, for example, but it is not necessary to avoid mutation to still have local reasoning. Here is an example of summing a collection of numbers. 

私の考えでは、関数型プログラミングの本質は不変性や「評価の置換モデル」などではない。これらは、局所推論と合成を可能にするための手段に過ぎず、目的ではないのである。たとえば、不変なコードは常に局所推論を可能にするが、可変性を避けなくても局所推論を行うことはできる。整数のコレクションを合計する例を考えてみよう。

```scala mdoc:silent
def sum(numbers: List[Int]): Int = {
  var total = 0
  numbers.foreach(x => total = total + x)
  total
}
```

In the implementation we mutate `total`. This is ok though! We cannot tell from the outside that this is done, and therefore all users of `sum` can still use local reasoning. Inside `sum` we have to be careful when we reason about `total` but this block of code is small enough that it shouldn't cause any problems.

この実装では `total` 変数の値を変更しているが、問題はない。そのような操作が行われていることは外部からは分からないし、`sum` の利用者は引き続き局所推論を行うことができる。`sum` の内部では `total` に関して慎重に考察する必要があるが、このコードブロックは十分に小さいため、特に問題を引き起こすことはないだろう。

In this case we can reason about our code despite the mutation, but the Scala compiler can determine that this is ok. Scala allows mutation but it's up to us to use it appropriately. A more expressive type system, perhaps with features like Rust's, would be able to tell that `sum` doesn't allow mutation to be observed by other parts of the system[]. Another approach, which is the one taken by Haskell, is to disallow all mutation and thus guarantee it cannot cause problems.

この場合、可変変数を使いながらも、我々はコードについて推論できるし、Scalaコンパイラもこれを問題なしと判断できる。Scalaは可変性を認めており、これを適切に使用するかどうかは我々に委ねられている。より表現力のある型システム、たとえばRustのような機能を持つ型システムであれば、`sum` がシステムの他の部分から観測される変更を認めていないことを保証できるだろう[^linear]。すべての可変性を禁止することによって、問題が生じないことを保証するアプローチもある。Haskellはこの方法を採用している。

Mutation also interferes with composition. For example, if a value relies on internal state then composing it may produce unexpected results. Consider Scala's `Iterator`. It maintains internal state that is used to generate the next value. If we have two `Iterators` we might want to combine them into one `Iterator` that yields values from the two inputs. The `zip` method does this.

This works if we pass two distinct generators to `zip`.

可変性は合成にも影響を与える。たとえば、ある値が内部状態に依存している場合、それを合成すると予期しない結果を生む可能性がある。Scalaの `Iterator` を考えてみよう。`Iterator` は次の値を生成するために内部状態を保持している。二つの `Iterator` があり、それらを組み合わせて二つの入力から値を返す `Iterator` を作りたいと思ったとする。このような合成をするには `zip` メソッドを使う。

この合成は、二つの別々のジェネレータを `zip` に渡せば、正しく動作する。

```scala mdoc:silent
val it = Iterator(1, 2, 3, 4)

val it2 = Iterator(1, 2, 3, 4)
```

```scala mdoc
it.zip(it2).next()
```

However if we pass the same generator twice we get a surprising result.

だが、同じジェネレータを二度使用すると、期待と異なる結果になる。

```scala mdoc:silent
val it3 = Iterator(1, 2, 3, 4)
```

```scala mdoc
it3.zip(it3).next()
```

The usual functional programming solution is to avoid mutable state but we can envisage other possibilities. For example, an [effect tracking system][effect-system] would allow us to avoid combining two generators that use the same memory region. These systems are still research projects, however.

関数型プログラミングにおける一般的な解決策は可変状態を避けることだが、他の方法も考えられる。たとえば、[エフェクト追跡システム][effect-system]を使用すれば、同じメモリ領域を使用する二つのジェネレータの組み合わせを避けることができる。しかし、これらのシステムはまだ研究段階にある。

So in my opinion immutability (and purity, referential transparency, and no doubt more fancy words that I have forgotten) have become associated with functional programming because they guarantee local reasoning and composition, and until recently we didn't have the language tools to automatically distinguish safe uses of mutation from those that cause problems. Restricting ourselves to immutability is the easiest way to ensure the desirable properties of functional programming, but as languages evolve this might come to be regarded as a historical artifact.

私の考えでは、不変性（および純粋性や参照透過性、そして覚えていないが何とかというもっと難解な概念）が関数型プログラミングと結びついているのは、それらが局所推論と合成を保証するからである。そして、最近まで、可変性の安全な使い方と問題を引き起こす使い方を自動的に区別できる言語ツールは存在しなかった。不変性の制約を受け入れることは、関数型プログラミングの望ましい特性を確保するための最も簡単な方法だが、言語が進化するにつれて、これが過去の遺物と見なされる可能性もある。

### なぜ重要なのか

I have described local reasoning and composition but have not discussed their benefits. Why are they are desirable? The answer is that they make efficient use of knowledge. Let me expand on this.

局所推論と合成について述べてきたが、その利点についてはまだ触れていない。これらが望ましいとされる理由は、それによって知識を効率的に活用できるからである。この点について詳しく説明しよう。

We care about local reasoning because it allows our ability to understand code to scale with the size of the code base. We can understand module A and module B in isolation, and our understanding does not change when we bring them together in the same program. By definition if both A and B allow local reasoning there is no way that B (or any other code) can change our understanding of A, and vice versa. If we don't have local reasoning every new line of code can force us to revisit the rest of the code base to understand what has changed. This means it becomes exponentially harder to understand code as it grows in size as the number of interactions (and hence possible behaviours) grows exponentially. We can say that local reasoning is compositional. Our understanding of module A calling module B is just our understanding of A, our understanding of B, and whatever calls A makes to B.

局所推論が重要なのは、コードベースの規模が大きくなっても、コードを理解する能力がスケールするからである。モジュールAとモジュールBをそれぞれ単独で理解できるなら、それらがひとつのプログラム中で一緒に用いられたとしても、その理解が変わることはない。定義上、AとBの両方が局所推論を可能とする場合、B（または他のコード）がAについての理解を変えることはないし、その逆も同じことが言える。もし局所推論ができなければ、新しいコードを1行追加するごとに、既存のコード全体を再確認して何が変わったのかを理解する必要が生じる。コードは、サイズが大きくなり、相互作用（およびそれに伴うかもしれない挙動）の数が急増するにつれて、理解するのが指数関数的に難しくなる。局所推論は合成的であると言える。モジュールAによるモジュールB呼び出しを理解したいなら、AおよびB、そしてAがBに対して行う呼び出しの内容を理解するだけでよい。

We introduced numbers and Lego as examples of composition. They have an interesting property in common: the operations that we can use to combine them (for example, addition, subtraction, and so on for numbers; for Lego the operation is "sticking bricks together") give us back the same kind of thing. A number multiplied by a number is a number. Two bits of Lego stuck together is still Lego. This property is called closure: when you combine things you end up with the same kind of thing. Closure means you can apply the combining operations (sometimes called combinators) an arbitrary number of times. No matter how many times you add one to a number you still have a number and can still add or subtract or multiply or...you get the idea. If we understand module A, and the combinators that A provides are closed, we can build very complex structures using A without having to learn new concepts! This is also one reason functional programmers tend to like abstractions such a monads (beyond liking fancy words): they allow us to use one mental model in lots of different contexts.

数とレゴを合成の例として見てきたが、これらには共通する興味深い性質がある。それらを組み合わせる操作（たとえば、数の場合は加算や減算、レゴの場合は「ブロックをくっつける」操作）を行うと、操作前と同じ種類のものが得られるという点である。数に数を掛ければ結果は数になるし、二つのレゴをくっつけたものも依然としてレゴである。この性質は閉包性と呼ばれ、ものを組み合わせたときに同じ種類のものが得られることを意味する。閉包性があるということは、合成操作（時にはコンビネータとも呼ばれる）を任意の回数適用できることを意味する。どれだけ1を足しても結果は数であり、さらに加算や減算、乗算を行うことができる。モジュールAについて理解しており、Aによって提供されるコンビネータが閉包性を持っていれば、新しい概念を学ばなくとも、Aを使って非常に複雑な構造を作ることができる。これは、関数型プログラマがモナドのような抽象概念を好む理由のひとつでもある（単に難しい言葉が好きだからではない）。これらの抽象概念は、ひとつのメンタルモデルをさまざまなコンテキストで使うことを可能にしてくれる。

In a sense local reasoning and composition are two sides of the same coin. Local reasoning is compositional; composition allows local reasoning. Both make code easier to understand.

ある意味、局所推論と合成は同じコインの裏表のような関係にある。局所推論は合成的であり、合成は局所推論を可能にする。どちらもコードを理解しやすくしてくれる。

### The Evidence for Functional Programming

I've made arguments in favour of functional programming and I admit I am biased---I do believe it is a better way to develop code than imperative programming. However, is there any evidence to back up my claim? There has not been much research on the effectiveness of functional programming, but there has been a reasonable amount done on static typing. I feel static typing, particularly using modern type systems, serves as a good proxy for functional programming so let's look at the evidence there.

In the corners of the Internet I frequent the common refrain is that [static typing has neglible effect on productivity][empirical-pl-luu]. I decided to look into this and was surprised that the majority of the results I found support the claim that static typing increases productivity. For example, the literature review in [this dissertation][merlin] (section 2.3, p16--19) shows a majority of results in favour of static typing, in particular the most recent studies. However the majority of these studies are very small and use relatively inexperienced developers---which is noted in the review by Dan Luu that I linked. My belief is that functional programming comes into its own on larger systems. Furthermore, programming languages, like all tools, require proficiency to use effectively. I'm not convinced very junior developers have sufficient skill to demonstrate a significant difference between languages.

To me the most useful evidence of the effectiveness of functional programming is that industry is adopting functional programming en masse. Consider, say, the widespread and growing adoption of Typescript and React. If we are to argue that FP as embodied by Typescript or React has no value we are also arguing that the thousands of Javascript developers who have switched to using them are deluded. At some point this argument becomes untenable.

This doesn't mean we'll all be using Haskell in five years. More likely we'll see something like the shift to object-oriented programming of the nineties: Smalltalk was the paradigmatic example of OO, but it was more familiar languages like C++ and Java that brought OO to the mainstream. In the case of FP this probably means languages like Scala, Swift, Kotlin, or Rust, and mainstream languages like Javascript and Java continuing to adopt more FP features.


### Final Words

I've given my opinion on functional programming---that the real goals are local reasoning and composition, and programming practices like immutability are in service of these. Other people may disagree with this definition, and that's ok. Words are defined by the community that uses them, and meanings change over time. 

Functional programming emphasises formal reasoning, and there are some implications that I want to briefly touch on. 

Firstly, I find that FP is most valuable in the large. For a small system it is possible to keep all the details in our head. It's when a program becomes too large for anyone to understand all of it that local reasoning really shows its value. This is not to say that FP should not be used for small projects, but rather that if you are, say, switching from an imperative style of programming you shouldn't expect to see the benefit when working on toy projects.

The formal models that underlie functional programming allow systematic construction of code. This is in some ways the reverse of reasoning: instead of taking code and deriving properties, we start from some properties and derive code. This sounds very academic but is in fact very practical, and how I develop most of my code.

Finally, reasoning is not the only way to understand code. It's valuable to appreciate the limitations of reasoning, other methods for gaining understanding, and using a variety of strategies depending on the situation.

[escape]: https://en.wikipedia.org/wiki/Escape_analysis
[substructural]: https://en.wikipedia.org/wiki/Substructural_type_system
[empirical-pl-luu]: https://danluu.com/empirical-pl/
[merlin]: https://web.cs.unlv.edu/stefika/documents/MerlinDissertation.pdf
[effect-system]: https://en.wikipedia.org/wiki/Effect_system
[monkey-patching]: https://ja.wikipedia.org/wiki/%E3%83%A2%E3%83%B3%E3%82%AD%E3%83%BC%E3%83%91%E3%83%83%E3%83%81
[fold]: https://www.cs.nott.ac.uk/~pszgmh/fold.pdf

[^linear]: The example I gave is fairly simple. A compiler that used [escape analysis][escape] could recognize that no reference to `total` is possible outside `sum` and hence `sum` is pure (or referentially transparent). Escape analysis is a well studied technique. In the general case the problem is a lot harder. We'd often like to know that a value is only referenced once at various points in our program, and hence we can mutate that value without changes being observable in other parts of the program. This might be used, for example, to pass an accumulator through various processing stages. To do this requires a programming language with what is called a [substructural type system][substructural]. Rust has such a system, with affine types. Linear types are in development for Haskell.
