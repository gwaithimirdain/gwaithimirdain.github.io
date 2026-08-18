---
layout: post
title: "Modal type theory + AI redux"
date: 2026-07-19
---

Modal type theory has landed in Narya!  I'm super excited about this: I've been looking forward to having a true multimodal proof assistant for over a decade, ever since I started writing about [cohesive type theory](https://arxiv.org/abs/1509.07584).  To my knowledge (please correct me if I'm wrong), Narya is the first full dependently-typed proof assistant to implement general multimodal type theory, including mode theories that are not locally posetal (have more than one 2-cell between some parallel modalities).  I plan to say more in another post about the difficulties involved in dealing with the non-locally-posetal case, and why Narya happened to already be ideally positioned to solve them.

I'm not going to write much about how to *use* modal type theory here, because I just did a whole lot of that in the Narya documentation.  You should go read it, and try it out!

- [Modal type theory](https://narya.readthedocs.io/en/latest/modal.html)
- [Modal parametric discreteness](https://narya.readthedocs.io/en/latest/discrete.html)
- [External parametricity](https://narya.readthedocs.io/en/latest/external.html)

But I do want to comment on a few choices related to previous blog posts, and on my use of AI agents in coding it.

- I went with the syntax `(x :□| A) → B` for modal function-types.  Only *generating* modalities have names, so if you want to annotate a variable by a composite of modalities you would in general have to separate their names by a space as in `(x :△ □| A)`.  However, it seems that often we give modalities single-character names, so I implemented a simplification that if all the modalities have single-character names, you can run them together as in `(x :△□| A)` and Narya will split them up for you.

- The syntax I settled on for keys is `M #key`.  I like this because being postfix, it looks like a "property" or a "version" of `M`, which it is, and the symbol `#` it uses is on a standard US keyboard.  (I like all syntax to at least have an equivalent version that can be typed on a standard US keyboard without fancy input methods; unfortunately I broke that rule with the names of the modalities for built-in mode theories, although at least you can rename them with a command-line flag.)

- For window modalities and modal field projections, I stuck with the same kind of modal annotations rather than introduce anything new: `match (x :□| _) [ ... ]` and `(x :□| _) .fld`.  Although it may be a bit uglier that some other possibilities, I think it reduces the cognitive load on the user (which is already heavy in a modal type theory) to use the same syntax every time a term is being checked in a locked context.

- The above choice means that to define a record type with a modal field, you *must* use the [self variable]({{ site.baseurl }}{% post_url 2026-06-17-self-variables %}) syntax for record types!

- Claude's function to strip keys off an environment did not work.  I tried writing the same function in a different way, but that didn't work either.  Eventually I settled on another way that I'm pretty confident works.  However, it seems hard to justify semantically, as it involves using "imaginary faces" of cubes that temporarily exist but eventually get canceled by a degeneracy.  (This doesn't happen in plain modal type theory, which is semantically fine, only in the more experimental version with "discrete modalities".)

In general, over the past month Claude has gotten better at doing what I want, or at least I've gotten better at getting it to do what I want.  For instance, it's been a long time since it tried to use `Obj.magic` or claim that a possible case was impossible.  I think it's partly because it's recording some of the things I tell it in its internal memory about the project.  (And also the models are probably continuing to improve; I had a brief promotional access to Fable for a couple of weeks.)  It's very good, for instance, at generating the boilerplate code for implementing new mode theories, given a description in words of the desired theory, which is very helpful given that all mode theories have to be implemented in OCaml so far.  And I've learned, as everyone else also says, that when you want it to do anything substantial, it makes a big difference if you explicitly ask it to "make a plan" first and then give it feedback on its plan before it starts implementing.

However, I still can't rely on it to come up with algorithms, or even to fill in the details of algorithms.  If I give it a clear and precise description of the algorithm I want, it's usually impressively good at implementing it.  But if my description is vague or missing pieces, then sometimes it will spin for a long time trying to figure out how to fill it in, or come back to me and complain that it's stuck -- or, which is maybe worse, it will make guesses about how to fill things in, which compile and look plausible, but often turn out to be subtly wrong.  I've had to fix a fair number of Claude-introduced bugs.

To be fair, though, Claude is also quite helpful at finding bugs, at least after I've discovered a test case.  Sometimes it can even fix them, although as often as not I can think of a better fix myself.  But *tracing through the code* to find exactly *where and why* an error is getting emitted is one of my least favorite parts of programming Narya, and Claude is very good at doing it for me.

Claude is also good at finding ways to speed up the code, which I've never been very good at.  If you haven't built Narya for a while, you may notice that the test suite runs substantially faster.

On balance, Claude has definitely saved me a lot of time: Modal Narya would definitely not be ready for use today without it.  And I believe that, *after* all the back and forth, the code quality is comparable to what I would have produced myself.  (Of course, I also make mistakes all on my own!)

I do have to discipline myself to look over all the code that it writes.  It's tempting to just trust it, especially when it also writes tests that pass; but I've been burned by doing that too many times.  I don't try to read and understand exactly *everything* the code is doing line by line, particularly the parts that are doing things like type-level arithmetic to make the parameters work out correctly: in those cases I can be fairly confident that anything that compiles is correct.  But I do try to understand at a high level what the code is doing, and make sure that I agree with it.  Most of the time there are some changes I want to make, sometimes minor, sometimes more substantial.  And it matters to me that I "own" the Narya codebase, in the sense that I have a good idea of all the moving parts and how they fit together, even if I didn't write it all myself.

I am also fairly impressed at how well Claude can write *Narya* code for new blackbox tests: it can't have a very big library of examples to be drawing from because there just *isn't* very much Narya code in the world yet.  Apparently Claude has somehow "learned how to program" in a way that is, if not completely language-independent, adaptable to new languages without much difficulty.  I'm curious how much of Claude's knowledge about Narya syntax comes from reading the existing Narya code in the other blackbox tests, how much comes from reading the Narya documentation (which is, of course, right there in the same repository), and how much comes from reading the Narya source code -- but I suppose I'll never know.

One thing I don't let Claude do, by the way, is write the documentation.  I'm sure it could do a passable job, and sometimes it offers to; but to me the documentation is a creative product, like a novel or a mathematical paper, and I want it to be written by a human who thought at least a little about every word.  Maybe that's old-fashioned and maybe I'll change my mind in the future, but at the moment that's how I feel.

Anyway, enough about AI.  We had a little demo/tutorial here at [CT26](https://ct2026.com/) and folks were excited about the modal type theory features.  I'm looking forward to seeing what people can do with them.  Please give them a try, and send along any feedback!
