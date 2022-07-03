## `int64&`

NOTES:

* Let's start with a warm-up...

@@@

```cc []
class Widget {
 public:
  void GetValue(long long& n) const;
};
    
void Function(const Widget& w) {
  int64 n;
  w.GetValue(n);
}
```

NOTES:

* ...Here we have a widget class, and it has a memeber 
  function that takes a `long long` by reference. Callers pass an `int64`.
* Remember this compiles and runs properly when `int64` is an alias for
  `long long`.
* By show of hands: Who thinks they understand how this breaks if we were to 
  change `int64` to be an alias for `long`?

@@@

```txt
<source>:8:14: error: non-const lvalue reference to type
'long long' cannot bind to a value of unrelated type 'int64'
(aka 'long')
  w.GetValue(n);
             ^
<source>:3:28: note: passing argument to parameter 'n' here
  void GetValue(long long& n) const;
```

NOTES:

* Here's the error message that Clang gives us.
* [Read the message]
* Even though the `long` and `long long` are interconvertible, pointers and 
  references to them are not.
* So what's the fix here?

@@@

```cc []
class Widget {
 public:
  void GetValue(int64& n) const;  // Ideal
};
    
void Function(const Widget& w) {
  int64 n;
  w.GetValue(n);
}
```

NOTES:

* Ideally, we replace `long long` with `int64`.
* Our C++ style guide is pretty clear about prefering `int64` over `long long`.
* Now when we change the definition for `int64`, the caller and the callee change simultaneously.
* But this isn't always so easy.

@@@

```cc []
class Widget : public WidgetBase {
 public:
  void GetValue(long long& n) const override;
};
    
void Function(const Widget & w) {
  int64 n;
  w.GetValue(n);
}
```

NOTES:

* What if `GetValue` is virtual?
* Now we have to worry not just about `Widget`'s `GetValue` and its callers,
  but also every class derived from `WidgetBase` and all of their callers.
* What used to be a relatively small change has suddenly grown to lots of
  classes that need to be updated simultaneously.
* Thankfully, we didn't see too much of this, and fixing it by hand was entirely
  reasonable.
* Okay, so what did we learn? What went well?

@@@

# &#x1F600;
Style guide

NOTES:

* I think the star of this story is our style guide. All of the situations I
  showed here were pretty rare, and I think that's because there's such a strong
  adherence to the style guide, and the guidance is well thought out.

@@@

# &#x1F600;
Code owners were receptive to changes

NOTES:

* We've really heavily invested in the idea that everyone has a sort of joint
  ownership over the entire codebase
* I received very little pushback when I asked someone to review a change to
  code they work on.
* But not everything was smiley faces.

@@@

# &#x1F629;
Virtual member functions

NOTES:

* Virtual member functions were a real pain.
* None of our refactoring tools were really helpful here. It was just a matter
  elbow grease.
* But we did get lucky here...

@@@

# &#x1F605;
Few large/heavily-used class hierarchies

NOTES:

* ... there were very few class hierarchies here that needed to be changed atomically.
* I haven't done any counting here, but I'd guess that the same wouldn't be true for strings.
