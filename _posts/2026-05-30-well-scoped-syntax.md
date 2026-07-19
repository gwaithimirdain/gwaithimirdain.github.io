---
layout: post
title: "Well-scoped syntax and type-level categories"
date: 2026-05-30
---

In coding Narya, one of my general principles is to use types as much as possible to eliminate the possibility of errors statically (that is, at compile time).  Which is, of course, what types are for, from a programming perspective.

In a fully dependently-typed language, it's theoretically possible to eliminate *all* errors statically.  If you can write a sufficiently detailed type for a function that basically says "this function does exactly what it's supposed to do", then as soon as you define a function of that type that compiles successfully, you can be absolutely sure (at least insofar as you trust the compiler) that it will, in fact, do exactly what it's supposed to do.  There are various problems with this theoretical dream, but I still find it useful as an ideal touchstone of typed programming.

One of the reasons we often can't achieve that dream is that we may not be programming in a fully dependently typed language.  Whether it's practical yet to write large-scale code in an actually dependently typed language is a discussion for another day; suffice it for now to say that at present I am not.  Narya is written in OCaml, a language which is in many ways a joy to program in, but it is not fully dependently typed: in general, types can't depend on terms.  However, types can depend on *other types*, and the type system of OCaml is flexible enough that we can encode a whole separate layer of "type-level terms" for our types to depend on.

This is a well-known technique in OCaml (and other languages that support it, like Haskell).  I'm using it for lots of things in Narya, but one of the most important is *static scoping*.

## Well-scoped indices

Like many implementations of type theory, Narya represents variables (in typechecked terms) by [De Bruijn indices](https://en.wikipedia.org/wiki/De_Bruijn_index).  However, in Narya all De Bruijn indices are all *intrinsically well-scoped*.  That is, it's impossible for the code to accidentally create a De Bruijn index that's too large and points outside all the enclosing binders: the *OCaml* typechecker won't allow it.

This is accomplished by indexing terms by a "phantom type" that records the length of their context, as a "type-level natural number".  Here is how we encode the natural numbers as types:

```ocaml
type zero = private Dummy_zero
type 'a suc = private Dummy_suc
```

(If you're going to read this blog you should probably know at least basic [OCaml syntax](https://ocaml.org/manual/5.4/index.html).)  Thus, among the usual types `int`, `string` and so on, we now also have types `zero`, `zero suc`, `zero suc suc`, and so on, which we view as a copy of the natural numbers.  Now we can define the type of De Bruijn indices as a [GADT](https://ocaml.org/manual/5.4/gadts-tutorial.html) depending on the length of their context:

```ocaml
type _ index =
  | Top : 'a suc index
  | Pop : 'a index -> 'a suc index
```

This is just the usual definition of the "finite types" in dependent type theory, but now with the indexing natural number being a *type* rather than an element of a type of natural numbers.  Thus we have the following indices:

- there is nothing in `zero index`
- `Top : zero suc index`
- `Top : zero suc suc index` and `Pop Top : zero suc suc index`
- `Top : zero suc suc suc index` and `Pop Top : zero suc suc suc index` and `Pop (Pop Top) : zero suc suc suc index`

and so on.  Now when we define terms, we use these indices to denote variables:

```ocaml
type _ term =
  | Var : 'a index -> 'a term
  ...
```

and *voila*: it's impossible for De Bruijn indices to be out of scope.

## Dimension-indexed indices

In fact, what what Narya actually does is more complicated than this.  In observational higher type theories, at least the way Narya implements them, each "variable" in the context can actually be a collection of variables forming a cube of some dimension.  An ordinary single variable is a zero-dimensional cube.  A one-dimensional cube consists (in the arity-2 case) of three variables: two endpoints and a path.

```
(x₀ : A) (x₁ : A) (x₂ : Id A x₀ x₁)
```

For instance, these are what we add to the context when defining an equality of functions in `Id (A → B) f₀ f₁`, since that identity type is (isomorphic to)

```
(x₀ : A) (x₁ : A) (x₂ : Id A x₀ x₁) → Id B (f₀ x₀) (f₁ x₁)
```

When we write this on a page, or even in proof assistant code, we write the variables `x₀`, `x₁`, `x₂` in linear order, and usually think of simply abstracting over three ordinary variables one at a time.  But one of the insights that makes Narya work is that internally, it's better to treat these three variables together as a "bundle", using a data structure that associates them directly to the faces of a cube (more on that in some other post, maybe), and *only* linearize them at the boundary of concrete syntax when interacting with the user.  This eliminates a whole class of potential bugs that could arise from inconsistent ways to order the faces of a cube, and makes the logic more intuitive and geometric: you are always just iterating over the faces of a cube, with no need to think about the order in which it happens.

Now, however, when we put a whole cube of variables into the context at once, we need more information to access a single variable: not just a De Bruijn index, which points to the whole *cube*, but a *face* of that cube.  This introduces a new source of potential "scoping" errors: what if the face has the wrong dimension to match the cube in the context?

Fortunately, we can also eliminate this possibility statically.  We represent *dimensions* as types: for now you can just think of them as another copy of the type-level natural numbers above.  Faces of a cube are then indexed by their domain and codomain, for instance an element of `(zero suc, zero suc suc) face` is a 1-dimensional face of a 2-dimensional cube (of which there are four).  And the parameter of a term, that above was just the *length* of a context, now becomes a *type-level list of dimensions* -- in fact, a [backwards list](https://github.com/RedPRL/ocaml-bwd).  Here are the constructors of type-level backwards lists:

```ocaml
type emp = private Dummy_emp
type ('xs, 'x) snoc = private Dummy_snoc
```

Thus, for instance, here is a type-level backwards list of type-level natural numbers corresponding to `[1; 2; 0]`:

```ocaml
(((emp, zero suc) snoc, zero suc suc) snoc, zero) snoc
```

One final change is to parametrize the type of indices by the dimension that some index points to in a type-level backwards list of dimensions:

```ocaml
type (_, _) index =
  | Top : ('n, ('a, 'n) snoc) index
  | Pop : ('n, 'a) index -> ('n, ('a, 'k) snoc) index
```

Now we can say the variables that appear in terms are determined by an index and a face *of the same dimension*:

```ocaml
type _ term =
  | Var : ('n, 'a) index * ('k, 'n) face -> 'a term
  ...
```

and the OCaml typechecker will guarantee at compile-time that we never have a mismatch.  (This isn't exactly what's in the Narya code, but it's the basic idea.)

## Free categories

My current project is to enhance Narya with [Multimodal Type Theory](https://arxiv.org/abs/2011.15021).  (I always intended to do that, but I'm doing it now because I have hopes that it will be useful in completing the implementation of HOTT; more on that later.)  In MTT, contexts aren't built just out of typed variables, but also *locks* arising from modalities.  Specifically, each context has a *mode*, and if `μ : p → q` is a modality (mode morphism) then a `q`-context can be `μ`-locked to become a `p`-context.

Of course, this opens up yet another class of potential bugs: the source or target of a lock modality might not match up correctly.  And so, of course, I set out on a quest to eliminate such errors statically.

Actually -- no, I didn't, not to start with.  At first, I put in the locks without any static checking.  And I got things mostly working, but there were some weird bugs that I didn't entirely understand, but which seemed like they had to do with putting the wrong modalities or modes in the wrong places.  So, instead of spending the time tracking down those specific bugs, I decided to rip everything apart and put in static modality-dependence in the context.  This had the result of not only squashing the bugs I can see now, but preventing any future such bugs from occurring.

This may seem like a strange way to code, but it's worked for me numerous times.  In particular, when I do this, the bugs I started out looking at do, in fact, get fixed, even though it's not always obvious to me exactly at what point I fixed them!  Sometimes I have a guess about more or less the point at which they got fixed, but other times I have no idea.  The latter is what happened with modal locks: to make the code compile under the new statically-typed locking regime, I had to rewrite enough of it more or less from scratch that I still have no idea where the bug was in the old version.  But it doesn't matter, because the OCaml typechecker promises me that the new version is free of (those) bugs.

Personally, I find this kind of coding to be a joy.  Among other things, it means that I spend more of my time thinking about how things *should* work and doing them correctly, rather than trying to figure out why something *isn't* working.  And it's also very category-theoretic, which of course appeals to me as a category theorist.

Why is it category-theoretic?  I mean that partly in [Tom Leinster's sense](https://golem.ph.utexas.edu/category/2010/03/a_perspective_on_higher_catego.html) that doesn't refer at all to categories: it solves problems in a conceptual way by moving to a higher level of generality.  But more concretely, this approach tends to incorporate more of the category-theoretic framework of mathematical descriptions of type theory into the implementation.

Static context locking is a case in point.  The parameter of a context should now be a type-level backwards list that contains two kinds of elements: dimensions, representing variables, and modalities, representing locks.  (Each modality, like each dimension, is itself represented by a type; I'll say more in some other post about how that works.)  In addition, each dimension is *also* paired with a modality, because variables in the context can be "modally annotated".

However, not any old list of (modality, dimension) pairs and modalities is a valid "context length", because the modes have to match up: if `μ : p → q` is a modality, then only a `q`-context can be `μ`-locked, and likewise only a variable in a `q`-context can be annotated by `μ`.  Mathematically, these context lengths, or "type-level contexts", are the *free category* generated by a [quiver](https://ncatlab.org/nlab/show/quiver) whose vertices are the modes, and with two kinds of edges: each modality `μ : p → q` yields a "lock" edge from `p` to `q`, while each pair `(μ , n)` of a modality `μ : p → q` with a dimension `n` yields a "variable" edge from `q` to `q` (adding a variable doesn't change the *mode* of the context).

The foundation of static context locking is, therefore, a library for type-level free categories.  Doing this right required a fair amount of boilerplate, but fortunately these days AI makes it easy to generate boilerplate.  [Here](https://github.com/gwaithimirdain/narya/blob/2d95834d220cd47ff400bb7ee2d9b3a7a9fb3e69/lib/util/category.ml) is the definition of type-level categories, and [here](https://github.com/gwaithimirdain/narya/blob/2d95834d220cd47ff400bb7ee2d9b3a7a9fb3e69/lib/util/path.ml) is the free category on a quiver (whose morphisms are "paths" of edges in the quiver).

Finally, as usual, once we have a good abstraction, it turns out to be useful for lots of things.  For example:

- When extending a context by a telescope, we "compose morphisms" in this "context category".

- There is a functor from the category of modes to this "context category" which makes every modality into a lock.  Locking a context is then just composition with the image of some modality under this functor.  (This is less trivial than it seems because for technical reasons, the basic "lock" constructor actually just locks by a *generating* modality.)

- There is also a functor in the other direction that discards all variables and composes up the locks in a context.  This is important when looking up a variable in a context: we need the composite of all the locks to its right to know what sort of "key" 2-cell is required to unlock it.

Multimodal Narya is not yet ready for release, but to whet your appetite here is some code that now compiles under the mode theory for an idempotent comonad `□`, exhibiting the functoriality and monad operations:

```
def □ (A :□| Type) : Type ≔ data [ box. (x :□| A) ]

def □map (A B :□| Type) (f :□| A → B) : □ A → □ B ≔ [ box. x ↦ box. (f x) ]

def ε (A :□| Type) (u :□ A) : A ≔ match u [ box. x ↦ x ]

def △ (A :□| Type) (u :□ A) : □ (□ A) ≔ match u [ box. x ↦ box. (box. x) ]

def □ε∘△ (A :□| Type) (u :□ A) : Id (□ A) (□map (□ A) A (ε A) (△ A u)) u
  ≔ match u [ box. x ↦ refl (box. x) ]

def ε□∘△ (A :□| Type) (u :□ A) : Id (□ A) (ε (□ A) (△ A u)) u ≔ match u [
| box. x ↦ refl (box. x)]
```
