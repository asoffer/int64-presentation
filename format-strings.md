## Format-strings

NOTES:

* Okay, let's talk about format-strings.

@@@

```cc[]
#include "stringprintf.h"

std::string PrintToString(int64 n) {
  return StringPrintf("The number is %lld", n);
}
```

```txt
file.cc:4:39: error: format specifies type 'long long' but
the argument has type 'long' [-Werror,-Wformat]
  return StringPrintf("The number is %lld", n);
```
<!-- .element: class="fragment" data-fragment-index="1" -->

NOTES:

* `StringPrintf` is a function, much like `printf` but instead of writing to 
  standard output, it writes to a string.
* All of the things we're going to discuss apply equally well to `printf`.
* Show of hands, who sees what the problem is here?

NEXT

* It's the same sort of pattern, where two components need to have their types 
  match, but one of them is hard-coded as `long long`. In this case, the format 
  specifier `%lld` is hard-coded.

* So we have several hundred thousand calls to these types of functions. We're 
  definitely going to use automated refactoring tools on this one.

* But beyond that, there are a few hundred other functions that are annotated with...

@@@

```cc[]
class Logger {
 public:
  void Log(const char *fmt, ...) 
    __attribute__((format(printf, 1, 2)));
 ...
};
```

NOTES:

* ... this attribute, saying that these functions also want format string checking.
* And many of these were ...

@@@

```cc[]
class Logger {
 public:
  virtual void Log(const char *fmt, ...) 
    __attribute__((format(printf, 1, 2)));
 ...
};
```

NOTES:

* ... virtual.
* We don't really have any automated refactoring tools at our disposal for these cases.
* To be honest we just put in a lot of elbow grease.
* I can talk at length about the challenges here, but I actually want to focus 
  on the common case that we can refactor automatically.

@@@

```diff
- #include "stringprintf.h"
+ #include "absl/strings/str_format.h"

  std::string PrintToString(int64 n) {
-   return StringPrintf("The number is %lld", n);
+   return absl::StrFormat("The number is %d", n);
  }
```

NOTES:

* We decided to reinvent printf to help solve this problem.
* `absl::StrFormat` is a reimplementation of `StringPrintf` that's smarter about
  types. It doesn't need the type information in the format specifier, because 
  it gets that information from the arguments.
* It still does type checking, so you can't accidentally use `%s` with an integer.
* I think this is one of the surprising benefits of this effort. 
  `absl::StrFormat` is a much better user experience than `StringPrintf` and we 
  probably would not have thought to take on that effort if it weren't for `int64`.
* And it's better for reasons beyond integers. For example...

@@@

```cc []
struct LogMessage {
  std::string message;

  std::string DebugString() const {
    return StringPrintf(
        R"(LogMessage(message = "%s", severity = %d))", 
        message.c_str(), kSeverity);
  }

  static const int kSeverity = 10;
};
```

<br/>
<p>&nbsp;</p>

NOTES:
* With StringPrintf, you can't pass a string directly, you need to call `.c_str()`.
* But with `absl::StrFormat`, ...

@@@

```cc []
struct LogMessage {
  std::string message;

  std::string DebugString() const {
    return absl::StrFormat(
        R"(LogMessage(message = "%s", severity = %d))", 
        message, kSeverity);
  }

  static const int kSeverity = 10;
};
```

Error: Undefined symbol **`LogMessage::kSeverity`**
<!-- .element: class="fragment" data-fragment-index="1" -->

NOTES:

* ...you can pass a `string` or a `string_view`, and it just works.
* So even though we didn't technically need to make these changes as part of 
  the `int64` effort, we decided to do so, because, how much harder could it be?

NEXT

* As shown, this snippet of code will compile just fine but give you a linker error.
* This error showed up hundreds of times and to this day is the most baffling 
  error message I have seen from a toolchain.
* What do you mean it's not defined? I'm staring at the definition is right there!
* The problem here is not actually visible on this slide. It's a combination 
  of some esoteric language rules and the definition of `absl::StrFormat`.

@@@

```cc []
// File: stringprintf.h
std::string StringPrintf(const char *fmt, ...) {
  ...
}
```

```cc []
// File: absl/strings/str_format.h
template <typename... Args>
std::string StrFormat(const FormatSpec<Args...>& fmt,
                      const Args&... args) {
  ...
}

```

NOTES:

* `StringPrintf` takes a `const char*` for its format and then C-style variadic arguments.
  One thing to note is that C-style variadic arguments are passed by value, so 
  any arguments passed are copied.
* `absl::StrFormat` is a variadic template, and pasess its arguments by 
  reference. Because it accepts `std::string`, it definitely doesn't want to 
  make copies.
* Now, I'm not going to show you the C++ standard, because I know everyone here has it
  memorized anyway, but this is the relevant section

@@@

[basic.def.odr.5](http://eel.is/c++draft/basic.def.odr#5)

NOTES:

* And roughly speaking it says if you pass a variable by reference, it needs a definition.
* Initializing a static inside the class doesn't count. Why? I have no idea.
* But, if the variable is `constexpr` it doesn't need a definition.

@@@

```cc []
struct LogMessage {
  std::string message;

  std::string DebugString() const {
    return absl::StrFormat(
        R"(LogMessage(message = "%s", severity = %d))", 
        message, kSeverity);
  }

  static const int kSeverity = 10;
};
```

Error: Undefined symbol **`LogMessage::kSeverity`**

NOTES:

* And that leads us to our solution.

@@@

```cc []
struct LogMessage {
  std::string message;

  std::string DebugString() const {
    return absl::StrFormat(
        R"(LogMessage(message = "%s", severity = %d))", 
        message, kSeverity);
  }

  static constexpr int kSeverity = 10;
};
```

&nbsp;<br/>
&nbsp;<br/>

NOTES:

* Turn `const` into `constexpr`.
* So we built a Clang-Tidy check to catch these cases.

@@@

# &#x1F600;
Clang tooling

NOTES:

* Clang tools were really instrumental to making this problem manageable.
* There's just no way we would have been able to make hundreds of thousands of 
  edits to our codebase safely without them.
* On the flip side...
@@@

# &#x1F629;
Language rules are baffling

NOTES:

* ... C++ is confusing.
* I don't intend this as a dig at the standards committee. There's either a good
  reason for these rules, or there's just too much work for too few people... 
  either way it's not their fault.
* But regardless, the complexity harms developers.

@@@

# &#x1F629;
Missing definitions identified late

NOTES:
* Another problem is that the missing definitions were identified late. What I 
  mean by that we know that many uses of these variables probably needed to have
  a definition, but the problem was only identified when someone actually began
  to use it.

@@@

# &#x1F629;
[Hyrum's law](http://hyrumslaw.com)

NOTES:

* And of course, Hyrum's law.
* No one promised this StringPrintf wasn't going to ODR-use its arguments, but 
  with a few hundred thousand calls, someone is going to end up relying on that
  fact.

@@@

# &#x1F605;
Most format strings checked at compile-time.

NOTES:

* The place where I think we got lucky is most format strings were compile-time checked.
* If we didn't have that, we wouldn't have had particularly robust ways to verify that
  our changes were safe.
* If we didn't have the format string checking, we wouldn't have gotten all these compiler
  errors, but we would have been able to silently change the type to not match the format
  specifier, and that's undefined behavior.
* In practice, it's probably undefined behavior that doesn't matter, but it's not a bet I'm
  willing to make.
