# Indexed Types

In this chapter we look at **indexed types**. Both data and codata can be indexed. An indexed type is a type constructor, so a type like `F[_]`, along with a set of types that can fill in the type parameters for the constructor. Let's say the set of types is `Int`, `String`, and `Option[A]`. Then, for a type constructor `F` we can construct an indexed type from the set `F[Int]`, `F[String]`, and `F[Option[A]]`. As the name suggests, the indices act as indexes into this set of types.

This is a very abstract definition and doesn't help us understand how indexed types are useful. We'll see a lot of details and examples in this chapter, but let's start with a more useful high-level overview. When we're treating a type as indexed data we create elements from our set of types. We can think of this as providing a proof that some type parameter, `A` in the example above, is equal to some other type. When we're treating a type as indexed codata we require 

**TODO: Complete**

As you might expect, indexed data and indexed codata are duals. We can 

We'll begin by revisiting the definition of algebras we gave in Section [@sec:interpreters:reification], where we said algebra consists of three different kinds of methods: constructors, combinators, and interpreters. Indexed types allows us to do two things:

- We can restrict where constructors and combinators can be used. We can think of representing some state using a type parameter of `F`, and we can only call particular methods when we are in the correct state. In this case we are working with **indexed codata**.

- We restrict the types produced interpreters, enabling us to create type-safe interpreters that guarantee they only encounter particular states when they run. Again these constraints are represented using type parameters. In this case we are working with **indexed data**.

Indexed data are more usually known as **generalized algebraic data types**. Indexed codata are sometimes known as **typestate**. Both can make use of what is known as **phantom types**. Indeed, an early name for indexed data was **first-class phantom types**.

As you might expect, indexed data and indexed codata are dual are one another. I've tried to represent this in the description above. Another way to look at this is terms of type equalities, which are proofs or guarantees that a particular type parameter is equal to a particular concrete type. When we work with indexed codata we require the user supplies us with these type equalities. When we work with indexed data we discover these type equalities as we destructure the data.

