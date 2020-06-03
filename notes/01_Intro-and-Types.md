# Intro & Types

Useful learning resources:

* [cppreference.com](https://en.cppreference.com/) Good for looking up APIs and recalling language rules  
**DO NOT USE CPLUSPLUS.COM**
* [abseil.io](https://abseil.io/) Good for looking up APIs and getting tips on how to use C++
* [code.visualstudio.com](https://code.visualstudio.com/) Documentation on how to use the course editor

## Basic Types

``` cpp
// `int` for integers
auto meaning_of_life = 42;

// `double` for rational numbers
auto size_feet_in_meters = 1.8288;

CHECK(six_feet_in_meters < meaning_of_life);

// `string` for text
auto course_code = std::string("COMP6771");

// `char` for single characters
auto letter = 'C';

CHECK(course_code.front() == letter);

// `bool` for truth
auto is_cxx = true;
auto is_danish = false;
CHECK(is_cxx != is_danish)
```

`auto` when used with variables deduces the type of the variable based on its initialised value

### Const

The keyword `const` specifies that a value cannot be modified.  
***Everything should be const*** unless you know it will be modified  
Const-correctness is a major topic in this course

``` cpp
auto const meaning_of_life = 42;
auto const size_feet_in_meters = 1.8288;
CHECK(six_feet_in_meters < meaning_of_life);
```

Why `const`?

* It provides clearer code; you can know a function won't try and modify something just by reading the signature).
* Immutable objects are easier to reason about
* The compiler **may** be able to make certain optimisations
* Immutable objects are **much** easier to use in multithreading situations

### Integer Expressions

C++ uses the same arithmetic operators as C

``` cpp
auto const x = 10;
auto const y = 173;

auto const sum = 183;
CHECK(x + y == sum);

auto const difference = 163;
CHECK(y - x == difference);
CHECK(x - y == -difference);

auto const product = 1730;
CHECK(x * y == product);

auto const quotient = 17;
CHECK(y / x == quotient);

auto const remainder = 3;
CHECK(y % x == remainder);
```

### Floating-point Expressions

``` cpp
auto const x = 15.63;
auto const y = 1.23;

auto const sum = 16.86;
CHECK(x + y == sum);

auto const difference = 14.4;
CHECK(y - x == difference);
CHECK(x - y == -difference);

auto const product = 19.2249;
CHECK(x * y == product);

auto const expected = 12.7073170732;
auto const actual = x/y;
CHECK(std::abs(expected - acutal) < acceptable_delta);
```

### String Expressions

``` cpp
auto const expr = std::string("Hello, expressions!");
auto const cxx = std::string("Hello, C++!");
CHECK(expr != cxx);
CHECK(expr.front() == cxx[0]);
auto consts concat = absl::StrCat(expr, " ", cxx);
CHECK(concat == "Hello, expressions! Hello, C++!");

auto expr2 = expr;

// Abort TEST_CASE if expression is false
REQUIRE(expr == expr2);
```

Note: `REQUIRE` aborts if the test case fails, while `CHECK` continue execution even if it fails.

C++ has value semantics:

``` cpp
auto const hello = std::string("Hello!")
auto hello2 = hello;

REQUIRE(hello == hello2);
hello2.append("2");
REQUIRE(hello != hello2);
CHECK(hello.back() == '!');
CHECK(hello2.back() == '2');
```

### Boolean Expressions

C++ introduces keywords `and`,`or` and `not` as alternative operators to `&&`, `||` and `!` respectively. Other operators like `=`, `!=`, `^` also have alternatives.

``` cpp
auto const is_comp6771 = true;
auto const is_about_cxx = true;
auto const is_about_german = false;
CHECK((is_comp6771 and is_about_cxx));
CHECK((is_about_german or is_about_cxx));
CHECK(not is_about_german);
```

## Type Conversion

In C++ we are able to convert types **implicitly** or **explicitly**. We will cover this later in more detail.

Implicit prompting conversions:

``` cpp
auto const i = 42;
{
    auto d = 0.0;
    REQUIRE(d == 0.0);

    d = i; // silent conversion from int to double
    CHECK(d == 42.0);
    CHECK(d != 41);
}
```

Explicit prompting conversions:

``` cpp
auto const i = 42;
{
    auto const d = static_cast<double>(i);
    CHECK(d == 42.0);
    CHECK(d != 41);
}
```

Explicit narrowing (lossy) conversions

``` cpp
auto const i = 42;
{
    // information lost, we're saying we know
    auto const b = gsl_lite::narrow_cast<bool>(i);
    CHECK(b == true);
    CHECK(b == gsl_lite::narrow_cast<bool>(42));
}
```
