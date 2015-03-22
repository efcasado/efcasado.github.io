---
layout: post
title:  "A Gentle Introduction to QuviQ's QuickCheck"
date:   2014-11-27 23:56:45
description: A gentle introduction to QuviQ's QuickCheck.
categories:
- blog
permalink: quviq-quickcheck-intro
---

## What is QuickCheck?

`QuickCheck` is a language for stating properties of programs.

{% highlight erlang %}
?FORALL(X, nat(), X*X >= 0)
{% endhighlight %}

It is implemented as a library of functions and macros (like the `?FORALL`
macro in the example above).

`QuickCheck` is also a **tool** that allows you to test those properties
using **randomly generated cases**.

`QuickCheck` comes from the academia and it was originally written in Haskell.
`QuickCheck` was first announced to the world in 2000, when the paper
[QuickCheck: a lightweight tool for random testing of Haskell programs](http://users.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf)
, authored by Koen Claessen and John Hughes from Chalmers University
(Gothenburg, Sweden), was awarded as the most influential paper at the
International Conference on Functional Programming (ICFP).

## Getting Erlang QuickCheck

Erlang QuickCheck is implemented by [QuviQ](http://www.quviq.com/) and it is
available in two flavours: a free version called QuickCheck mini, and a premium
feature-complete version. [QuviQ's downloads](http://www.quviq.com/downloads/)
page contains some information about the differences between the two.

Installing `eqc-mini` on your system should be as easy as downloading a zip
file from QuviQ's downloads page, unzipping it, and adding it to your
`ERL_LIBS` environment variable (e.g.,
`export ERL_LIBS="$ERL_LIBS:$HOME/Development/eqc/eqc-1.0.1"`). Once
you have done this, you can test the installation by opening and Erlang shell
(i.e., `erl`) and typing `eqc_gen:sample(eqc_gen:int())`. It should produce a
sequence of random integers.

The rest of this article assumes you are using QuickCheck's free version
(hereinafter referred to as `eqc-mini`), but it
should be equally applicable to QuickCheck's commercial version since the free
version is nothing but a subset of the commercial one.

## Using Erlang QuickCheck

Now that you have got `eqc-mini` installed on your system, all what is left in
order for you to be able to write your first QuickCheck tests is including
Erlang QuickCheck's header file into your test module.

To do so, open you favourite text editor and add the following line to your
test module.

{% highlight erlang %}
-include_lib("eqc/include/eqc.hrl").
{% endhighlight %}

Below is a sample module featuring a simple `eqc` test.

{% highlight erlang %}
-module(hello_eqc).
-include_lib("eqc/include/eqc.hrl").
-compile([export_all]).

even_nat() ->
    ?SUCHTHAT(N, nat(),
              N rem 2 == 0).

prop_nat() ->
    ?FORALL(N, even_nat(),
            N * N >= 0).
{% endhighlight %}

`even_nat/0` is a generator and produces random even natural numbers,
whereas  `prop_nat/0` is a property met by all natural numbers.

You can sample a generator using the `eqc_gen:sample/1` function. That is,
`eqc_gen:sample(hello_eq:even_nat())` should give you a sequence of random
even natural numbers.

When it comes to properties, you can test them using the `eqc:quickcheck/1`
function. In the example above, you can test the `prop_nat/0` property by
running `eqc:quickcheck(hello_eq:prop_nat())`, which should output something
like

{% highlight erlang %}
Starting eqc mini version 1.0.1 (compiled at { {2010,6,13},{11,15,30} })
....................................................................................................
OK, passed 100 tests
{% endhighlight %}


### Generators

Generators allow you to produce sample data of a certain type.

#### Built-in generators

`eqc` ships with a bunch of ready-to-use generators.

Below is a list of `eqc`'s primitive generators:

- binary()
- bitstring()
- bool()
- char()
- int()
- largeint()
- nat()
- real()

`eqc` also lets you generate lists consisting in elements drawn from any of the
generators listed above (e.g., `list(char())`). Note that lists can (and most likely
will) be of variable length. If you want to generate a fixed-size list of values, you
must use the `vector/2` generator instead (e.g. `vector(4, int())`). Another
interesting generator is `orderedlist/1`, which allows you to generate (guess what?) lists where the elements are sorted in ascending order.

`eqc` ships also with some special generators, namely `choose/2`, `elements/1`,
`frequency/1` and `oneof/1`.

- `choose/2` allows you to generate random values within a range (e.g., `choose(7, 13)`).

- `elements/1` takes a list of Erlang terms as input and will output a random term
from that list every time it is called (e.g., `elements(['a', 1, pid(0,0,0)])`).

- `oneof/1` may look similar to `elements/1` but it has different
semantics. It takes a list of generators as input instead of a list of Erlang terms. Every time `oneof/1` is called, it picks a random generator from the provided list and draws a value from it.

- `frequency/1` is similar to `oneof/1`, but whereas `oneof/1`'s probability of
drawing a value from any of the specified generators is uniformly distributed,
`frequency/1` allows you to specify different weights for each generator (e.g.,
`frequency([{1, bool()}, {9, char()}])`).

Last but not least, `eqc` comes with the handy `?SIZED` macro, which can be
used to generate elements of a certain type featuring different sizes (e.g.,
`?SIZED(Size, vector(Size, int()))`).

#### Custom generators

You are not limited to use QuickCheck built-in generators. QuickCheck features
a set of functions and macros that you can combine to create your very own
custom generators.

The `?LET` macro lets you bind values from a generator to a variable. It also lets
you refine this values by applying functions and or filters onto them.

{% highlight erlang %}
?LET(L, list(int()), lists:usort(L))
{% endhighlight %}

The above example generates random list of integers without duplicates. First,
random lists of integers (which may contain duplicates) are generated. Each one
of these lists are bound to the variable `L`, one at a time. Duplicates are then
deleted by applying the `lists:usort/1` function to the list bound to the variable `L`.

The `?SUCHTHAT` macro allows you to refine a bit more your custom
generators. It allows you to specify a predicate that must hold true for the value
drawn from the specified generator. Otherwise, that value is dropped.

{% highlight erlang %}
?SUCHTHAT(N, int(), N rem 2 == 0)
{% endhighlight %}

The expression above illustrates how the `?SUCHTHAT` macro can be used to
generate random even numbers.

### Properties

QuickCheck properties are always defined by means of the `?FORALL` macro.
This macro takes a variable name, a generator and a predicate and makes sure
that the provided predicate holds true for the random sample drawn from the
specified generator.

{% highlight erlang %}
?FORALL(N, nat(), N*N >= 0)
{% endhighlight %}

The example above generates a random sequence of natural number and checks
whether the `N*N >= 0` property holds true for all of them.

The `?IMPLIES` macro can be used within a `?FORALL` macro in order to filter
out some of the values drawn from the generator provided in the `?FORALL`
macro.

{% highlight erlang %}
?FORALL(N, nat(),
?IMPLIES(N /= 0, N*N > N))
{% endhighlight %}

The above example illustrates how the `?IMPLIES` macro can be used to filter
out all zeros from the random sample.

Note that `?FORALL` macros can be nested. This is useful when one has
generators that depend on other generators.


{% highlight erlang %}
?FORALL(Idxs, bitmap_eqc:idxs(),
?FORALL(Bitmap, bitmap_eqc:bitmap(Idxs),
        lists:all(fun(0) -> false;
                     (1) -> true
                  end,
                  bitmap:idxs(Idxs, bitmap:new(Idxs, Size)))))
{% endhighlight %}
