# Indexed Types

In this chapter we look at **indexed types**. Both data and codata can be indexed. An indexed type is a type constructor, so a type like `F[_]`, along with a set of types that can fill in the type parameters for the constructor. Let's say the set of types is `Int`, `String`, and `Option[A]`. Then, for a type constructor `F` we can construct an indexed type from the set `F[Int]`, `F[String]`, and `F[Option[A]]`. As the name suggests, the indices act as indexes to select an element from this set.

This is a very abstract definition. How are indexed types actually useful? We'll see a lot of details and examples in this chapter, but let's start with a high-level overview:

- indexed data allows us to 

- indexed codata allows us to use some state, determined by the type parameters of `F`, to control when certain methods on `F` can be called.

For data, indexed types are more usually known as **generalized algebraic data types**. Indexed codata is sometimes known as **typestate**. Both can make use of what is known as **phantom types**. Indeed, an early name for indexed data was **first-class phantom types**.
