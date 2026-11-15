---
title: "Better Exhaustiveness"
author: "Filippo Nardon, Giovanni Maria Zanchetta"
date: \today

geometry: margin=0.75in
toc: true
bibliography: references.bib
link-citations: true
colorlinks: true
---

# Motivation

Many ML-style languages base their branching capabilities around a pattern
matching construct. This makes it possible for such languages to check at
compile time the exhaustiveness of the branches with respect to all the possible
shapes of the values they match against. The
[Gleam language](https://gleam.run/) in particular uses an `exhaustiveness`
module to check that, based on the algorithm described by
@jacobs2021patternmatching.

There are many cases in which some branches of computation may be ruled out.
Take the following example:

```gleam
let a = [0, 1, 2]
case a {
  [] -> panic as "Unexpected"
  [x, .._] -> io.println(int.to_string(x))
}
```

The `[]` branch can never be taken, yet it is required: without it the checker
rejects the program as non-exhaustive. We would like to write just:

```gleam
let a = [0, 1, 2]
case a {
  [x, .._] -> io.println(int.to_string(x))
}
```

This of course is a trivial example, however, we can think of cases in which the
result of the analysis avoids the necessity of specifying unreachable branches
at compile time.

The final goal is to extend the Gleam compiler so that such programs compile
without the need to specify a pattern matching branch whenever the analysis
rules that branch out.

Gleam is a deliberately simple language, with only 22 reserved keywords. This
makes it a good candidate for implementing the analysis, and working on the
actual compiler will allow us to test it on real projects.

# From constant propagation to constructor analysis

Some of this precision is already built into the compiler, in particular the
following snippet of code compiles:

```gleam
type ABC { A B C }

fn works() {
  let a = A
  case a { A -> 0 }
}
```

Here the compiler does some basic form of constant propagation, however, less
trivial examples quickly require the developer to introduce useless
branches. In particular, the following snippet of code does not compile:

```gleam
type ABC { A B C }

fn fails() {
  let a = case foo() {
    0 -> A
    _ -> B
  }
  case a {
    A -> 0
    B -> 1
  }
}
```

We propose going from constant propagation to constructor analysis where we
check the possible constructors that a variable may hold. Our analysis will be
conservative in the sense that the possible set of abstract constructors that a
variable may hold is a superset of the set of concrete constructors that the
variable will hold. Similar analyses have been used to verify the safety of
incomplete pattern matches in Haskell [@mitchell2008notallpatterns;
@jones2021intensional], and GHC's coverage checker propagates what is known
about a value from an outer match to the inner ones
[@graf2020loweryourguards].

# From constructor analysis to interval analysis

The language also provides primitive types such as `Int`. Those may be embedded
into an algebraic data type thus allowing for pattern matching of the following
form:

```gleam
type ABC { A(Int) B C }

fn works() {
  let a = A(5)
  case a {
    A(n) if 0 <= n && n <= 10 -> 0
    _ -> panic as "Unexpected"
  }
}
```

Our objective here is to perform interval analysis on the contents of the
constructors in order to rule out as many useless branches as possible.
That would allow the following code, which is rejected today, to compile:

```gleam
type ABC { A(Int) B C }

fn fails() {
  let n = 5
  let a = case foo() {
    0 -> A(n-1)
    _ -> A(n+1)
  }
  case a {
    A(n) if 4 <= n && n <= 6 -> 0
  }
}
```

# Considerations

We expect to take advantage of the following techniques:

- Monotone frameworks (for defining the constructor analysis)
- Interval analysis
- Path sensitivity (for handling possible paths depending on the local
  variables)
- Interprocedural analysis (for basing our analysis on the result of functions,
  since the Gleam language favors recursion compared to iteration)

# Plan

We expect to spend around 60 to 80 hours in total on the project. The schedule
below is a guideline rather than a fixed plan: after each step we want a
working and tested analysis, and we will decide what comes next based on how
far we got.

| Period | Milestone |
|---|---|
| Nov 16–22 | Getting to know the Gleam compiler and its `exhaustiveness` module. Intraprocedural constructor analysis on `let` bindings and `case` expressions, connected to the exhaustiveness check so that ruled-out branches can be omitted, with tests. |
| Nov 23–29 | Interprocedural analysis for function results, including recursive functions. Path sensitivity, and interval analysis for `Int` fields and guards. |
| Nov 30–Dec 4 | Evaluation on real Gleam projects and fixes; presentation on December 3 and report on December 4. |

The first week is mostly about finding our way around the compiler, so we keep
the analysis itself narrow there and leave function calls and intervals for the
second week.

# References
