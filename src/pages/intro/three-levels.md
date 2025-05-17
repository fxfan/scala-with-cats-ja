<!--

## Three Levels for Thinking About Code {#sec:intro:three-levels}

Let's start thinking about thinking about programming, with a model that describes three different levels that we can use to think about code. The levels, from highest to lowest, are paradigm, theory, and craft. Each level provides guidance for the ones below.

The paradigm level refers to the programming paradigm, such as object-oriented or functional programming. You're probably familiar with these terms, but what exactly is a programming paradigm? To me, the core of a programming paradigm is a set of principles that define, usually somewhat loosely, the properties of good code. A paradigm is also, implicitly, a claim that code that follows these principles will be better than code that does not. For functional programming I believe these principles are composition and reasoning. I'll explain these shortly. Object-oriented programmers might point to, say, the SOLID principles as guiding their coding decisions.

The importance of the paradigm is that it provides criteria for choosing between different implementation strategies. There are many possible solutions for any programming problem, and we can use the principles in the paradigm to decide which approach to take. For example, if we're a functional programmer we can consider how easily we can reason about a particular implementation, or how composable it is. Without the paradigm we have no basis for making a choice.

The theory level translates the broad principles of the paradigm to specific well defined techniques that apply to many languages within the paradigm. We are still, however, at a level above the code. Design patterns are an example in the object-oriented world. Algebraic data types are an example in functional programming. Most languages that are in the functional programming paradigm, such as Haskell and O'Caml, support algebraic data types, as do many languages that straddle multiple paradigms, such as Rust, Scala, and Swift.

The theory level is where we find most of our programming strategies.

At the craft level we get to actual code, and the language specific nuance that goes into it. An example in Scala is the implementation of algebraic data types in terms of `sealed trait` and `final case class` in Scala 2, or `enum` in Scala 3. There are many concerns at this level that are important for writing idiomatic code, such as placing constructors on companion objects in Scala, that are not relevant at the higher levels.

In the next section I'll describe the functional programming paradigm. The remainder of this book is primarily concerned with theory and craft. The theory is language agnostic but the craft is firmly in the world of Scala. Before we move onto the functional programming paradigm are two points I want to emphasize:

1. Paradigms are social constructs. They change over time. Object-oriented programming as practiced today differs from the style originally used in Simula and Smalltalk, and functional programming today is very different from the original LISP code.

2. The three level organization is just a tool for thought. In the real world it is more complicated.


-->

## コードを考えるための三つのレベル {#sec:intro:three-levels}

それでは、プログラミングについて考えることについて考えていこう。コードについて考えるための三つの異なるレベルを記述したモデルを用いる。それらのレベルとは、上から順にパラダイム（paradigm）、理論（theory）、そして技法（craft）である。各レベルは、それより下位のレベルに対して指針を提供してくれる。

パラダイムレベルとは、オブジェクト指向や関数型プログラミングといったプログラミングパラダイムのことを指す。これらの用語には馴染みのある方も多いと思うが、プログラミングパラダイムとは一体何だろうか。私にとって、プログラミングパラダイムの核心は、よいコードとはどういうものかを緩やかに定める一連の原則である。また、「これらの原則に従ったコードは、そうでないコードよりも優れている」という暗黙の主張もパラダイムには含まれている。関数型プログラミングにおける原則は合成（composition）と推論（reasoning）であると私は考えている。このふたつの原則については改めて説明する。一方、オブジェクト指向プログラマは、たとえば SOLID 原則をコーディングにおける意思決定の指針として挙げるかもしれない。

パラダイムは、異なる実装戦略の中からいずれかを選ぶための基準を示してくれるという点で重要である。プログラミングの課題には多くの解決策が存在するが、どのアプローチを取るべきかを、パラダイムによって示される原則にもとづいて判断することができる。たとえば、関数型プログラマであれば、特定の実装についてどれだけ簡単に理解し実行結果を予測できるか、それがどれだけ合成可能であるかを検討するだろう。パラダイムがないというのは、そういった選択を行うための基準がないということである。

理論レベルでは、パラダイムのもつ概略的な原則が具体的で明確に定義されたテクニックに翻訳される。それらのテクニックは、パラダイム内の多くの言語に適用できるが、この段階でもまだコードレベルの具体性はもたない。オブジェクト指向の世界では、デザインパターンがその一例であり、関数型プログラミングにおいては、代数的データ型がその一例である。多くの関数型プログラミング言語、たとえば Haskell や O'Caml は代数的データ型をサポートしているし、マルチパラダイム言語である Rust、Scala、Swift なども同様にサポートしている。

プログラミング戦略の大部分は理論レベルにおいて見出される。

技法レベルでは、具体的なコードや、それに関連する言語固有のニュアンスを取り扱う。Scala における例として、代数的データ型は Scala2 では `sealed trait` と `final case class` を使って実装され、Scala3 では `enum` で実装される。このレベルには、Scala でコンパニオンオブジェクトにコンストラクタを配置するなど、上位レベルには存在しなかった、慣用的なコードを書くための多くの考慮事項が存在する。

次節では関数型プログラミングのパラダイムについて説明し、本書の残りの部分では主に理論と技法に焦点を当てる。理論は特定の言語に依存しないが、技法は Scala に特化したものとなる。関数型プログラミングのパラダイムへと進む前に、ふたつの点を強調しておきたい。

1. パラダイムは社会的構築物であり、時間とともに変化する。現在のオブジェクト指向プログラミングは、元々 Simula や Smalltalk で使用されていた流儀とは異なるし、今日の関数型プログラミングも、初期の LISP のコードとは大きく異なる。

2. この三層構造は、あくまで思考のためのツールに過ぎない。現実の世界はもっと複雑である。
