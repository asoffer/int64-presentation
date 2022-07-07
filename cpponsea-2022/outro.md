## Lessons Learned

NOTES:

Looking back, at the good and the bad that we identified, I think there are
three major themes here.

@@@

Change is inevitable

NOTES:

* Hyrum Wright, of "Hyrum's Law" fame often says "you will eventually want to 
  change every line of code you have ever written. And I think this really well
  exemplified here. What could be more stable than a primitive type alias.
* And if you're looking at this thinking, "yeah, we'll never do that." you may
  be right. But this is just one example of a situation where you might need to
  make changes that affect a codebase broadly.

@@@

C++ is Complicated

NOTES:

* C++ is complicated. And while the subset that we call "modern C++" is 
  certainly simpler, our codebases are, for the most part, a mix of modern and
  legacy c++. Insofar as we have legacy code, we need to understand all of its
  complexity.
* As the language and our codebases grow we have to be able to manage that
  complexity.
* There are lots of tools at our disposal here: Style guides, refactoring tools,
  and improvements in compiler diagnostics, are just a few that this effort has
  shown to be valuable.

@@@

Quick Feedback is Vital

NOTES:

* If you look broadly at the issues we discussed, the easiest ones to address
  were the ones where we had a compiler error.
* As we pushed the feedback further and further away, to linker errors, to test
  failures, and ultimately to performance regressions, the costs and the risks
  of fixing issues goes up dramatically.
* This is why things like clang-tidy and sanitizers are so valuable.
  The best thing you can do with potential problems is learn about them early.
PAUSE
* And with that, I want to say thank you, and open it up for questions.

@@@@@

# &#x1F64B;
Questions?

<br/>

https://asoffer.github.io/int64-presentation/cpponsea-2022/
