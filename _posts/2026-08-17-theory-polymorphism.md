---
layout: post
title: "Theory polymorphism"
date: 2026-08-17
---

## The problem

To first approximation, a proof assistant is an implementation of some type theory.  However, some proof assistants actually implement many different type theories, and the user can specify which one to use in any given development.  For example, in Agda the user can switch on or off features like cubical type theory, two-level type theory, pattern matching that relies on axiom K, strict propositions, rewriting, and so on, by using command-line options or the `OPTIONS` pragma.

Narya behaves like this too.  As its `README` says,

> Narya is eventually intended to be a proof assistant implementing Multi-Modal, Multi-Directional, Higher/Parametric/Displayed Observational Type Theory.

Multi-modal means that the user can specify an arbitrary mode theory; that's now [implemented](https://narya.readthedocs.io/en/latest/modal.html) (although fully realizing the "arbitrary" requires a bit of OCaml coding --- often done easily by an AI --- a number of common mode theories are supplied).  Multi-directional parametric type theory means that the user can specify any number of "directions" of parametricity, each with their own arity; this is work in progress, as currently we allow only one direction at a time (which might be a HOTT direction with fibrancy), but it can have any arity from 0 to 9.

Whenever there are theory selection flags like this, we encounter the problem of how different source files that were written with different flags can interact when loading each other.  Agda manages this with [infective and coinfective](https://agda.readthedocs.io/en/latest/tools/command-line-options.html#checking-options-for-consistency) options, which place requirements on which options can be active in two source files in order for one of them to be allowed to load the other.  In effect, this means that there is a poset of "sets of options", in which `S ≤ T` means that a file using options `S` can be loaded by a file using options `T`.  This gives a sort of polymorphism where a "core" file can be written using a set of options towards the bottom of the poset, allowing it to then be loaded by many other developments that use different option sets.

Currently, Narya's poset of option-sets is, technically, discrete: one file can only be loaded from another one if they use exactly the same type theory (mode theory and parametricity direction).  I say "technically" because there's a loophole, arising from the fact that type theory options are currently *only* specified on the command-line, allowing the same source file to be typechecked under more than one set of options.  What Narya currently does in this case is store the type theory options in the compiled `.nyo` file, and if a file using different options tries to load that file, instead of loading the `.nyo` it re-typechecks the source `.ny` and saves a new compiled file instead.  So in fact the same "core" library file can be used in multiple developments, but if you use the same disk file it'll get re-typechecked every time you switch.  (Agda's documentation suggests that it also does this sort of re-typechecking when options change.)

However, because Narya's type theory flags are so rich, this sort of "source reuse" is not really enough.  For instance, the most basic "core" files are probably going to be written using the default trivial mode theory, with one mode called `Type`.  Such a file can then also be loaded using a different mode theory, but --- currently --- *only* if that other mode theory also has a mode *called* `Type`, and then only *at* that mode called `Type`.  What we'd really like is to be able to write a core library file using the trivial mode theory, but then load that same library into a multi-mode theory *once for each mode*.  Similarly, we should be able to write a library file developing the modal type theory of a comonad, and then load that library into a multi-mode theory specialized to *any* comonad that happens to exist in its mode theory, and so on.

In particular, this means that the collection of Narya's theory options is *not a poset but a category*: we have to be able to specify *how* the theory options of one file are transformed into those of another file that we want to load it into.  It also means that specifying theory flags on the command-line is inherently unstable; the code of a file itself is really the only reliable source for the information about what type theory should be used for it.

## A solution

Here's my current plan for dealing with this.  Command-line theory flags will be deprecated; instead every source file must begin with zero or more `option` commands specifying its type theory.  Option commands can't go anywhere except at the beginning of a file.

The first kind of `option` command is `option parametric`.  Its syntax looks something like this:

```
option parametric p (rel, Br) ≔ 2
```

This example is equivalent to the old sequence of command-line flags `-parametric -direction p,rel,Br -arity 2`.  You can add `(external)` after the arity to make parametricity external.

Eventually, you'll be able to have up to 25 `option parametric` commands defining different directions of parametricity, each denoted by a lowercase roman letter, with `e` reserved for the HOTT direction that'll always be present.  (If anyone ever needs more than 25 directions of parametricity, we'll cross that bridge when we come to it.)  But until multi-direction is implemented, only one `option parametric` command will be allowed, and if present it will disable HOTT the way `-parametric` does.  If no `option parametric` command is given, then HOTT mode will remain active by default.

The second kind of `option` command is `option modal`.  Its syntax looks something like this:

```
option modal ≔ adjunction
```

This example is equivalent to the old command-line flag `-adjunction`.  To get `-discrete-adjunction` you can add `(discrete ≔ p)` at the end; this syntax is to support multi-directional parametricity later, in which case some parametricity directions could be discrete but not others.  Other aspects of the mode theory will also become configurable in this command, e.g. you can specify manually which modalities have which kind of transparency to use as match windows.  You can also rename the modes, modalities, and cells with suffixes like `(modalities ≔ tri sq)` similar to the `-modalities` command-line flag.

It may seem a little verbose to have every source file start with `option` commands, but I think it shouldn't be worse than Agda's omnipresent `OPTIONS` line, and the mode and parametricity theories chosen are so basic to a development that I think it's a good idea to have them recorded in every file.  I'm trying to make the syntax fairly concise, so each `option` command will usually fit on one line.  For example, the current `-dtt` will be equivalent to

```
option parametric d ≔ 1 (external)
option modal ≔ tconn (discrete ≔ d)
```
which we might allow to continue to be abbreviated by `option dtt`.

Now what happens when you `import` another source file?  If that file uses the exact same `option`s as the current file -- including the *letter names* of the parametricity directions, though not necessarily their prefix names like `rel`/`Br`, and not necessarily any *renamings* of the modal data -- then you can import it directly.  But otherwise, you must specify a *translation*.  A translation consists of:

- A *function* from the parametricity directions of the imported file to those of the importing file, preserving arity, externality, and fibrancy; and
- A *2-functor* from the mode theory of the imported file to the mode theory of the importing file, preserving tangibility, transparency, discreteness (along the function on parametricity directions), and parametricity lockers (if any).

Just like a mode theory, specifying such a 2-functor will (at least at first) have to be done in OCaml.  That is, Narya will ship with a bunch of 2-functors connecting different mode theories, and you can pick from these by name when building a translation, or have an AI write a new one and add it to Narya.  (AI is very good at coding new mode theories; I expect it will be just as good at coding functors between them.)  The function on parametricity directions can easily be specified by the user, however, and then preservation of all the other data can be checked automatically.

I'm thinking of a syntax something like the following, which makes translation appear part of the Yuujinchou machinery for [import modifiers](https://narya.readthedocs.io/en/latest/imports-and-scoping.html#import-modifiers).  Suppose `core.ny` is a library written using the trivial mode theory; then in some other file we can write:

```
option modal ≔ adjunction

import "core" | translate trivial_at_disc, renaming . ds
import "core" | translate trivial_at_type, renaming . ty
```

Then, for instance, `ds.ℕ` will denote the natural numbers at mode `Disc`, while `ty.ℕ` will denote the natural numbers at mode `Type`, and we only had to define `ℕ` once in `core`.  And if we need to also rename parametricity directions, we would use something like this:

```
import "param" | translate some_functor (p ↦ q, a ↦ b)
```

As far as the implementation goes, it should be possible to mechanically walk *typechecked* terms applying a translation, so that for instance `core` only needs to be typechecked once and can then be imported multiple times under different translations.  In fact, Narya *already* walks all the typechecked terms when loading a compiled file in a "linking" pass, updating all the constant and import ID numbers to those currently in effect; so with some finesse we should be able to fold the translation into that.  And this sort of translation seems like it will be necessary anyway in order to implement fibrancy at arbitrary modes and modalities (which is a hole in the current implementation).

Some additional work will be needed because a translated import must be "generative": the same constant imported under two different translations yields two *different* objects, unlike the behavior of ordinary imports which simply give multiple names to the same thing.  But I don't think that should be insurmountable.

Overall, I think this is the best idea I've come up with for managing a multi-theory proof assistant.  But it's not set in stone yet, and I'd be interested to hear reactions!

## Towards universe polymorphism

It's worth comparing something like this to existing frameworks for universe polymorphism, as both involve some sort of "parametrization" of a definition or a development that's not fully internal to the theory: "for all universe levels `i`" is morally similar to "for all modes `Type`".

The most obvious difference is that universe polymorphism usually happens at the level of individual definitions, while what I'm proposing here would happen at the level of source files (although it could also be applied to "sections" or "modules").  But either one could be implemented the other way in theory.  On one hand, we could allow each definition to specify separately its directions of parametricity and mode theory, and to specify a translation whenever it uses any other definition.  I don't think that would be very ergonomic, since the category of translations is not a poset and they couldn't in general be inferred automatically, but it's possible.  On the other hand, we could parametrize an entire source file (or section/module) by a choice of universes, and then specialize those universes when it's imported.  Universe-polymorphic proof assistants do often support this, although I think it's generally treated as simply a shorthand for parametrizing each definition separately, like ordinary "section variables".

Another important difference is in the explicit specification of a translation.  Applied to universe polymorphism, this is reminiscent of Conor McBride's [Crude but Effective Stratification](https://mazzo.li/epilogue/index.html%3Fp=857&cpage=1.html).  Specifically, if we imagine each module declaring some collection of universe level variables and inequalities between them, and a translation specifying how to map each level variable in the imported file to a "level expression" in the importing file (just as a 2-functor between mode theories can map each generating modality to a not-necessarily-generating modality), then we obtain basically a file-level version of the "ℋ-polymorphic type theory" described by Favonia, Angiuli, and Mullanix in [An Order-Theoretic Analysis of Universe Polymorphism](https://favonia.org/files/mugen.pdf).  Their ℋ is a monad on strict posets describing the "level expressions", and they show how a suitable choice of ℋ recovers a generalization of CbES.  (The original CbES declares exactly one level variable `i` for each definition, and the level expressions are of the form `i+n` for some external natural number `n`, so that translations are just "upward shifts".)

If this approach works well for parametricity and mode theories, I might try extending it to universe levels.  It's not obvious to me whether it would be a problem to have only file-level universe polymorphism.  I think we would definitely want cumulativity, so that we could define for instance `prod : Type → Type → Type` in a core file, with all `Type`s denoting the same level, and then apply a suitable translation of that to two types at any *two* levels in some other file.

The main annoyance I can see at the moment is that instances of the same definition at different universe levels would have to have different names.  But this isn't all that different from the explicit shift operator of CbES.  We could perhaps even introduce such a term-level shift operator for any *endo*-translation of the level theory --- that restriction would maintain the invariant that all typechecked terms available in one "run" have the same underlying theory, which I think is an important simplification made possible by file-level polymorphism.  (We could do the same for parametricity and mode theories, but in those cases there are not very many interesting endo-translations.)
