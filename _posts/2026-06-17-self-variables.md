---
layout: post
title: "Self variables"
date: 2026-06-17
---
In Narya there are two ways to define a record type.  One is the more common, where the fields are named and referred to like local variables:
```
def Σ (A : Type) (B : A → Type) : Type ≔ sig (
  fst : A,
  snd : B fst )
```
The other uses a "self variable" belonging to the record type itself, with the fields accessed using the usual field projection syntax from that variable:
```
def Σ (A : Type) (B : A → Type) : Type ≔ sig (
  x .fst : A,
  x .snd : B (x .fst) )
```
(You can't mix the two syntaxes: you have to pick one or the other and use it for all the fields.)

Although self variables are a common way to access members and methods in mainstream object-oriented programming languages, they aren't particularly common in proof assistants.  But Narya not only permits them for record types, it *mandates* them for codatatypes.  The *only* way to define streams is
```
def Stream (A : Type) : Type ≔ codata [
| x .head : A
| x .tail : Stream A ]
```

My original reason for using self variables was that in [displayed type theory](https://arxiv.org/abs/2311.18781), we wanted to consider a kind of codatatype that uses the self variable in a *different* way than just projecting out its fields.  In particular, we have the coinductive definition of semi-simplicial types:
```
def SST : Type ≔ codata [
| X .z : Type
| X .s : X .z → SST⁽ᵈ⁾ X ]
```
However, in the process of implementation I discovered that self variables are much easier to deal with than field variables.  For instance, rather than the types of the fields being stored as a telescope, each in a longer context than the previous one, they can all be stored separately in a context that is only *one* variable longer (the self variable).  So in the internals of Narya, everything is self variables; the translation from field variables (when the user writes the first syntax for a record type) happens at typechecking time.

Today I realized another benefit of self variables.  Namely, field variables are kind of funny in that they look like variables, but they are not subject to alpha-equivalence.  If you rename the fields of a record type, you get a different record type: isomorphic to the first one, but different.  This, in turn, prevents arbitrary alpha-equivalence from being applied to other ordinary variables in enclosing binders.

Normally, if I have a nested λ-abstraction such as
```
x y z ↦ f x y z
```
I'm free to rename the variable `y` to anything I want, as long as it doesn't shadow an outer variable such as `x`, at the cost of possibly also renaming other inner variables such as `z`.  For instance, I can write that abstraction as
```
x w z ↦ f x w z
```
and also
```
x z z′ ↦ f x z z′
```
This order of alpha-renaming is convenient when printing a term, transforming the internal De Bruijn representation into a named variable one: as we descend through abstractions, at each point we choose a name for the bound variable that doesn't conflict with any previously used names for outer variables.  Thus, at the point of (re)naming `y` we are free to choose `z`, and then later we get to `z` and we just choose a different name for it there.

This is what breaks if field variables can't be renamed.  An equivalent way to write the above definition of `Σ` is
```
def Σ : (A : Type) (B : A → Type) → Type ≔ A B ↦ sig (
  fst : A,
  snd : B fst )
```
But in the λ-abstraction
```
A B ↦ sig ( fst : A, snd : B fst )
```
the ordinary variable `B` can't be renamed to `fst`, because it would conflict with the *inner* field variable `fst`.  We can't print `Σ` as
```
A fst ↦ sig ( fst′ : A, snd : fst fst′ )
```
because that would be a different record type, with a field called `fst′` rather than `fst`.

One might think of trying to rename "backwards", renaming any outer variables that conflict with a field variable name.  But aside from the difficulty of implementing something like that (if we're displaying the term by a normal recursive traversal, other parts of the term referring to that outer variable by its original name might already have been displayed), the outer variable could *also* be a non-renameable field variable.

For instance, here's a definition written using self variables that can't be written directly using field variables:
```
def foo₁ : Type ≔ sig (
  x .fst : Type,
  x .snd : sig (
    y .fst : Type,
    y .snd : x .fst ))
```
If we try to write it with field variables, we get stuck:
```
def foo₂ : Type ≔ sig (
  fst : Type,
  snd : sig (
    fst : Type
    snd : ? ))
```
The `?` is supposed to refer to the *outer* field variable `fst`, but that is shadowed by the *inner* field variable `fst`, and we can't rename either of them.  The only thing we can do is something like
```
def foo₃ : Type ≔ sig (
  fst : Type,
  snd : let outer_fst ≔ fst in sig (
    fst : Type
    snd : outer_fst ))
```
But it's a bit much to expect a pretty-printer to insert `let`-bindings.  So if the user writes the above definition of `foo₃`, then internally it gets normalized (making the `let`-binding disappear) and represented like `foo₁`, and the pretty-printer gets stuck trying to print it like `foo₂`.  Thus, having the option of a user-facing syntax that uses self-variables is necessary for the pretty-printer to have something to output in this case.  Probably it would stick with field variables for the outer record type, and then when it reaches the inner one and discovers a conflict, it falls back to self variables for that:
```
def foo₄ : Type ≔ sig (
  fst : Type,
  snd : sig (
    y .fst : Type
    y .snd : fst ))
```

Now, I should admit that most other proof assistants would not let you write *any* of these definitions, so the question of field variable alpha-equivalence doesn't arise.  In proof assistants like Rocq, Agda, and Lean, a record type definition is a top-level declaration that can't appear as a term inside any other definition.  Interestingly, the language feature I know of that comes closest to what Narya allows is OCaml's [labeled tuples](https://ocaml.org/manual/5.4/labeledtuples.html):
```
fst:int * snd:(fst:int * snd:string)
```
However, these are not dependently typed, so the issue of names doesn't come up.

I should also admit that although Narya will let you *write* `foo₁`, `foo₃`, and `foo₄`, the question of how to print them doesn't come up at present, because record types are generative and nominal.  That is, all of those definitions define a new type `foo` (or whatever) that doesn't reduce to any previously existing thing such as `sig ( ... )`; instead the `≔ sig ( ... )` serves to specify the *behavior* of `foo`.  However, it's useful to be able to inspect the definitions of existing types without having to search for their definition, and I'm working on an extension that will print them.  In that case, this issue will arise, and the printer will fall back to self variables in case of a conflict of field names, as I described above.  (And in fact, I discovered this issue while working on that extension: Claude wrote some code that uniquified the field names, producing `fst′` and so on, and I said Hey, that's not right!)

Until now, I always thought of self-variable syntax for records (as opposed to codata) as a bit of an odd beast.  I implemented it in response to a request from a user (who can identify themselves if they wish) who wanted to write some code that (I thought) was at least a little semantically suspect, referring to the self variable of a record in a non-field-access way similar to the codata definition of `SST`, and I've always been ambivalent about it.  However, the realization that self variables are necessary to consistently print all nested record types makes me much happier about them (although when we get around to implementing consistency checks like termination and positivity, we should probably check that the self variable of a record type *is* only used to access its fields, and at least issue a warning if it is not).  Perhaps we should even deprecate field variables!
