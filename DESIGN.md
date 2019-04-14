# Parabuf

*Author: [Eric Conlon](https://twitter.com/econlon)*

*Last updated: 2019-04-14*

*NOTE: This is a mirror of the [original document](https://github.com/ejconlon/parabuf/blob/master/DESIGN.md).*

## Motivation

For years, I’ve wanted algebraic data types over the wire, but the best I could do was roll my own type system or use
custom codecs to map to host language features. Unfortunately, these strategies hit a wall in large, heterogeneous
projects, so people often reach for protocol buffers to define well-understood, high-performance interfaces and
encodings. However, protobuf brings with it a rigid and inexpressive type system that all but precludes frictionless
encoding of algebraic data types.

Here I’m going to propose one possible solution: parametrically polymorphic protobuf, or parabuf for short. I believe
it is possible to create a system with interesting and useful types that is wire-compatible with protobuf (and equally
well-performing). At the moment I’m not interested in preserving IDL syntax, adhering to existing codegenned interfaces,
or even supporting every feature (e.g. extensions, default values, annotations).

Before proto3 I had to define my own KV pair types to implement maps too many times to count. Thankfully, the project
maintainers have lifted polymorphic maps into the language now, but I don’t find it entirely satisfying. Why do they
get to define polymorphic types and we don’t? Let’s add them them to the IDL and see what shakes out.

Here are the advantages of parametric polymorphism at the IDL level:

-   Boilerplate reduction in type definitions that share structures <EXAMPLE>
-   Late binding of type parameters between IDL modules <EXAMPLE>
-   IDL-native Option types (no blessed wrapper types to direct codegen) <EXAMPLE>

If your language supports this polymorphism, you get additional benefits:

-   Boilerplate reduction in code that manipulates values of generated types <EXAMPLE>
-   Better reusability of generated types as domain types <EXAMPLE>

The primary restriction is that types representing serialized values must be monomorphic.  (Otherwise, how would you 
decode them if you didn’t know what all the type parameters were? Protobuf message encodings, unlike some Thrift 
encodings, do not carry type name on the wire.)  Explicit type synonyms would be useful for monomorphizing type 
constructors. <EXAMPLE> Later I’ll will consider type representation and reflection to quantify over type parameters.

## Implementation

For a first pass, all parameters to type constructors must themselves be types (base types or fully applied type
constructors). <EXAMPLE>

Since all user defined types in protobuf are monomorphic, most implementations use singleton-based codecs.  Every type
constructor is nullary, so it has only one codec implementation.  These implementations refer to each other by name.
<EXAMPLE>

In the polymorphic case, it becomes necessary to parameterize each type constructor codec by the codecs for its type
parameters.  <EXAMPLE>

First consider the codegen case for a monomorphic language.  Start by identifying all monomorphic type definitions
(including type synonyms).  Then recursively enumerate all fully applied type constructors in the bodies of those
definitions and assign unique type names to them.  Next rewrite the type definitions to refer to those unique names,
and finally with this generate protobuf IDL or codegen directly using the singleton strategy.  Note that we pay for
every unique specialization of a type constructor with additional generated types and code.  (This strategy is similar
to C++ template expansion.)

Codegen is even easier in languages that support typed parametric polymorphism.  In C++, for example, we can generate
templated structs and let monomorphic types direct template expansion. <EXAMPLE>  In Haskell, we can generate ADTs and
constrained typeclass instances and let monomorphic types direct constraint solving. <EXAMPLE>

For dynamic OO languages, we can use a hybrid approach. The key is to maintain a global mapping of type constructor
identifier to codec class constructor. Each constructor must be parameterized by a list of representations of its type
parameters. At construction time or codec time we use these representations to resolve and apply constructors. Note
that these codec classes are no longer singletons and generally cannot be forced into the companion class or static
context of a data class.

Schema Evolution and Subtyping

Often we need to ensure that a codec for type `A` can read data written by a codec for type `B`.  Safe schema changes
are those that redefine B as A such that `A <= B` in the sense of record subtyping; for example, adding an optional
field. Following proto3 we can use field numbers as labels. (Base types have no subtyping relation.)  The addition of
type parameters may add new clauses to the the proposition.  For example, `A <= B` implies `Option<A> <= Option<B>`.
Phantom type parameters add no clauses.

Extension 1: Reflection

We can decode polymorphic types or support existential types with the addition of some type representation. This representation would be something like a tree of type names including builtins
TypeRep - a tree of type names, including builtins <EXAMPLE>
Reflected - a pair of TypeRep and value of that type <EXAMPLE>

Extension 2: Higher kinded types

It’s not unreasonable to consider lifting the IDL restriction that all type parameters must be types, but it will
require more careful design. (In other words, we can consider allowing the definition of type constructors whose
arguments are themselves type constructors.) Some parametrically polymorphic languages do not support higher kinded
types, for example, so the embedding might not feel as natural as we want.

Call to Action

Are you a believer? Do you want to make this happen? Follow me on [Twitter](https://twitter.com/econlon)! Watch [this project](https://github.com/ejconlon/parabuf)! Review [this doc](https://github.com/ejconlon/parabuf/blob/master/DESIGN.md)! Send PRs!
