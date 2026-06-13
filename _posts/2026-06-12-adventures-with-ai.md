---
layout: post
title: "Adventures with Agentic Coding"
date: 2026-06-12
---

I've been slowly experimenting with agentic coding, using Gemini CLI (now "Antigravity") and Claude Code.  It's interesting to see in particular what they're good at and what they're not so good at.

They're impressively good at GADT hacking, e.g. doing arithmetic on type-level nats and other stuff to ensure that types match up.  And the presence of the strong type information means that I'm more confident in the results they produce: in many cases as long as the function has the right type, and the OCaml compiler accepts it, and a cursory examination indicates that it has the correct recursive structure, I feel like I can be pretty sure it's right without following all the type-level arithmetic myself.

However, they do like to try to circumvent the type system.  On one project Claude made lots of calls to `Obj.magic`, OCaml's unsafe coercion function with type `'a -> 'b` that doesn't change the runtime representation but just tells the compiler "trust me".  It claimed they were all fine because the runtime representations agree; it was obviously wrong.  It also liked to close off difficult cases with exceptions, claiming they were unreachable, when they obviously were not.  And it was very lazy: it kept telling me that what I wanted was too big of a project to accomplish "in one session" until I shouted at it that yes, it's a big project, and I want you to GET STARTED ON IT.  I expect a more skilled AI-user would be able to guide it past these issues more smoothly, but I'm managing.

I mentioned in a previous post that it's great for generating boilerplate, like functors between type-level free categories.  It's also great for implementing little features that I've wanted to do for a while but never had the time, like [this pull request](https://github.com/gwaithimirdain/narya/pull/108) that allows you to specify different "default variable names" for particular canonical types.  That one required understanding Narya's parser and postprocessor as well as the readback and printing functions; I was impressed at how well Claude managed that.

For the moment, I'm keeping a tighter rein on parts of the code that involve serious mathematical understanding of the typechecking and normalization algorithm.  But I did ask Claude to help me past a sticky point in the implementation of normalization for modal type theory with nonparametric modalities, reimplementing a recursive function (that I originally wrote) to strip the keps off of an environment to handle intermediate operations in several different ways.  I haven't yet successfully completed the implementation, but Claude's contribution looks promising, although it does still have some issues that are work in progress.

In other news, Ambrus and Chaitanya are visiting this week and we've made some real progress on proving that the universe is fibrant in HOTT.  One of these days I'll finally post about how that relates to modal type theory.
