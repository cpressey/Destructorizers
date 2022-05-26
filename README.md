Destructorizers
===============

_See also:_ [Lariat](https://github.com/catseye/Lariat#readme)
∘ [LCF-style-ND](https://github.com/cpressey/LCF-style-ND#readme)
∘ [Nested Modal Transducers](https://github.com/cpressey/Nested-Modal-Transducers#readme)

- - - -

The legacy of `car` and `cdr`
-----------------------------

One thing that has long bugged me about Lisp is that `car` and `cdr`
aren't total functions.  Even if you only consider what they do
when given lists.  If you give them a list that isn't a cons
cell, "an error occurs", whatever that means exactly.  (Well, you'll
know it when you see it, I suppose.)

Further, this is something that very many subsequent functional
languages simply copied uncritically.  Scheme fixed the dynamic scope
problem of Lisp, but it didn't fix this — nor did it improve on the
error handling.  And Haskell adopted partial `head` and `tail` functions
into its standard prelude.

In _Theories of Programming Languages_, John Reynolds provides a
formulation that avoids it.  If I recall correctly (which I certainly
might not be — I no longer have a copy of this book), he supplies
`car` with two continuations, one for when it succeeds, and one
for when the value isn't a cons cell.  And same for `cdr`.

This does solve the problem, but it also means writing all list
manipulation in continuation-passing style.

I'd like to discuss a similar, but slightly different, approach,
that I don't recall seeing having been used anywhere, at least
not in a systematic form, intended as a programming language feature.
(If you have seen this somewhere, I'd be quite interested to know
about it.)

List destructorizers
--------------------

Let's suppose that, to deal with data of the list data type, we
are given a function called a _list destructorizer_, which for the
sake of concreteness we'll call `un-list`.

`un-list` takes a list — which _might or might not be a cons cell_ —
and two functions.  The first of these two functions takes two
arguments, while the other function takes no arguments.

*If* the list is a cons cell, the first function is applied.  It
is passed the head of the cons cell as its first argument and
the tail of the cons cell as its second argument.

But if the list is `nil`, the second function is applied.

The important thing to note is that the function `un-list` does the
job of both `car` and `cdr` — and `nil?`, too — but quite unlike
`car` and `cdr`, it can accept any kind of list without causing an
error condition.

You may object that, in a dynamically-typed language like Lisp,
it's still not a total function, because you're not obligated to even
give it a list.  That's true, and I'll talk more about this in a moment,
but there's something else I'd like to talk about before we get there.

Destructorizers for booleans
----------------------------

This sort of _destructorizer function_ can be generalized to other
data types.

What if we write a destructorizer function for booleans?

Well, on the face of it, it seems that `un-boolean` would take
a boolean, and two arguments.  *If* the boolean is `#t`, the
first function is applied.  But if the boolean is `#f`, the
second function is applied.

But — this is exactly how `if` would be defined, if it were to
be defined as a library function instead of built into the
language!

So `if` is the destructorizer function for booleans.

Destructorizers for algebraic data types
----------------------------------------

At this point, you're probably seeing the pattern here.

Given an algebraic data type — that is, a disjoint sum type of
product types — its destructorizer is a function that takes a
value of that type, and one function for every possibility in
the disjoint sum.  Exactly one of those functions is applied,
and when it is applied, the members of the product are passed
as the arguments.

There are some record typing packages for Scheme which come close to
this.  You give it the definition of a record and you get
some functions for creating a value of that function type, and
telling if a given value is an instance of that record type, and
extracting values from the record type.  But every one I've seen
recently creates a set of predicates and a set of extractor functions.
So for example, if you defined a record for lists, it would give you
functions like `mk-cons` and `mk-nil` and `cons?` and `nil?` and
`get-head` (aka `car`) and `get-tail` (aka `cdr`).

But it could just give you `mk-cons` and `mk-nil` and `un-list`.
If you really wanted `car` or `cdr` or `nil?` for some reason, you
could build them out of `un-list`.

And really, it would be better for this to be built around a
type system anyway, instead of presented as a macro package that
any given installation may or may not be using.

Speaking of types, destructors in type theory also come close
to this idea, but again, not quite.  There are two main differences
that I can see.  The first is that, in type theory, destructors are
reduction rules — essentially, syntactic constructs — whereas
destructorizers (in their ideal form) are higher-order functions,
"first-class" values that can be passed to and returned from other
functions.  The other is that destructorizers have nothing to do
with "types" (as type theory perceives them), and can easily be adapted
for use in "untyped" (as type theory regards them) languages such
as Scheme.

Some other programming languages have pattern matching.  They don't
need to use destructorizers, because it's simple enough to destruct
the data type with a pattern, and very expressive too.  In Haskell,
for example,

    data List α = Cons α List | Nil

    length Nil = 0
    length (Cons _ tail) = 1 + length tail

So if you have this, why would you want to use destructorizers
anyway?  Some reasons might be:

*   Pattern-matching is a bit of work to implement.
*   It's even more work to make the implementation detect
    when a pattern is total.
*   It's even more work (of a different sort) to define what
    happens when a pattern is not total.

In a language with pattern-matching, it's easy enough to write
a destructorizer for a given algebraic data type:

    unList :: List α -> (β) -> (α -> List -> β)
    unList Nil c1 c2 = c1
    unList (Cons a b) c1 c2 = c2 a b

In fact this is entirely mechanical, and one could imagine
something like the following in Haskell

    data List α = Cons α List | Nil
        deriving (Ord, Eq, Show, Destructorize)

producing the `unList` function automatically.

I should note that Haskell's libraries do contain some examples
of individually-defined destructorizers, such as
[either](https://hackage.haskell.org/package/base-4.15.0.0/docs/Prelude.html#v:either),
but there doesn't seem to be anything systematic behind them,
not even a convention.

Dynamic versus static typing
----------------------------

One pertinent question for destructorizers is whether the language
is statically typed or dynamically typed.

If it's statically typed, the situation is quite simple — mechanically
generate a destructorizer for any given algebraic data type.

But in a dynamically-typed language, we need to handle the case,
mentioned above, where the destructorizer is passed something that
isn't even an instance of the datatype that the destructorizer
destructorizes.

That's not difficult at all — just have the destructorizer expect
an additional argument, a function which is called with the value
when the value is not of the appropriate type.

This function could itself call another destructorizer, for a different
type, to try that type on for size.  And so forth until the code has
checked for all the data types that it deems appropriate to handle.

In fact the Scheme language advertises that its built-in types form
a disjoint set.  This suggests you could even have a single
"base destructorizer" that handles any of the built-in types of the
language.  (Though of course you probably wouldn't want to make
a recursive definition of things like inexact rational numbers
_just_ for the sake of being able to destruct them into smaller
components; but, all the same, you might.)
