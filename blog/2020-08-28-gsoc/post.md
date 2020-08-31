# Wrapping up Google Summer of Code

Summer of code is over! Here I keep a list of pointers to relevant work with its respective status. What I ended up doing was somewhat different from [my original proposal](gsoc-2020.pdf) as we prioritized along the way. It was a huge privilege to work on GHC with my mentors. Over the coming weeks I will gradually close off the work and hopefully get a bunch of this work merged. Overall the community feedback about the novel `arrayOf#` primitive has been very positive.

- [GHC Proposal: Extending array primitives](https://github.com/ghc-proposals/ghc-proposals/pull/335)
- [Merge request: Perf improvement for Enum instances (merged)](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3273)
- [Merge request: Add `arrayOf2#` primitive (closed)](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3372)
- [Merge request: Variadic `smallArrayOf#` primop (open)](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3571)
- [Benchmark: new `smallArrayOf#` primop](https://github.com/buggymcbugfix/arrayOf-benchmark)
- [Blog post: Benchmarking the new `smallArrayOf#` primop](../2020-08-17-array/post.md)
- [Talk: From faster array primops to static data; or "I signed up for Summer of Haskell and ended up writing C!?"](../../talks/2020-08-28-hiw/abstract.md)
- [Merge request: Update QuickCheck to support GHC-9.0 + bugfix (open)](https://github.com/nick8325/quickcheck/pull/314)
- [Merge request: New array primops for delete, insert, update (open)](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3951)
- [Benchmark: new `insert`, `update` and `delete` primops](https://github.com/buggymcbugfix/array-benchmark)
- [Library patch: use new primops in unordered-containers](https://github.com/buggymcbugfix/unordered-containers/commit/b23b1107d2095db5383eb9647c547fccc2b9dc3e)
- [Merge request: Implement `appendArray#` (external) primop (open)](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/3952)

## Thank yous

- Google for funding Summer of Code
- My mentors
   - Andreas Klebinger (Well-Typed)
   - Andrew Thaddeus Martin (Layer3)
   - Chessai (Mercury)
- My employer Sqream
- My PhD adviser Dominic Orchard
- My school, University of Kent
- My Family, especially my wife Elis
