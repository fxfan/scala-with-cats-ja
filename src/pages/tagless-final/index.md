<!--

# Tagless Final Interpreters

In this chapter we'll explore the codata approach to interpreters, building up to a strategy known as **tagless final**.
Along the way we will build two interpreters: one for terminal interaction and one for user interfaces.

We've seen the duality between data and codata in many places, starting with Chapter [@sec:codata]. 
This chapter will begin by applying that duality to build an interpreter using codata, which contrasts with the data approach we saw in Section [@sec:interpreters:reification].
This will illustrate the technique and give us a concrete example to discuss its shortcoming.
In particular we'll see that extensibility is limited, a problem we first encountered in Section [@sec:codata:extensibility].

Solving the problem of extensibility, otherwise known as the **expression problem**, will lead us to tagless final. 
In the context of interpreters, solving the expression problem means allowing extensibility of both the programs we write and the interpreters that run them.
We'll start with the standard encoding of tagless final in Scala, and see that it is a bit painful to use in practice.
We'll then develop an alternative encoding that is easier to use. 
Solving the expression problem allows for very expressive code but it adds complexity, so we'll finish by talking about when tagless final is appropriate and when it's best to use a different strategy.


-->

# Tagless Final インタープリタ

本章では、インタプリタに対する余データ的アプローチを探求し、**Tagless Final** として知られる戦略へと至る道をたどる。その過程で、ターミナル操作用とユーザインターフェース用の、二種類のインタプリタを構築する。

データと余データの双対性については[@sec:codata]章をはじめさまざまな箇所で見てきた。本章では、この双対性を応用して余データを用いたインタプリタを構築する。これは、[@sec:interpreters:reification]節で見たデータを用いるアプローチとは対照的である。この作業を通じて技法について描き出し、またこれを具体例とすることでこの技法の欠点について議論できるようにする。特に、拡張性が制限されるという欠点が明らかになるだろう。これは[@sec:codata:extensibility]節ですでに触れた問題である。

この拡張性の問題、すなわち**式の問題**の解決を追っていくと、Tagless Final という戦略にたどり着く。インタープリタという文脈において式の問題を解決するとは、作成するプログラムとそれを実行するインタープリタの両方について拡張性をもたせることを意味する。まずは Scala における Tagless Final の標準的なエンコーディングから始めるが、実際にはすこし使いにくいことがわかるだろう。そこで、もっと扱いやすい別のエンコーディングを開発する。式の問題を解決することで非常に表現力豊かなコードを書けるようになるが、代償として複雑性が増す。そこで最後に、Tagless Final が適している場合と、別の戦略を用いたほうがよい場合について述べる。
