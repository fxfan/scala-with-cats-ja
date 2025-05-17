<!--

## Applications of Monoids

We now know what a monoid is---an abstraction of the concept of adding or combining---but where is it useful?
Here are a few big ideas where monoids play a major role.
These are explored in more detail in case studies later in the book.

### Big Data

In big data applications like Spark and Flink we distribute data analysis over many machines,
giving fault tolerance and scalability.
This means each machine will return results over a portion of the data,
and we must then combine these results to get our final result.
In the vast majority of cases this can be viewed as a monoid.

If we want to calculate how many total visitors a web site has received,
that means calculating an `Int` on each portion of the data.
We know the monoid instance of `Int` is addition, which is the right way to combine partial results.

If we want to find out how many unique visitors a website has received,
that's equivalent to building a `Set[User]` on each portion of the data.
We know the monoid instance for `Set` is the set union, which is the right way to combine partial results.

If we want to calculate 99% and 95% response times from our server logs,
we can use a data structure called a `QTree` for which there is a monoid.

Hopefully you get the idea. Almost every analysis that we might want to do over a large data set is a monoid,
and therefore we can build an expressive and powerful analytics system around this idea.
This is exactly what Twitter's Algebird and Summingbird projects have done.
We explore this idea further in the map-reduce case study in Section [@sec:map-reduce].

### Distributed Systems

In a distributed system,
different machines may end up with different views of data.
For example,
one machine may receive an update that other machines did not receive.
We would like to reconcile these different views,
so every machine has the same data if no more updates arrive.
This is called *eventual consistency*.

A particular class of data types support this reconciliation.
These data types are called conflict-free replicated data types (CRDTs).
The key operation is the ability to merge two data instances,
with a result that captures all the information in both instances.
This operation relies on having a monoid instance.
We explore this idea further in the CRDT case study.

### Monoids in the Small

The two examples above are cases where monoids inform the entire system architecture.
There are also many cases where having a monoid around makes it easier to write a small code fragment.
We'll see lots of examples in the remainder of this book.


-->

## モノイドの応用

モノイドが加算や結合の概念を抽象化したものであることは理解できた。だが、どのような場面で役立つのだろうか。ここでは、モノイドが重要な役割を果たす大きなアイデアをいくつか紹介する。これらについては、後の章でケーススタディとして詳細に取り上げる。

### ビッグデータ

ビッグデータを扱う Spark や Flink のようなアプリケーションでは、データ分析を多数のマシンに分散させることで、耐障害性とスケーラビリティを実現する。つまり、各マシンがデータの一部を解析した結果を返した後、それらを結合して最終結果を得る必要がある。各マシンの出力結果とそれらを結合する演算は、多くの場合、モノイドと見なすことができる。

ウェブサイトの総訪問者数を計算したい場合、データの各部分に対して `Int` 型の値をひとつ出力するような計算が行われる。Cats によって提供されるデフォルトの `Int` モノイドインスタンスがもつ演算は加算であり、このユースケースにおける部分的な結果同士を結合するのに適している。

ユニーク訪問者数を計算したいのであれば、データの各部分に対して `Set[User]` オブジェクトを構築することになる。Cats によって提供される `Set` モノイドインスタンスがもつ演算は和集合であり、これらの部分的な結果同士を結合するのに適している。

サーバログから99%および95%応答時間を計算したい場合、`QTree` というデータ構造を利用できる。このデータ構造にはモノイドが存在する。

これでイメージがつかめたのではないかと思う。大規模データセットに対する分析はほぼすべてモノイドとして扱うことができるため、モノイドを中心に強力かつ汎用性の高い分析システムを構築することができる。これはまさに Twitter の Algebird や Summingbird プロジェクトが行ったことでもある。このアイデアについては、[@sec:map-reduce]節において MapReduce のケーススタディでさらに詳しく探求する。

### 分散システム

分散システムでは、マシンによってデータの見え方が異なる場合がある。たとえば、あるマシンがデータの更新を受信したが、他のマシンはそれを受け取っていないかもしれない。こうした異なる見え方を調整し、すべての更新が届いた後は、すべてのマシンが同じデータをもつようにしたい。これを*結果整合性（eventual consistency）*と呼ぶ。

ある種類のデータ型がこの調整をサポートする。それらのデータ型は、競合しない複製可能データ型（conflict-free replicated data type）略して CRDT と呼ばれる。CRDT の鍵となる操作は、ふたつのデータインスタンスをマージし、両方の情報をすべて含んだ結果を得ることである。この操作はモノイドインスタンスに依存している。このアイデアについては CRDT のケーススタディでさらに詳しく探る。

### 小規模なモノイドの活用

上記のふたつの例は、モノイドがシステム全体のアーキテクチャを形作る役割を果たしているケースだった。一方で、モノイドがあることによってちょっとしたコードを簡単に書ける場面も多く存在する。本書の残りの部分では、そうした例を多数紹介していく。
