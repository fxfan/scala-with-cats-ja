\partimage[width=.5\linewidth]{src/pages/parts/part1.png}
\part{Foundations}

In this first part of the book we're building the foundational strategies on which the rest of the book will build and elaborate. 
In Chapter [@sec:adt] we look at algebraic data types, which are our main way of modelling data.
We turn to codata in Chapter [@sec:codata], which is the opposite, or dual, or algebraic data.
Type classes are the focus on Chapter [@sec:type-classes], while fundamentals of interpreters are discussed in Chapter [@sec:interpreters]. These four strategies all describe code artifacts. For example, we can label part of code as an algebraic data type or a type class. We'll also see strategies that help us write code but don't necessarily end up directly reflected in it, such as following the types.

本書の最初のパートでは、それ以降の部分が拠って立ち、発展させていく、基盤となる戦略を構築していく。[@sec:adt]章ではデータモデリングの主要な手段である代数的データ型について見ていく。[@sec:codata]章では余データ型について取り扱う。これは、代数的データ型の対極、もしくは双対となるものである。型クラスについては[@sec:type-classes]章で、そしてインタプリタの基礎については[@sec:interpreters]章で議論する。以上4つの戦略はすべてコードの構造を記述するものである。たとえば、コードの一部を代数的データ型や型クラスとしてラベリングすることができる。また、型に従うといった、コードを書く上で役立つが直接コードに反映されるわけではない戦略も登場する。
