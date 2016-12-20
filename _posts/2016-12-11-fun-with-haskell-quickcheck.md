---
layout: post
title: Fun with Haskell QuickCheck
---

Recently, I found out that there exists a library named QuickCheck that randomly generates test inputs for your code, so you don't have to.

At first it sounded really unwieldy - were they basically going to toss a bunch of garbage values, that would never actually be relevant in production, into my functions? Unsurprisingly, QuickCheck is much cleverer than that. QuickCheck expects a specification of a function's input, so the test inputs it creates actually match what your functions would be called with IRL.

Currently, QuickCheck implementations exist for [many languages](http://hypothesis.works/articles/quickcheck-in-every-language/). But it was originally written in Haskell, and since I'm trying to learn Haskell I'll be testing out [Haskell's QuickCheck](https://hackage.haskell.org/package/QuickCheck-2.9.2/docs/Test-QuickCheck.html)!

### The function I'll be testing
I am testing my implementation of function that simplifies a list of possibly redundant time intervals. A time interval will be modeled as a 2-tuple of `Int`s. The idea is that the function takes a list of time intervals representing when a business is open - for example, `[(1, 5), (5, 8)]`, and reduces it to its minimal representation - in this case, `[(1, 8)]`.

Click [here](https://github.com/dianjin/fun-with-quickcheck) for the code.

### Coming up with properties
*Disclaimer: I don't understand Haskell enough to give a formal definition of what's going on, so I'm mainly showing examples.*

Every QuickCheck test is associated with a property, which is basically a function that returns a `Bool`. The function `prop_idempotent` in the code below is an example of a property. It checks that the function I wrote returns the same thing when applied once vs. twice on the input:

```haskell
import Test.QuickCheck

type TimeInterval = (Int, Int)

simplifyTimes :: [TimeInterval] -> [TimeInterval]
simplifyTimes inputTimes = <my-implementation-here>

prop_idempotent :: [TimeInterval] -> Bool
  prop_idempotent inputTimes =
    simplifyTimes (simplifyTimes inputTimes) == simplifyTimes inputTimes
```

Now, when I call `quickCheck prop_idempotent` the results are printed to `stdout`. You can also call `verboseCheck prop_idempotent` to be shown the randomly generated inputs used on your program.

### Debugging

Idempotence does not imply that my function is correct, but my function is definitely incorrect if it isn't idempotent.

My first implementation of `simplifyTimes` was not correct. Here's the output I got from `quickCheck prop_idempotent`:

```
*** Failed! Falsifiable (after 10 tests and 7 shrinks):
[(0,0),(1,0),(0,1)]
```
A cool thing about QuickCheck is that it uses a shrinking heuristic to find a minimal test case that violates the property. That way, you can use the small failing input to debug your program.

The first thing I noticed when I saw the failing input was that it wasn't even a valid input for my function. I wrote my function assuming that the input `TimeInterval`s were 2-tuples representing the start time and end time respectively - meaning `(1, 0)` was not an acceptable `TimeInterval` for my function. Is there a way to restrict the kinds of input QuickCheck is generating?

### Generating valid inputs attempt 1: Conditional properties
Turns out there is this combinator, `==>`, an infix function that discards test cases that don't satisfy a predicate. This sounded promising, so I rewrote `prop_idempotent` thusly:

```haskell
prop_idempotent :: [TimeInterval] -> Property
prop_idempotent inputTimes =
  let
    validTimeInterval :: TimeInterval -> Bool
    validTimeInterval (startTime, endTime)
      endTime >= startTime
  in
    (all validTimeInterval inputTimes) ==>
      simplifyTimes (simplifyTimes inputTimes) == simplifyTimes inputTimes
```

Output from running `quickCheck prop_idempotent`:

```
*** Failed! Falsifiable (after 15 tests and 11 shrinks):
[(0,0),(0,1),(1,1)]
```
Since `quickCheck` is not deterministic, I called it a few more times to make sure it was always giving me a valid list of `TimeInterval`s (it was).
Having witnessed my function breaking on valid inputs, I went back to the drawing board and came up with something in which I had much more confidence. The new output:

```
*** Gave up! Passed only 65 tests.
```

Wot? I wasn't sure whether to celebrate QuickCheck "giving up" on testing my function. A [StackOverflow answer](http://stackoverflow.com/questions/18502798/why-does-quickcheck-give-up) suggested that giving up is bad, since it means QuickCheck did not exercise the function on enough random inputs. Out of 100 attempted test cases, 35 were discarded since they didn't satisfy the predicate I passed to `==>`.

According to the StackOverflow, there are 2 ways to prevent QuickCheck from giving up:

1. Pass in a bigger `maxSuccess` arg to manipulate the "Maximum number of successful tests before succeeding". This seemed like a quick and dirty hack, since QuickCheck would be wasting resources generating inputs that are ultimately thrown out.
2. Write your own custom generator that only produces valid test inputs, and pass that into `quickCheck`.

### Generating valid inputs attempt 2: Test data generators
From the outdated  [QuickCheck](http://www.cse.chalmers.se/~rjmh/QuickCheck/manual.html) manual:

"Test data is produced by test data generators. QuickCheck defines default generators for most types, but you can use your own with forAll, and will need to define your own generators for any new types you introduce. Generators have types of the form Gen a; this is a generator for values of type a. The type Gen is a monad, so Haskell's do-syntax and standard monadic functions can be used to define generators."

I still don't get what monads are, but after some head-scratching I managed to produce a custom generator for a list of valid `TimeInterval`s.

```haskell
arbitraryTimeIntervals :: (Integral a, Arbitrary a) => Gen [(a, a)]
  arbitraryTimeIntervals =
    let
      -- Positive :: a -> Positive a
      -- arbitrary :: Arbitrary a => Gen
      arbitraryTimeInterval = do
        Positive start <- arbitrary
        Positive duration <- arbitrary
        return (start, start + duration)
    in
      -- sized :: (Int -> Gen a) -> Gen a
      sized (\n -> vectorOf n arbitraryTimeInterval)
```
To be honest I don't really understand what that code is doing - especially the `Positive` constructor(?) thing. The way I understand it is that since `a` is in the typeclass `Integral`, any calls to `arbitrary` will yield integers that can then be "forced" into positive numbers with the Positive constructor(?).

Anyway, now that we have a good generator for test inputs, we can change our property back to:

```haskell
prop_idempotent :: [TimeInterval] -> Bool
prop_idempotent inputTimes =
  reduceOverlapping (reduceOverlapping inputTimes) == reduceOverlapping inputTimes
```
and call:

```haskell
-- forAll :: (Testable prop, Show a) => Gen a -> (a -> prop) -> Property
quickCheck (forAll arbitraryTimeIntervals prop_idempotent)
```
which for me resulted in:

```
+++ OK, passed 100 tests.
```
Cool, my function is idempotent! Or at least, QuickCheck couldn't find an input that proved it *wasn't* idempotent in 100 tries.

### Coming up with good properties

Obviously, demonstrating idempotence does not suggest that my function to simplify time intervals actually works. So I spent some time thinking of other assertions I could make on my function's output. Here are two, that I believe demonstrate correctness when both are satisfied:

1. `prop_all_inputs_covered`: Checks that every `TimeInterval` in the input list is covered by exactly one `TimeInterval` in the output list.
2. `prop_no_extraneous_output`: Checks that you get an empty list when you "subtract" every `TimeInterval` in the input list from the output list.

In other words, the first assertion checks that the input is a subset of the output, and the second checks that the output is a subset of the input. Checking for both is thus a check for set equality.

### Is QuickCheck worth it?

Implementing the second property I described in the section above did take a non-trivial amount of time. There is a cost associated with using QuickCheck - it does not free you from writing thorough assertions about your expected output. However, it does free you from having to come up with test cases or a way to generate them. The `TimeInterval` generator I wrote was declarative, and I didn't need to fiddle with randomness libraries or types.

Personally, I was pretty happy with the increase in confidence gained from the amount of time spent writing properties and generators for QuickCheck. Sure, I don't see myself using QuickCheck for small functions but it seems incredibly powerful as a way to simplify testing of bigger and more complex functions.

I think it would be great to get to a point where examining the tests alone provides enough confidence that the code works, and I think QuickCheck definitely moves us in that direction. I mean, it's obviously better than having faith that the developer has enumerated all the edge cases in their test inputs. Plus, generators (as opposed to hardcoded examples) makes the specification of the functions under test more comprehensive and approachable to other developers. Ugh, now I'm thinking about the futility of Python and JS docstrings...

I'm really sad I just found out about QuickCheck, it makes writing tests and debugging really enjoyable!
