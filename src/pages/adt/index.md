# 代数的データ型 {#sec:adt}

This chapter has our first example of a programming strategy: **algebraic data types**. Any data we can describe using logical ands and logical ors is an algebraic data type. Once we recognize an algebraic data type we get three things for free:

この章では、プログラミング戦略の最初の例として **代数的データ型** を取り上げる。論理積と論理和を使って表現できるデータはすべて代数的データ型である。代数的データ型をただ認識するだけで、以下の三つのものが手に入る。

- the Scala representation of the data;
- a **structural recursion** skeleton to transform the algebraic data type into any other type; and
- a **structural corecursion** skeleton to construct the algebraic data type from any other type.

- Scalaにおけるデータ表現
- 代数的データ型を他の型に変換するための**構造的再帰**の骨組み
- 他の型から代数的データ型を構築するための**構造的余再帰**の骨組み


The key point is this: from an implementation independent representation of data we can automatically derive most of the interesting implementation specific parts of working with that data.

重要なのは、実装非依存のデータ表現から、そのデータを扱うための実装固有の興味深い部分の多くを自動的に導き出すことができるという点である。

We'll start with some examples of data, from which we'll extract the common structure that motivates algebraic data types. We will then look at their representation in Scala 2 and Scala 3. Next we'll turn to structural recursion for transforming algebraic data types, followed by structural corecursion for constructing them. We'll finish by looking at the algebra of algebraic data types, which is interesting but not essential.

まず、データの例をいくつか紹介し、そこから、代数的データ型を導く共通の構造を抽出する。その後、Scala2とScala3での代数的データ型の表現を見ていく。次に、代数的データ型を変換するための構造的再帰に触れ、続いて、代数的データ型を構築するための構造的余再帰について説明する。最後に、代数的データ型の代数について考察する。これは本書の内容にとって必須ではないが、興味深いテーマである。

## 代数的データ型の構築

Let's start with some examples of data from a few different domains. These are simplified description but they are all representative of real applications.

まず、いくつかの異なる分野からデータ例を挙げてみよう。これらは簡略化されてはいるが、いずれも実際のアプリケーションにおける典型と言えるものである。

A user in a discussion forum will typically have a screen name, an email address, and a password. Users also typically have a specific role: normal user, moderator, or administrator, for example. From this we get the following data:

ディスカッションフォーラムのユーザは通常、表示名、メールアドレス、そしてパスワードといった情報をもつ。また、ユーザには通常、特定のロールが割り当てられる。たとえば、一般ユーザ、モデレータ、管理者など。これを以下に整理しておく。

- a user is a screen name, an email address, a password, and a role; and
- a role is normal, moderator, or administrator.

- ユーザは、表示名、メールアドレス、パスワード、ロールから構成される
- ロールは、一般ユーザ、モデレータ、管理者のうちのいずれかである

A product in an e-commerce store might have a stock keeping unit (a unique identifier for each variant of a product), a name, a description, a price, and a discount.

eコマースストアの製品は、在庫管理単位（製品の各バリエーションに対する一意の識別子）、名前、説明、価格、割引率といった情報をもっている。

In two-dimensional vector graphics it's typical to represent shapes as a path, which is a sequence of actions of a virtual pen. The possible actions are usually straight lines, Bezier curves, or movement that doesn't result in visible output. A straight line has an end point (the starting point is implicit), a Bezier curve has two control points and an end point, and a move has an end point.

二次元ベクターグラフィックスでは、図形をパスとして表現することが一般的に行われる。パスは、仮想ペンの一連の動作で構成される。よくある動作として、直線、ベジエ曲線、または目に見える結果を伴わない移動が挙げられる。直線には終点があり（始点は暗黙的に決まる）、ベジエ曲線には二つの制御点と終点があり、移動には終点がある。

What is common between all the examples above is that the individual elements---the atoms, if you like---are connected by either a logical and or a logical or. For example, a user is a screen name *and* an email address *and* a password *and* a role. A 2D action is a straight line *or* a Bezier curve *or* a move. This is the core of algebraic data types: an algebraic data type is data that is combined using logical ands or logical ors. Conversely, whenever we can describe data in terms of logical ands and logical ors we have an algebraic data type. 

上記すべての例に共通しているのは、個々の要素、いわば「原子」が論理積または論理和で結びつけられているという点である。たとえば、ユーザは表示名*および*メールアドレス*および*パスワード*および*ロールから構成されている。2Dアクションは、直線*または*ベジエ曲線*または*移動である。これが代数的データ型の核となる考え方で、代数的データ型とは、論理積や論理和で結びつけられたデータのことをいう。逆に言えば、データを論理積や論理和で表現できる場合、それは代数的データ型であると言える。

### 直和と直積

関数型プログラマとしては、単純な概念にちょっと難しそうな専門用語をつけずにはいられないものである。

- a **product type** means a logical and; and
- a **sum type** means a logical or.

- **直積型**は論理積を意味し、
- **直和型**は論理和を意味する

代数的データ型は直和型と直積型で構成される。

### 閉じた世界

Algebraic data types are closed worlds, which means they cannot be extended after they have been defined. In practical terms this means we have to modify the source code where we define the algebraic data type if we want to add or remove elements.

代数的データ型は閉じた世界であり、一度定義されると、それが拡張されることはない。要素を追加または削除したい場合、代数的データ型を定義した元のソースコードを変更する必要がある。

The closed world property is important because it gives us guarantees we would not otherwise have. In particular, it allows the compiler to check that we handle all possible cases when we use an algebraic data type. This is known as **exhaustivity checking**. This is an example of how functional programming prioritizes reasoning about code---in this case automated reasoning by the compiler---over other properties such as extensibility. We'll learn more about exhaustivity checking soon.

この閉じた世界の性質は重要である。これによって通常は得られない保証を得ることができる。特に、代数的データ型を使用する際に、すべてのありうるケースを処理しているかどうかをコンパイラがチェック可能となる。これを**網羅性チェック**と呼ぶ。このように、関数型プログラミングは、拡張性などの特性よりも、コードに対する推論――この場合はコンパイラによる自動的な推論――を優先する傾向がある。網羅性チェックについては、今後さらに学んでいく。
