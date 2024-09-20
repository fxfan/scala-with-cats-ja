# 余データとしてのオブジェクト {#sec:codata}

In this chapter we will look at **codata**, the dual of algebraic data types.
Algebraic data types focus on how things are constructed.
Codata, in contrast, focuses on how things are used.
We define codata by specifying the operations that can be performed on the type.
This is very similar to the use of interfaces in object-oriented programming, and this is the first reason that we are interested in codata: codata puts object-oriented programming into a coherent conceptual framework with the other strategies we are discussing.

この章では、代数的データ型の双対である余データについて見ていく。代数的データ型はモノがどのように構築されるかに焦点を当てているが、それに対して余データはモノの利用方法に焦点を当てている。余データは、型に対して実行できる操作を指定することで定義される。これはオブジェクト指向プログラミングにおけるインターフェースの使い方と非常に似ており、これが余データに注目する理由のひとつである。オブジェクト指向プログラミングは、余データという概念によって、我々が今議論している戦略と整合性のある概念的枠組みの中に位置づけられる。

We're not only interested in codata as a lens to view object-oriented programming.
Codata also has properties that algebraic data does not.
Codata allows us to create structures with an infinite number of elements, such as a list that never ends or a server loop that runs indefinitely. 
Codata has a different form of extensibility to algebraic data.
Whereas we can easily write new functions that transform algebraic data, we cannot add new cases to the definition of an algebraic data type without changing the existing code.
The reverse is true for codata. We can easily create new implementations of codata, but functions that transform codata are limited by the interface the codata defines.

しかし、余データに注目するのはオブジェクト指向プログラミングを捉えるためだけではない。余データには、代数的データにはない特性がある。たとえば、余データを使うことで、無限の要素を持つ構造、終わりのないリストや永遠に実行されるサーバーループのようなものを作成することができる。また、余データは拡張性の点でも代数的データとは異なる形をもっている。代数的データ型では、新しい変換関数を簡単に作成できるが、新しいケースを定義に追加するには既存のコードを変更する必要があります。余データはその逆で、新しい実装を簡単に作成できるが、余データを変換する関数は余データが定義するインターフェースに制約される。

In the previous chapter we saw structural recursion and structural corecursion as strategies to guide us in writing programs using algebraic data types.
The same holds for codata.
We can use codata forms of structural recursion and corecursion to guide us in writing programs that consume and produce codata respectively.

前章では、代数的データ型を使用してプログラムを記述するための戦略として、構造的再帰と構造的余再帰を取り上げた。余データについても同様で、余データを消費するプログラムを書く際には構造的再帰を、余データを生成するプログラムを書く際には構造的余再帰を使用することができる。

We'll begin our exploration of codata by more precisely defining it and seeing some examples. 
We'll then talk about representing codata in Scala, and the relationship to object-oriented programming.
Once we can create codata, we'll see how to work with it using structural recursion and corecursion, using an example of an infinite structure.
Next we will look at transforming algebraic data to codata, and vice versa.
We will finish by examining differences in extensibility.

まず、余データをより正確に定義し、いくつかの例を見ていこうと思う。次に、Scalaにおける余データの表現方法とオブジェクト指向プログラミングとの関係について説明する。余データの作成方法を理解した後は、無限構造の例を使って、構造的再帰や構造的余再帰を用いた余データの操作方法を学ぶ。その後、代数的データ型と余データの相互変換について考察し、最後に拡張性の違いを検証する。

A quick note about terminology before we proceed. We might expect to use the term algebraic codata for the dual of algebraic data, but conventionally just codata is used. I assume this is because data is usually understood to have a wider meaning than just algebraic data, but codata is not used outside of programming language theory. For simplicity and symmetry, within this chapter I'll just use the term data to refer to algebraic data types.

進める前に用語について簡単に触れておきたい。代数的データの双対としては「代数的余データ」という用語を期待されるかもしれないが、慣例的に単に「余データ」という用語が使われる。これは、データという言葉が、一般的に代数的データに限定されず、より広い意味を持つのに対して、余データはプログラミング言語理論の外では使われないからだと考えられる。この章では、簡潔さと対称性のために、代数的データ型を指す際に「データ」という用語を使用する。
