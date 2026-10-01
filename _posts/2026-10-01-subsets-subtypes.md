---
layout: post
title: "On subsets and subtypes"
date: 2026-10-01
---

## The problem of subsets

When we introduce type theory (or "structural/categorical set theory", such as ETCS) to an audience familiar with more traditional membership-based set theory (such as the Zermelo-Fraenkel axioms), we often emphasize something like

> Each element belongs to only one type.  Two distinct types cannot "intersect" or contain each other as "sub-types".

We also argue that this matches the practice of mathematics better.  My new favorite example of this is that in membership-based set theory wo can ask what the intersection is of, say, ℕ and ℕ×ℕ, and not only is this a meaningful question, but its answer depends on our chosen conventions.  For instance, with the von Neumann definition of natural numbers as being the set of all previous ones, we have `2 = {0,1}`; and with the almost-Kuratowski definition of ordered pairs as `(a,b) = {a,{a,b}}` (which works, assuming the axiom of foundation) we have `(0,0) = {0,{0,0}} = {0,{0}} = {0,1} = 2`.  So under these conventions, ℕ and ℕ×ℕ intersect nontrivially, which is bizarre.
 
There are, of course, times in mathematics when we do want to talk about things like set inclusions and intersections, but the standard type-theoretic / structural response is that this only happens when the sets in question are given as subsets of some ambient type.  Such subsets are represented in type theory by their corresponding predicates, for which we can talk about correspondents of inclusion and intersection.

By and large, this is true.  However, there are some places in mathematics where we conventionally do think of one *type* as being a sub-type of another, and some of them are really front and center.  For instance, we generally imagine that we have `ℕ ⊆ ℤ ⊆ ℚ ⊆ ℝ ⊆ ℂ`, and we want to consider ℕ to be a type, not just a "subset of ℂ".  More generally, in such cases, rather than having a fixed ambient set of which we consider subsets, we are "building up sets from below", starting with a smaller set like ℕ and adding things to get to larger ones.  And we even get failures of confluence: for instance, it's natural to also consider ℚ as a subset of the p-adic numbers, but there is no (non-artificial) set containing both the real numbers and the p-adic numbers as subsets that intersect in the rationals.

The standard type-theoretic / structural response to this is to say that these "subset inclusions" are actually *implicit coercions*.  That is, that there's really a function `j : ℕ → ℤ` that is not an "inclusion" (which doesn't mean anything in type theory anyway), and whenever we want an integer and we are given a natural number, we silently apply `j` to what we have.  Some proof assistants, such as Rocq and Lean, allow the user to declare some function as a coercion, and will then insert it silently in this way during elaboration, and leave it out when printing terms.

This works, but I've come to find it a bit unsatisfying.  Most of us learn from our early school days that every natural number *is* an integer, and every integer *is* a rational number, and it's disconcerting to be told that that's not "really" the case.  It's also pedagogically annoying when I want to teach students to use proof assistants.  And concretely, I think it's hard to implement coercions in a truly transparent way; my own experience is that often when doing anything complicated, I need to turn on printing of coercions to figure out what's actually going on.

What if there were a better way?

## Subtypes are a thing

In fact the statement "Each element belongs to only one type" is a lie, at least as regards type theory as a discipline.  Certainly in some type theories it is true, but many type theories do include a notion of *subtyping*, usually written `A ≤ B`, that is "definitional" in the sense that if `x:A` then also `x:B`: the same term, with no coercion needed.

However, because type theory is a programming language, and terms in types are not featureless points but have structure depending on the type, there are restrictions on what subtyping relations make sense.  Specifically, to have `A ≤ B`, we must guarantee that *a term in `A` can be used anywhere a term of `B` is expected*.  In other words, we have to be able to compute the elimination rules of `B` when applied to the introduction rules of `A`.

Generally speaking this means that `A` and `B` must both be types of the same sort and that their defining structures are subtypes in the appropriate direction.  For instance:

- If `A ≤ B` and `C ≤ D`, then we can have `A × C ≤ B × D`.  This makes sense since if `u : A×C` then `u .fst : A` and hence `u .fst : B`, and similarly `u .snd : C` and hence `u .snd : D`, so the two projections of `u` behave like those of an element of `B×D`.
- If `A ≤ B` and `C ≤ D`, then we can have `A ⊔ C ≤ B ⊔ D`.  This makes sense since the canonical elements of `A⊔C` are `left. a` for `a:A` and `right. c` for `c:C`, so if we match on it with branches that bind `left. b` for `b:B` and `right. d` for `d:D`, we can use `a` as `b` and `c` as `d`.
- If `A ≤ B` and `C ≤ D`, then we can have `B → C ≤ A → D` (note the contravariance in the domain).  This makes sense since if `f : B → C` and `a:A`, then also `a:B`, so `f a : C` and thus `f a : D`, so `f` can be applied like a function `A→D`.

This sort of covariant/contravariant subtyping is already useful, but it can also be extended significantly.  In Narya, at least, datatype constructor names and record/codata field names are not namespaced or associated to any particular type, but belong to a global flat name domain.  This meshes well with bidirectional typechecking: a constructor application `left. a` checks at any type that has a constructor named `left` whose type `a` checks against, while `u .fst` synthesizes as long as `u` synthesizes any type that has a field named `fst`.  It also enables a more general kind of subtyping:

- We can have `A ≤ B` if `A` and `B` are both record types, and for every field of `B` there is a field of `A` *with the same name* and whose type is a subtype of the type of the field of `B`.  For in this case, if `u:A` and `fld` is a field of `B`, the application `u .fld` has the type of the field `fld` of `A`, and hence also the type of the field `fld` of `B`.  It doesn't matter if `A` has more fields than `B`; those just can't get accessed once we regard `a:A` as belonging to `B` instead.

- Dually, we can have `A ≤ B` if `A` and `B` are both datatypes and for every constructor of `A` there is a constructor of `B` *with the same name* and the same number of arguments, and such that the argument types of each constructor of `A` are subtypes of those of the corresponding constructor of `B`.  For in this case, if we match on an element of `A` with branches labeled by the constructors of `B`, any canonical element of `A` will determine some branch with its arguments usable as the pattern variables.  It doesn't matter if `B` has more constructors than `A`; those branches in a match against `B` just won't get used if the discriminee happens to be an element of `A`.

The first of these, for record types, is probably more familiar: it's the sort of subtyping we find in object-oriented programming languages.  A "derived class" just adds more methods to a "base class", so a derived object can be used wherever you need a base object.  Note, though, that the corresponding "coercion function" in this case is not an injection, so this sort of "subtype" is not semantically a "subset".

The second subtyping condition, for datatypes, is an obvious "dual" of the one for records.  It doesn't seem to be as commonly known or implemented (with the exception of OCaml's "polymorphic variants").  But I think it's the key to a better representation of informal subset relationships.

## Subtyping of number systems

Let's start with the usual definition of `ℕ`:
```
def ℕ : Type ≔ data [ zero. | suc. (_:ℕ) ]
```
Here's one way to define `ℤ` so that `ℕ ≤ ℤ`:
```
def ℤ : Type ≔ data [ zero. | suc. (_:ℕ) | negsuc. (_:ℕ) ]
```
By using the same names for the constructors `zero` and `suc`, we ensure the consistency condition is satisfied.  Note that the constructor `suc` of `ℕ` is recursive, while that of `ℤ` is not, but that doesn't matter to the subtyping condition.

Defining `ℚ` so that `ℤ ≤ ℚ` is only slightly trickier: we need to add a fourth constructor that constructs only non-integer fractions and constructs them each exactly once.
```
def ℚ : Type ≔ data [
| zero.
| suc. (_:ℕ)
| negsuc. (_:ℕ) 
| frac. (num : ℤ) (den_minus_two : ℕ) (_ : rel_prime num (den_minus_two + 2)) ]
```
That is, a non-integer fraction has an integer numerator and a natural number denominator that is at least 2, which are relatively prime to each other.

Maybe this is a good place to pause and observe that these subset relations `ℕ ⊆ ℤ ⊆ ℚ` *also* require work to force to hold in formal membership-based set theory.  Simple definitions of ℤ and ℚ in ZFC also do not produce supersets of ℕ.  Nowadays I think most people who found mathematics on ZFC also paper this over with something like implicit coercions, but Bourbaki took it seriously: after defining each new number system, they carefully redefined it by cutting out the naturally-occurring copy of the previous system and pasting in the original version.  Thus, for instance, they might define `ℚ′` as a quotient of `ℤ × ℕ`, but then set `ℚ` to be the union of `ℤ` with the complement of the image of the inclusion `ℤ ↪ ℚ′`.  This is very similar to our datatype `ℚ` above.

However, defining `ℝ` is harder for us: no matter what definition of real numbers we take, there is no "inductive" way to ensure that it excludes the rational numbers.  We could, of course, add an argument to the new constructor asserting that, for instance, the Dedekind cut *isn't* determined by a rational number, corresponding to what Bourbaki did with sets.  But this wouldn't give us the correct answer *constructively*: it would only be the type of real numbers that are either rational or irrational, and asserting that every real number is either rational or irrational is a nonprovable case of excluded middle.

The solution is to extend the subtyping condition to *higher* inductive types, so in particular we can have `A ≤ B` if `B` adds additional path-constructors.  The same rationale applies: any match against `B` applied to a discriminee from `A` can ignore its clauses for constructors not appearing in `A`, even if those are path-constructors.  In this case the coercion function `A → B` may not be injective, but as we saw with records, that isn't a necessary condition for a "subtype".

Now we can put in *all* the Dedekind cuts or Cauchy sequences or whatever, and then add path-constructors collapsing the rational cuts back to the rational numbers.
```
def ℝ : Type ≔ data [
| zero.
| suc. (_:ℕ)
| negsuc. (_:ℕ) 
| frac. (num : ℤ) (den_minus_two : ℕ) (_ : rel_prime num (den_minus_two + 2))
| cut. (L : ℚ → Prop) (R : ℚ → Prop) (_ : is_cut L R)
| zero_cut. : Id ℝ zero. (cut. (x ↦ x < 0) (y ↦ 0 < y) (…))
| suc_cut. : …
| negsuc_cut. : …
| frac_cut. : … ]
```
(This is not valid Narya syntax yet, since higher inductive types aren't implemented yet.  But they will be, trust me.)

Homotopy theorists will recognize this as the *mapping cylinder* of the original non-inclusion `ℚ → ℝ`.  In homotopy theory this construction is used to convert an arbitrary map into a cofibration.  And we can view it as doing the same thing in type theory, if we define a "cofibration" as a map having the left lifting property against "trivial fibrations", the latter now meaning dependent projections with contractible fibers.  (I learned this many years ago from [Peter LeFanu Lumsdaine](https://www.semanticscholar.org/paper/MODEL-STRUCTURES-FROM-HIGHER-INDUCTIVE-TYPES-Lefanu-Warren/5e480fefd587a84a5887a30dea6613d70f0339a9).)

Of course, it would be a bit annoying if we had to write nine branches every time we define some operation on ℝ, but we don't always need to do that.  For instance, we can prove that `ℝ` defined above is *equivalent* to the type `ℝᵈ` of Dedekind cuts, hence by univalence *equal* to it, and so we can transport anything defined for `ℝᵈ` to something defined for `ℝ`.  In fact more is true: the inclusion `cut. : ℝᵈ → ℝ` is a "trivial cofibration" (as for any mapping cylinder), so anything defined for `ℝᵈ` can be extended strictly to `ℝ` preserving its original behavior definitionally on explicit cuts.

On the other hand, in some cases we might *want* to write out the nine cases to get better definitional behavior.  For instance, if we define addition on `ℝ` by transporting it from Dedekind cuts, then `(1:ℝ)+(1:ℝ)` (which elaborates to `ℝ.plus (suc. zero.) (suc. zero.)`) will compute to a `cut.` defining two.  But if we define addition on `ℝ` with separate cases for `zero.` and `suc.`, we can ensure that `(1:ℝ)+(1:ℝ)` computes to `2:ℝ` (that is, `suc. (suc. zero.)`).  The constructors `zero_cut` and `suc_cut` force us to verify that this is in fact equal to the `cut` defining two, but it certainly looks nicer for the user.

(A full definition of addition on `ℝ` in this style would of course require 81 cases, which is admittedly a lot.  There are some enhancements that could be made to reduce this.  For instance, it might be more ergonomic to replace the four `_cut` constructors with a single recursive one saying that any two real numbers, however-defined, that induce the same cut are equal.  And some syntactic support could help too, for instance the ability for a match on a supertype to be defined by "extending" a known match on a subtype, so that the repeated branch cases don't have to be copy-and-pasted.  Of course nowadays an AI won't blink at 81 cases anyway.)

We could also extend this to include more real numbers that have common names, for instance adding a `pi.` constructor or a `sqrt.` constructor, so that operations that yield such numbers also wouldn't get collapsed to cuts.  And this isn't just for pretty printing, either: in other cases some set might have a subset on which we have more efficient algorithms, or even on which an otherwise noncomputable operation becomes computable.  So it could be very useful to be able to define such an operation efficiently with a separate match case on the subset, and then just have to verify on the path-constructor that the result is propositionally equal to the abstract value (i.e. prove the efficient algorithm correct).

## The Von Neumann universe of constructor forms

With constructor-based subtyping at the front of our minds, it becomes possible to envision a different picture of the universe of types.  Rather than a bunch of disconnected type-bubbles, each containing their own terms with never any containment or intersection, we can picture an undifferentiated collection of all the "canonical nested constructor forms" such as
```
zero.
suc. (suc. zero.)
suc. (left. (cons. (right. (suc. zero.)) nil.))
left. (x ↦ pair. (right. (suc. x)) nil.)
```
Every datatype is then a sub-collection of this, containing those constructor forms specified by its definition.  Thus, we can talk about when one datatype is a subtype of another in terms of containment of such collections.

Semantically, a constructor form is a certain kind of well-founded tree with labels.  This has something of the flavor of a ZFC-set (a set of sets of sets of ...), which can also be represented as a well-founded tree.  Intuitively, a constructor form is built up like a ZFC-set except that at each stage of set-formation we give the new set a label and we explicitly order or parametrize its elements.  So the "world of datatypes" has at least an ontological flavor similar to the "von Neumann universe" of ZFC sets.
