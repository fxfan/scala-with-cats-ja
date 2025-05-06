<!--

## Conclusions

In this chapter we've explored codata, the dual of data. Codata is defined by its interface---what we can do with it---as opposed to data, which is defined by what it is. More formally, codata is a product of destructors, where destructors are functions from the codata type (and, optionally, some other inputs) to some type. By avoiding the elements of object-oriented programming that make it hard to reason about---state and implementation inheritance---codata brings elements of object-oriented programming that accord with the other functional programming strategies. In Scala we define codata as a `trait`, and implement it as a `final class`, anonymous subclass, or an object.

We have two strategies for implementing methods using codata: structural corecursion, which we can use when the result is codata, and structural recursion, which we can use when an input is codata. Structural corecursion is usually the more useful of the two, as it gives more structure (pun intended) to the method we are implementing. The reverse is true for data.

We saw that data is connected to codata via fold: any data can instead be implemented as codata with a single destructor that is the fold for that data. The reverse is also: we can enumerate all potential pairs of inputs and outputs of destructors to represent codata as data. However this does not mean that data and codata are equivalent. We have seen many examples of codata representing infinite structures, such as sets of all even numbers and streams of all natural numbers. We have also seen that data and codata offer different forms of extensibility: data makes it easy to add new functions, but adding new elements requires changing existing code, while it is easy to add new elements to codata but we change existing code if we add new functions.

The earliest reference I could find to codata in programming languages is @HAGINO1989629. This is much more recent than algebraic data, which I think explains why codata is relatively unknown. There are some excellent recent papers that deal with codata. 
I highly recommend *Codata in Action* [@downen2019codata], which inspired large portions of this chapter. 
*Exploring Codata: The Relation to Object-Orientation* [@DRP-201905-Sullivan] is also worthwhile.
*How to Add Laziness to a Strict Language Without Even Being Odd* [@wadler1998add] is an older paper that discusses the implementation of streams, and in particular the difference between a not-quite-lazy-enough implementation they label odd and the version we saw, which they call even. These correspond to `Stream` and `LazyList` in the Scala standard library respectively.
*Classical (Co)Recursion: Programming* [@DBLP:journals/corr/abs-2103-06913] is an interesting survey of corecursion in different languages, and covers many of the same examples that I used here.
Finally, if you really want to get into the weeds of the relationship between data and codata, *Beyond Church encoding: Boehm-Berarducci isomorphism of algebraic data types and polymorphic lambda-terms* [@kiselyov05:beyond] is for you.


-->

## まとめ

この章では、データの双対である余データについて探求した。データが「それが何であるか」によって定義されるのに対し、余データは「それで何ができるか」というインターフェースによって定義される。より形式的にいえば、余データはデストラクタの積である。ここでいうデストラクタとは、余データ型（および必要に応じて他の入力）を受け取って、何らかの型を返す関数のことを指す。余データによって、オブジェクト指向プログラミングのうち、推論を困難にする可変状態や実装継承のような要素を避け、関数型プログラミングの戦略に合致する要素を取り込むことができる。Scala では、余データを `trait` として定義し、それを `final class`、匿名サブクラス、あるいはオブジェクトとして実装する。

余データを使ったメソッドの実装にはふたつの戦略がある。結果が余データである場合には構造的余再帰を使用し、入力が余データである場合には構造的再帰を使用する。構造的余再帰のほうが、実装するメソッドをより構造的にしてくれるので、有用である場合が多い。データの場合にはその逆が成り立つ。

データと余データは `fold` を介して結びつけられる。任意のデータは、そのデータの `fold` を単一のデストラクタとする余データとして実装することができる。逆も同様で、すべてのデストラクタの入力と出力のペアを列挙することで、余データをデータとして表現することができる。ただし、そのことはデータと余データが等価であることを意味しない。すべての偶数の集合やすべての自然数のストリームなど、余データが無限の構造を表現できる例をいくつも見てきた。また、データと余データが異なる拡張性を提供することも学んだ。データは新しい機能の追加が容易だが、新しいバリアントの追加には既存コードの変更を必要とする。一方、余データでは新しいバリアントを簡単に追加できるが、新しい機能を追加するには既存コードを変更する必要がある。

プログラミング言語における余データへの最も古い言及は、私が見つけたかぎりでは @HAGINO1989629 である。これは代数的データと比べてはるかに新しく、余データが比較的知られていないのはそのせいであるように思われる。余データに関する最近の優れた論文がいくつかある。まず *Codata in Action* [@downen2019codata] を強くお勧めしたい。この論文は本章の大部分にインスピレーションを与えてくれた。また、*Exploring Codata: The Relation to Object-Orientation* [@DRP-201905-Sullivan] も読む価値がある。すこし古いが、*How to Add Laziness to a Strict Language Without Even Being Odd* [@wadler1998add] はストリームの実装、特に彼らが odd と呼ぶ不完全な遅延評価実装と、彼らが even と呼んでいる、本章で見たスタイルとの違いを論じている。これらはそれぞれ Scala 標準ライブラリの `Stream` と `LazyList` に対応する。*Classical (Co)Recursion: Programming* [@DBLP:journals/corr/abs-2103-06913] は、さまざまな言語における余再帰についての興味深い調査であり、ここで扱った多くの例をカバーしている。最後に、データと余データの関係を徹底的に掘り下げたいなら、*Beyond Church encoding: Boehm-Berarducci isomorphism of algebraic data types and polymorphic lambda-terms* [@kiselyov05:beyond] をお勧めする。
