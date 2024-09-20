## Conclusions

We have covered a lot of material in this chapter. Let's recap the key points.

この章では多くの内容を扱った。ここで重要なポイントを振り返っておこう。

Algebraic data types allow us to express data types by combining existing data types with logical and and logical or. A logical and constructs a product type while a logical or constructs a sum type. Algebraic data types are the main way to represent data in Scala.

代数的データ型は、既存の型を論理積と論理和で組み合わせることによって作られるデータ型でである。論理積は直積型を構築し、論理和は直和型を構築する。代数的データ型はScalaでデータを表現する主要な方法である。

Structural recursion gives us a skeleton for transforming any given algebraic data type into any other type. Structural recursion can be abstracted into a `fold` method. 

構造的再帰は、与えられた代数的データ型を他の任意の型に変換するための骨組みを提供する。構造的再帰は `fold` メソッドとして抽象化することができる。

We use several reasoning principles to help us complete the problem specific parts of a structural recursion:

構造的再帰において問題固有の部分を完成させる助けとなる考え方の原則がいくつかある。

1. reasoning independently by case;
2. assuming recursion is correct; and
3. following the types.

1. 各ケースを独立して推論すること
2. 再帰呼び出しの結果は正しいと仮定すること
3. 型に従うこと

Following the types is a very general strategy that is can be used in many other situations.

型に従うことは他の多くの状況で利用できる極めて汎用的な戦略である。

Structural corecursion gives us a skeleton for creating any given algebraic data type from any other type. Structural corecursion can be abstracted into an `unfold` method. When reasoning about structural corecursion we can reason as we would for an imperative loop, or, if the input is an algebraic data type, use the principles for reasoning about structural recursion.

構造的余再帰は、任意の型から代数的データ型を作成するための骨組みを提供する。構造的余再帰は `unfold` メソッドとして抽象化できる。構造的余再帰について推論する際は、命令型のループの処理手順を考えるのと同じように進めるか、あるいは入力が代数的データ型である場合は、構造的再帰についての推論の原則を利用することができる。

Notice that the two main themes of functional programming---composition and reasoning---are both already apparent. Algebraic data types are compositional: we compose algebraic data types using sum and product. We've seen many reasoning principles in this chapter.

関数型プログラミングの二つの主要なテーマである合成と推論がすでにはっきり認識できる形で示されている点に注目してほしい。代数的データ型は合成的である。我々は直和型と直積型を使って代数的データ型を合成する。また、この章では多くの推論原則も見てきた。

I haven't covered everything there is to know about algebraic data types; I think doing so would be a book in its own right.
Below are some references that you might find useful if you want to dig in further, as well as some biographical remarks.

代数的データ型について知っておくべきすべてのことを説明したわけではない。これを完全に網羅すれば、それだけで一冊の本になるだろう。さらに掘り下げたい場合に役立つ参考文献や、著者の経歴に関するいくつかの補足を以下に示しておく。

Algebraic data types are standard in introductory material on functional programming. 
Structural recursion is certainly extremely common in functional programming, but strangely seems to rarely be explicitly defined as I've done here.
I learned about both from *How to Design Programs* [@felleisen18:htdp].

代数的データ型は、関数型プログラミングの入門資料では標準的に取り扱われている。構造的再帰も関数型プログラミングで非常によく使われるが、ここで私が行ったように明確に定義されることは珍しいようである。私はこれらの概念を *How to Design Programs* [@felleisen18:htdp] から学んだ。

I'm not aware of any approachable yet thorough treatment of either algebraic data types or structural recursion.
Both seem to have become assumed background of any researcher in the field of programming languages,
and relatively recent work is caked in layers of mathematics and obtuse notation that I find difficult reading.
The infamous *Functional Programming with Bananas, Lenses, Envelopes and Barbed Wire* [@10.1007/3540543961_7] is an example of such work.
I suspect the core ideas of both date back to at least the emergence of computability theory in the 1930s, well before any digital computers existed.

代数的データ型や構造的再帰について、手軽にアクセスできつつも詳細に扱った資料は私の知る限り存在しない。これらの概念は、プログラミング言語研究の分野ではもはや前提知識とされているようで、最近の研究は数学的で難解な表記に覆われており、私にとっては読みづらいものが多い。悪名高い *Functional Programming with Bananas, Lenses, Envelopes and Barbed Wire* [@10.1007/3540543961_7] はその一例である。両アイデアの核となる発想は、少なくとも計算可能性理論が登場した1930年代、デジタルコンピュータが存在するはるか以前にまで遡るのではないかと思われる。

The earliest reference I've found to structural recursion is *Proving Properties of Programs by Structural Induction* [@10.1093/comjnl/12.1.41]. 
Algebraic data types don't seem to have been fully developed, along with pattern matching, until [NPL][npl] in 1977. 
NPL was quickly followed by the more influential language [Hope][hope], which spread the concept to other programming languages.

構造的再帰に関する最も古い参考文献として私が見つけたのは、*Proving Properties of Programs by Structural Induction* [@10.1093/comjnl/12.1.41] である。代数的データ型とパターンマッチングの発展は、1977年の [NPL][npl] までは完全ではなかったようだが、その後すぐに、より影響力のある言語である [Hope][hope] が登場し、この概念が他のプログラミング言語にも広まった。

Corecursion is a bit better documented in the contemporary literature. *How to Design Co-Programs* [@GIBBONS_2021] covers the main ideas we have looked at here, while @10.1145/291251.289455 discusses uses of `unfold`.

余再帰については現代の文献ですこしくわしく記録されている。*How to Design Co-Programs* [@GIBBONS_2021] では、ここで取り上げた主なアイデアが説明されているし、[@10.1145/291251.289455] では `unfold` の使い方について議論されている。

*The Derivative of a Regular Type is its Type of One-Hole Contexts* [@mcbride01:deriv] describes the derivative of algebraic data types.

*The Derivative of a Regular Type is its Type of One-Hole Contexts* [@mcbride01:deriv] は代数的データ型の微分について記述している。

[banana]: https://ris.utwente.nl/ws/portalfiles/portal/6142049/meijer91functional.pdf
[structural-induction]: https://academic.oup.com/comjnl/article/12/1/41/311605
[npl]: https://en.wikipedia.org/wiki/NPL_(programming_language)
[hope]: https://en.wikipedia.org/wiki/Hope_(programming_language)
[htdc]: https://www.cs.ox.ac.uk/jeremy.gibbons/publications/copro.pdf
[unfold]: https://dl.acm.org/doi/pdf/10.1145/289423.289455
[deriv]: https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=7de4f6fddb11254d1fd5f8adfd67b6e0c9439eaa
