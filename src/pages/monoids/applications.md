## モノイドの応用 Applications of Monoids

We now know what a monoid is---an abstraction of the concept of adding or combining---but where is it useful?
Here are a few big ideas where monoids play a major role.
These are explored in more detail in case studies later in the book.

モノイドが、加算や結合の概念を抽象化したものであることは理解できた。だが、どのような場面で役立つのだろうか。ここでは、モノイドが重要な役割を果たすいくつかの大きなアイデアを紹介する。これらについては、後の章でケーススタディとして詳細に取り上げる。

### ビッグデータ Big Data

In big data applications like Spark and Flink we distribute data analysis over many machines,
giving fault tolerance and scalability.
This means each machine will return results over a portion of the data,
and we must then combine these results to get our final result.
In the vast majority of cases this can be viewed as a monoid.

ビッグデータを扱うSparkやFlinkのようなアプリケーションでは、データ分析を多数のマシンに分散させることで、耐障害性とスケーラビリティを実現する。つまり、各マシンがデータの一部を解析した結果を返した後、それらを結合して最終結果を得る必要がある。各マシンの出力結果とそれらを結合する演算は、多くの場合、モノイドと見なすことができる。

If we want to calculate how many total visitors a web site has received,
that means calculating an `Int` on each portion of the data.
We know the monoid instance of `Int` is addition, which is the right way to combine partial results.

ウェブサイトの総訪問者数を計算したい場合、データの各部分に対して `Int` 型の値をひとつ出力するような計算が行われる。Catsによって提供されるデフォルトの `Int` モノイドインスタンスがもつ演算は加算であり、このユースケースにおける部分的な結果同士を結合するのに適している。

If we want to find out how many unique visitors a website has received,
that's equivalent to building a `Set[User]` on each portion of the data.
We know the monoid instance for `Set` is the set union, which is the right way to combine partial results.

ユニーク訪問者数を計算したいのであれば、データの各部分に対して `Set[User]` オブジェクトを構築することになる。Catsによって提供される `Set` モノイドインスタンスがもつ演算は和集合であり、これらの部分的な結果同士を結合するのに適している。

If we want to calculate 99% and 95% response times from our server logs,
we can use a data structure called a `QTree` for which there is a monoid.

サーバログから99%および95%応答時間を計算したい場合、`QTree` というデータ構造を利用できる。このデータ構造にはモノイドが存在する。

Hopefully you get the idea. Almost every analysis that we might want to do over a large data set is a monoid,
and therefore we can build an expressive and powerful analytics system around this idea.
This is exactly what Twitter's Algebird and Summingbird projects have done.
We explore this idea further in the map-reduce case study in Section [@sec:map-reduce].

これでイメージが掴めたのではないかと思う。大規模データセットを用いたほぼすべての分析はモノイドとして扱うことができるため、モノイドを中心に強力かつ汎用性の高い分析システムを構築することができる。これはまさにTwitterのAlgebirdやSummingbirdプロジェクトが行ったことでもある。このアイデアについては、[@sec:map-reduce]節においてMapReduceのケーススタディでさらに詳しく探求する。

### 分散システム Distributed Systems

In a distributed system,
different machines may end up with different views of data.
For example,
one machine may receive an update that other machines did not receive.
We would like to reconcile these different views,
so every machine has the same data if no more updates arrive.
This is called *eventual consistency*.

分散システムでは、マシンによってデータの見え方が異なる場合がある。たとえば、あるマシンがデータの更新を受信したが、他のマシンはそれを受け取っていないかもしれない。こうした異なる見え方を調整し、すべての更新が届いた後は、すべてのマシンが同じデータをもつようにしたい。これを*結果整合性（eventual consistency）*と呼ぶ。

A particular class of data types support this reconciliation.
These data types are called conflict-free replicated data types (CRDTs).
The key operation is the ability to merge two data instances,
with a result that captures all the information in both instances.
This operation relies on having a monoid instance.
We explore this idea further in the CRDT case study.

ある種類のデータ型がこの調整をサポートする。それらのデータ型は、競合しない複製可能データ型（conflict-free replicated data type）略してCRDTと呼ばれる。鍵となる操作は、二つのデータインスタンスを両方がもつすべての情報を反映させながらマージする能力をもっている。この操作はモノイドインスタンスに依存している。このアイデアについては、CRDTのケーススタディでさらに詳しく探る。

### 小規模なモノイドの活用 Monoids in the Small

The two examples above are cases where monoids inform the entire system architecture.
There are also many cases where having a monoid around makes it easier to write a small code fragment.
We'll see lots of examples in the remainder of this book.

上記の二つの例では、モノイドがシステムの全体アーキテクチャに影響を与えるケースを見た。一方で、モノイドがあることで小さなコード片を簡単に書ける場面も多く存在する。本書の残りの部分では、そうした例を多数紹介していく。
