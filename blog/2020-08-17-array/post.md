# Benchmarking the new `smallArrayOf#` primop

As part of my Summer of Haskell 2020 project I [implemented](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3571) a new GHC primop `smallArrayOf#` which creates a `SmallArray#` from any number of elements passed via an arbitrarily nested unboxed tuple. `Array#` and `SmallArray#` are the substrate for other packages' data structures, e.g. [unordered-containers](https://hackage.haskell.org/package/unordered-containers)' `HashMap` \[[0](https://hackage.haskell.org/package/unordered-containers-0.2.12.0/docs/src/Data.HashMap.Internal.html#HashMap), [1](https://hackage.haskell.org/package/unordered-containers-0.2.12.0/docs/src/Data.HashMap.Internal.Array.html#Array), [2](https://hackage.haskell.org/package/unordered-containers-0.2.12.0/docs/src/Data.HashMap.Internal.Array.html#Array%23)\].

## Results

I benchmarked how long it takes to initialize a known-length `SmallArray#` with the new primop (blue). We compare this to the best (to our knowledge) previously available method which is fully unrolled array initialization via Template Haskell (green) and dynamic initialization in a (non-unrolled) loop (yellow). (Benchmarked on 16-core i9 MacBook Pro under macOS.)

![A graph comparing the execution time for 3 array initialization methods, showing 3 fairly linear curves explained below. The graph is generated from the first two columns of output.csv.](arrayOf-benchmark.png)

**We achieve a whopping ~2.5x for array initialization versus fully unrolled code and ~5x versus dynamic initialization!**

The benchmark code plus results live at [github.com/buggymcbugfix/arrayOf-benchmark](https://github.com/buggymcbugfix/arrayOf-benchmark).

## Background

This is how you use the new primop:

~~~hs
let
  a1 :: SmallArray# Bool = smallArrayOf# (# True, False #)
  a2 :: SmallArray# Bool = smallArrayOf# (# (# #), (# True, (# False #) #) #)
in
  ...
~~~

> **Side note:** if we leave out the type signatures, GHC is prone to reject these bindings with the error _You can't mix polymorphic and unlifted bindings_.

Here `a1` and `a2` compile to equal arrays because GHC flattens unboxed tuple arguments to curried function arguments during the [unarise](https://gitlab.haskell.org/ghc/ghc/blob/master/compiler/GHC/Stg/Unarise.hs) STG->STG pass since the code generation backends have no notion of unboxed tuples (nor sums). Hence both `a1` and `a2` eventually compile to essentially `smallArrayOf# True False`.

In other words, `smallArrayOf#` is a **variadic primop**—the first of its kind in GHC!

Its type signature is currently:

~~~hs
smallArrayOf# :: forall r (a :: TYPE r) b. a -> SmallArray# b
~~~

In other words it is pretty much untyped and highly unsafe: it would for example allow filling an array with `Bool`s and typing it at `SmallArray# Int`. Yikes!

It would perhaps be nice to restrict the type to something like this:

~~~hs
smallArrayOf# :: forall rs (as :: TYPE ('TupleRep rs)) a. as -> SmallArray# a
~~~

This rejects anything but unboxed tuples as an argument to `smallArrayOf#`, which means for example that the type checker will ~~yell at~~ _gently remind_ you if you pass arguments in the wrong order. We lose the ability to write the singleton tuple without the unboxed array syntax—that's hardly a loss.

Sadly it is not that straightforward to give this more restricted type signature as the primop definitions get generated via some quite arcane code involving (but not limited to!) the magic file `primops.txt.pp`. To give you an example, when I wanted to make my primop runtime-rep polymorphic, a.k.a. "[levity polymorphic](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/levity-pldi17.pdf)" \[PDF\]—this is the `forall r (a :: TYPE r)` bit in the type signature—I eventually figured out that I need to use `o` in the primop's type signature in `primops.txt.pp`. (Aside: `o` for "open" [the old name for runtime-rep polymorphic {a.k.a. levity polymorphic}] type variables). See for example [`cloneArray#`'s signature here](https://gitlab.haskell.org/ghc/ghc/-/blob/master/compiler/GHC/Builtin/primops.txt.pp).

So I wrote `o -> SmallArray# a`, thinking that I would get something like `forall r (o :: TYPE r) a. o -> SmallArray# a`. Alas, no! Instead this generated `forall r (a :: TYPE r). a -> SmallArray# a`, which is not only wrong—it also took me DAYS to figure out! It would be awesome to see this part of the codebase modernized somewhat.

## Time for activating our type-level arsenal?

We could definitely do some type class or type family magic to enforce a safe interface. However, many (most?) primops are unsafe anyway and it seems to be better to relegate the job of exposing a safe interface to a library, such as [primitive](https://hackage.haskell.org/package/primitive), for several reasons:

1. I don't know what the best possible mechanism is and I think it is healthy to let library writers try out different ways.
2. There is no precedent for class constraints or type families in [GHC.Exts](https://hackage.haskell.org/package/base-4.14.0.0/docs/GHC-Exts.html
) ("GHC-specific extensions").
3. More work for me.

### A Typed Template Haskell Interface for Arrays of Fully Evaluated Static Data

[MkSmallArray.hs](https://github.com/buggymcbugfix/arrayOf-benchmark/blob/master/MkSmallArray.hs) provides an example of a possible Typed Template Haskell interface for creating `SmallArray#`s from lists and it in fact guarantees typesafe usage, however at the cost of having a `Lift` constraint on the element type (and I think it's not possible to have thunks, e.g. for infinite structures, as the elements):

~~~hs
smallArrayOf :: Lift a => [a] -> Code Q (SmallArray# a)
~~~

With this we can easily write code for generating static lookup tables for computations that we don't want to have to do at runtime.

~~~hs
let a :: SmallArray# Int = $$(smallArrayOf [expensive i | i <- [0..3]]) in ...
~~~

> **Side note:** again, the type signature is usually needed, or alternatively an explicit type application to `smallArrayOf`.

This generates the following code (where `w`..`z` have been bound to the result of `expensive 0`..`expensive 3`):

~~~hs
let a :: SmallArray# Int = smallArrayOf# (# w, x, y, z #)
~~~

Without our new primop, the best we can do is to precompute our data into a list and generate code for an unrolled array initialization. This corresponds to the green line on the benchmark.

[MkSmallArrayOld.hs](https://github.com/buggymcbugfix/arrayOf-benchmark/blob/master/MkSmallArrayOld.hs) does exactly this (don't look at the code—I ended up writing the `Exp` AST manually as I ran into issues with quoting and unquoting). For example we generate something along the lines of the following code with Template Haskell:

~~~hs
let a :: SmallArray# Int
  = unbox (runST (ST (\ s -> case newSmallArray# 4# undefined s of
    (# s0, marr #) ->
      let s1 = writeSmallArray# marr 0# w s0 in
      let s2 = writeSmallArray# marr 1# x s1 in
      let s3 = writeSmallArray# marr 2# y s2 in
      let s4 = writeSmallArray# marr 3# z s3 in
      let (# s5, arr #) = unsafeFreezeSmallArray# marr s4
      in (# s5, MkBox arr #))))
~~~

Basically what this code does is to make a new `MutableSmallArray#`, setting all the fields to `undefined` and then writing all the fields and finally freezing the array to an immutable one. (GHC can't just give us uninitialized "junk" memory because the GC would go haywire.)

So we are doing at least twice as much work as we should—great that the benchmark results reflect this intuition!

Remember that you can check out the benchmarks here: [github.com/buggymcbugfix/arrayOf-benchmark](https://github.com/buggymcbugfix/arrayOf-benchmark).

Cross your fingers that the `smallArrayOf#` primop will get merged into GHC: [https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3571](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3571)!
