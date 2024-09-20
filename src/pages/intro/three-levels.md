## コードを考えるための3つのレベル

Let’s start thinking about thinking about programming, with a model that describes three different levels that we can use to think about code. The levels, from highest to lowest, are paradigm, theory, and craft. Each level provides guidance for the ones below.

ここでは、コードについて考えるための三つの異なるレベルを記述したモデルを紹介する。それらを用いて、プログラミングについて考えることについて考えていこう。それらのレベルとは、上から順にパラダイム（paradigm）、理論（theory）、そして技能（craft）である。各レベルは、それより下位のレベルに対して指針を提供してくれる。

パラダイムレベルとは、オブジェクト指向や関数型プログラミングといったプログラミングパラダイムのことを指す。これらの用語には馴染みのある方も多いと思うが、プログラミングパラダイムとは一体何だろうか。私にとって、プログラミングパラダイムの核心は、よいコードとはどういうものかを緩やかに定める一連の原則である。また、「これらの原則に従ったコードは、そうでないコードよりも優れている」という暗黙の主張もパラダイムには含まれている。関数型プログラミングにおける原則は合成（composition）と推論（reasoning）であると私は考えている。この二つの原則については改めて説明する。一方、オブジェクト指向プログラマは、たとえばSOLID原則をコーディングにおける意思決定の指針として挙げるかもしれない。

パラダイムは、それが異なる実装戦略の中からいずれかを選ぶための基準を示してくれるという点で重要である。プログラミングの課題には多くの解決策が存在するが、どのアプローチを取るべきかを、パラダイムによって示される原則にもとづいて判断することができる。たとえば、関数型プログラマであれば、特定の実装についてどれだけ簡単に理解し実行結果を予測できるか、それがどれだけ合成可能であるかを検討するだろう。パラダイムがないというのは、そういった選択を行うための基準がないということである。

理論レベルでは、パラダイムの広範な原則が具体的で明確に定義されたテクニックに翻訳される。それらのテクニックは、パラダイム内の多くの言語に適用できるが、この段階でもまだコードレベルの具体性はもたない。オブジェクト指向の世界では、デザインパターンがその一例であり、関数型プログラミングにおいては、代数的データ型がその一例である。多くの関数型プログラミング言語、たとえばHaskellやO'Camlは代数的データ型をサポートしているし、複数のパラダイムをまたぐRust、Scala、Swiftといった言語も同様にサポートしている。

理論レベルは、我々がプログラミング戦略の大部分を見出す場所となる。

At the craft level we get to actual code, and the language specific nuance that goes into it. An example in Scala is the implementation of algebraic data types in terms of `sealed trait` and `final case class` in Scala 2, or `enum` in Scala 3. There are many concerns at this level that are important for writing idiomatic code, such as placing constructors on companion objects in Scala, that are not relevant at the higher levels.

技能レベルでは、具体的なコードや、それと関連する言語固有のニュアンスを取り扱う。Scalaにおける例として、代数的データ型はScala2では `sealed trait` と `final case class` を使って実装され、Scala3では `enum` で実装される。このレベルには、Scalaでコンパニオンオブジェクトにコンストラクタを配置するなど、上位レベルには存在しなかった、慣用的なコードを書くための多くの考慮事項が存在する。

In the next section I'll describe the functional programming paradigm. The remainder of this book is primarily concerned with theory and craft. The theory is language agnostic but the craft is firmly in the world of Scala. Before we move onto the functional programming paradigm are two points I want to emphasize:

次節では関数型プログラミングのパラダイムについて説明し、本書の残りの部分では主に理論と技能に焦点を当てる。理論は特定の言語に依存しないが、技能はScalaに特化したものとなる。関数型プログラミングのパラダイムへと進む前に、二つの点を強調しておきたい。

1. Paradigms are social constructs. They change over time. Object-oriented programming as practiced todays differs from the style originally used in Simula and Smalltalk, and functional programming todays is very different from the original LISP code.

1. パラダイムは社会的構築物であり、時間とともに変化する。現在のオブジェクト指向プログラミングは、元々SimulaやSmalltalkで使用されていた流儀とは異なるし、今日の関数型プログラミングも、初期のLISPコードとは大きく異なる。

2. The three level organization is just a tool for thought. In real world is more complicated.

2. この三層構造は、あくまで思考のためのツールに過ぎない。現実の世界はもっと複雑である。
