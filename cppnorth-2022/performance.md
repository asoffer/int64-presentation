## Performance

NOTES:
* Let's talk about performance.

@@@

```diff
  typedef signed char int8;
  typedef short       int16;
  typedef int         int32;
- typedef long long   int64;
+ typedef long        int64;
```

~2% performance regression for websearch
<!-- .element: class="fragment" data-fragment-index="1" -->

NOTES:

* Remember when I said that processors don't distinguish between these types?
* Then you'll be as surprised as I was when I measured and found out [[NEXT]]
  that it caused a 2 percent performance regression in some cases.
* This really blew my mind when I first found out about it.

@@@

### Profile-Guided Optimization

NOTES:

* The idea here is you take profiling samples from your code and use them as
  input into the optimization pipeline for the next version of your binary.
* For every function in the binary we jot down some performance related notes,
  and then when we go to recompile, we use those notes to improve branch
  prediction, or cache locality, or any number of other things.

@@@

```cc
void MyFunction(long long n);
```

```cc
void MyFunction(long n);
```

NOTES:

* But profile guided optimization chooses the function based on its name and
  signature. When it's using an old profile it knows about a function that
  accepts `long long`.
* Then our new version comes along and, as far as it can tell, deletes that
  function, adding one with the same name but a different signature.
* The optimizer treats these as different functions and so it doesn't reuse the
  profiles.

@@@

# &#x1F600;
`-fprofile-remapping-file`

NOTES:

* Thankfully, this is relatively easy to fix. Clang can accept a file where you
  can specify patterns that should be considered identical for the purposes of
  using profiles for optimizations.
* We used this to say that should be treated identically.

@@@

# &#x1F600;
Testability

NOTES:

* It was crucial that we were able to determine if this was going to affect 
  performance *before* we released the binary.

@@@

# &#x1F629;
Unable to explore early

NOTES:

* There is no way for us to know how much this change affects profiles without
  measuring, and we can't measure until all the code compiles.
* What I'd really love here is the ability to selectively remove profiling data 
  before it goes into the optimizer. If we could selectively remove profiling 
  data for functions that accepted `long long`, we would have been able to get a
  decent approximation for the potential performance costs before we made any 
  actual changes.
@@@

# &#x1F605;
Institutional knowledge

NOTES:

* If a coworker hadn't mentioned this to me I would have had no expectation
  that this could have affected performance and wouldn't have bothered to
  measure.

@@@

# &#x1F605;
What if it didn't work?

NOTES:

* What if profile remapping was insufficient, or it caused other functions to
  be mapped together incorrectly?
