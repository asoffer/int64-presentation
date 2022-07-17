## Overload Resolution

NOTES:

* Let's talk about templates

@@@

```cc []
#include <algorithm>
    
int64 ClampAboveZero(int64 n) {
  return std::max(n, 0LL);
}
``` 

NOTES:

* Show of hands, who sees what can go wrong here?
* As it turns out, when we change `int64` to be an alias for `long`, this will
  stop compiling, but you can't tell just by looking at this slide. You need to
  understand how `std::max` does type deduction.

@@@

```cc []
namespace std {

template <typename T>
const T& max(const T& lhs, const T& rhs) {
  ...
}

}  // namespace std
```

NOTES:

* So here's what `std::max` looks like. The body of the function isn't relevant
  for us today. What is important are the parameters. they're both references to
  a constant `T`.
* The type deduction algorithm puts them on equal footing. Neither is preferred
  over the other.
* When we go back to our example...

@@@

```cc []
#include <algorithm>
    
int64 ClampAboveZero(int64 n) {
  return std::max(n, 0LL);
}
``` 

NOTES:
* ... we see that if we change the meaning of `int64`,  the types now differ; one is 
  `long` and the other is `long long`, so we can't find a valid `T` that works
  for both.
* This problem showed up thousands of times.
* So how do we fix this case?
* We're going to use automated refactoring tools, but what do we want to change this example to?

@@@

```cc []
#include <algorithm>
    
int64 ClampAboveZero(int64 n) {
  return std::max<int64>(n, 0);
}
``` 

NOTES:
* Here's one option. Tell `std::max` explicitly which instantiation you want to
  use. Explicit is better than implicit, right?
* This isn't the option we went with.
* It's nice though right? We get to just write `0`. There's an implicit
  conversion from `int` to `int64` to make that work.
* But therein lies the problem. When you explicitly specify template arguments,
  you're effectively working with a normal function, and all the implicit
  conversions apply...

@@@

```cc []
#include <algorithm>
    
int64 ClampAboveZero(int64 n) {
  return std::max<int64>(n, true);  // Why?!
}
``` 

NOTES:

* ...Not just the useful ones.
* We didn't want to introduce that pattern into our codebase at scale, because 
  we know it's an error prone one,
* So instead we fixed it like this.

@@@

```cc []
#include <algorithm>
    
int64 ClampAboveZero(int64 n) {
  return std::max(n, int64{0});
}
```

NOTES:

* By brace initializing an `int64` and still relying on type deduction, we can 
  be assured that no implicit conversions take place.
* So ultimately we built a Clang-Tidy check that makes this change, and ran it over 
  our entire codebase.
* It's worth asking the question: "Why was `0LL` there in the first place?"
* Most likely, folks originally typed `std::max(n, 0)`, the compiler complained
  and their response to an error message about mismatched types and `long long`
  was to add the suffix to zero. That's entirely understandable but also rather unfortunate.
* Let's change it up a little bit...

@@@

```cc[]
template <typename T>
bool Compare(const T& lhs, const T& rhs) {
  return lhs < rhs;
}





bool IsGreaterThanZero(int64 value) {
  return Compare(0LL, value);
}
```

&nbsp;<br/>

NOTES:

* ...here we have the same pattern. Type deduction for `Compare` is going to
  require the two parameters to have the same type.
* But I've left some space around line 6, so you know I'm going to throw a
  wrinkle in there.

@@@

```cc[]
template <typename T>
bool Compare(const T& lhs, const T& rhs) {
  return lhs < rhs;
}

bool Compare(bool lhs, bool rhs) {
  return lhs < rhs;
}

bool IsGreaterThanZero(int64 value) {
  return Compare(0LL, value);
}
```

&nbsp;<br/>

NOTES:

* Let's think about what happens if we add an overload specifically for `bool`.
* This is an actual example we found in the wild. I'm not sure why this was
  added to the codebase, but this was the code as I found it.
* What happens now?

@@@

```cc[]
template <typename T>
bool Compare(const T& lhs, const T& rhs) {
  return lhs < rhs;
}

bool Compare(bool lhs, bool rhs) {
  return lhs < rhs;
}

bool IsGreaterThanZero(int64 value) {
  return Compare(0LL, value); // Calls the bool overload
}
```

[See on Compiler Explorer](https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:19,endLineNumber:1,positionColumn:19,positionLineNumber:1,selectionStartColumn:19,selectionStartLineNumber:1,startColumn:19,startLineNumber:1),source:'using+int64+%3D+long%3B%0A%0Atemplate+%3Ctypename+T%3E%0Abool+Compare(T+const%26+lhs,+T+const%26+rhs)+%7B%0A++return+lhs+%3C+rhs%3B%0A%7D%0A+%0Abool+Compare(bool+lhs,+bool+rhs)+%7B%0A++return+lhs+%3C+rhs%3B%0A%7D%0A+%0Abool+IsGreaterThanZero(int64+value)+%7B%0A++return+Compare(0LL,+value)%3B+//+Calls+the+bool+overload%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:36.0313315926893,l:'4',m:100,n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang_trunk,filters:(b:'0',binary:'1',commentOnly:'0',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'-std%3Dc%2B%2B17',selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1,tree:'1'),l:'5',n:'0',o:'x86-64+clang+(trunk)+(C%2B%2B,+Editor+%231,+Compiler+%231)',t:'0'),(h:output,i:(compilerName:'x86-64+clang+(trunk)',editorid:1,fontScale:14,fontUsePx:'0',j:1,wrap:'1'),l:'5',n:'0',o:'Output+of+x86-64+clang+(trunk)+(Compiler+%231)',t:'0')),k:63.9686684073107,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)

NOTES:

* This compiles and calls the bool overload.
* Just as before, when we change the type of int64, the template parameters 
  don't match anymore. But substitution failure isn't an error. It's silently 
  ignored and we try a different overload.
* And look at that! Both `long` and `long long` implicitly convert to `bool`.
* So the bool overload is called, but that produces subtly incorrect results. 
  The literal zero will be converted to `false`. If the `int64` `value` where 
  negative, the answer should certainly be `false`. Instead it would be
  converted to `true` and the comparison would say that `false` is less than 
  `true`, so the result would incorrectly return `true`.
* Terrifying.
* Alright, lets think about our questions. What went well?

@@@

# &#x1F600;
Clang Tools

NOTES:

* Being able to write and run Clang-Tidy checks over our entire codebase made this problem feasible to solve.

@@@

# &#x1F600;
Unit tests

NOTES:

* It was a unit test that caught this boolean `Compare` overload.
* Unit tests don't just make sure your code works today. They make sure it 
  continues to work when I go and fiddle with what `int64` means.

@@@

# &#x1F629;
Diagnostics lead engineers to write antipatterns.

NOTES:

* On the other hand, I think that diagnostics caused a lot of trouble here.
* To be clear, I don't think the diagnostics themselves are to blame, nor the 
  engineers interpretting them. But in a codebase that highly discourages
  spelling out `long long` adding suffixes to literals should also be 
  discouraged.
* I think this could be solved with Clang-Tidy, but I'd really love scriptable
  diagnostics where I could tailor the suggestions to my particular style guide.

@@@

# &#x1F629;
Overload resolution is complicated

NOTES:

* Overload resolution is complicated.
* After staring at this for a year, I could tell you some best practices in how 
  to design overload sets.
* But it's unreasonable to expect the average engineer to learn these.
