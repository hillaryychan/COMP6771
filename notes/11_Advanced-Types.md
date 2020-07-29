# Advanced Types

## `decltype`

**`decltype`** is the semantic equivalent of a "typeof" function for C++.

Rules:

1. If expression `e` is any sort of:
    * variable in local scope
    * variable in namespace scope
    * static member variable
    * function parameters
    then result is  variable/parameters type T
2. if `e` is an lvalue, result is `T&`
3. if `e` is an xvalue, result is `T&&`
4. if `e` is a prvalue, result is `T`

A non-simplified set of rules can be found [here](https://en.cppreference.com/w/cpp/language/decltype)

Examples:

``` cpp
int i;
int j& = i;

decltype(i) x;   // int variable
decltype((i)) z; // int - lvalue
decltype(j) y;   // int& - variable
decltype(5);     // int - prvalue
```

### Determining Return Types

An iterator used over a templated collection returns a reference to an item at a particular index.  We know the return type should be `decltype**beg`, since we know the type of what is returned is type `*beg`, but we can't do the following as `beg` is not declared until after the reference to `beg`.

``` cpp
template <typename It>
decltype(*beg) find (It beg, It end, int index) {
    for (auto it = beg, int i; it != end; ++it; ++i) {
        if (i == index) {
            return *it;
        }
    }
    return end;
}
```

The introduction of C++ trailing return types solves this problem for us:

``` cpp
template <typename It>
auto find (It beg, It end, int index) -> decltype(*beg) {
    for (auto it = beg, int i; it != end; ++it; ++i) {
        if (i == index) {
            return *it;
        }
    }
    return end;
}
```

## Type Transformations

A number of **add**, **remove**, and **make** functions exist as part of [type traits](https://en.cppreference.com/w/cpp/header/type_traits) that provide an ability to transform types.

``` cpp
#include <iostream>
#include <type_traits>

template <typename T1, typename T2>
auto print_is_same() -> void {
    std::cout << std::is_same<T1, T2>() << "\n";
}

auto main() -> int {
    std::cout << std::boolalpha;
    print_is_same<int, int>(); // true

    print_is_same<int, int &>(); // false
    print_is_same<int, int &&>(); // false
    print_is_same<int, std::remove_reference<int>::type>(); // true

    print_is_same<int, std::remove_reference>int &>::type>(); // true
    print_is_same<int, std::remove_reference<int &&>::type>(); // true
    print_is_same<const int, std::remove_reference<const int &&>::type>(); // true
}
```

``` cpp
#include <iostream>
#include <type_traits>

auto main() -> int {
    using A = std::add_rvalue_reference<int>::type;
    using B = std::add_rvalue_reference<int&>::type;
    using C = std::add_rvalue_reference<int&&>::type;
    using D = std::add_rvalue_reference<int*>::type;

    std::cout << std::boolalpha;
    std::cout << "typedef of int&&:" << "\n";
    std::cout << "A: " << std::is_same<int&&, A>::value << "\n"; // true
    std::cout << "B: " << std::is_same<int&&, B>::value << "\n"; // false
    std::cout << "C: " << std::is_same<int&&, C>::value << "\n"; // true
    std::cout << "D: " << std::is_same<int&&, D>::value << "\n"; // false
}
```

Since C++14/C++17 you can use shortened type trait names:

``` cpp
#include <iostream>
#include <type_traits>

auto main() -> int {
    using A = std::add_rvalue_reference<int>;
    using B = std::add_rvalue_reference<int>;

    std::cout << std::boolalpha;
    std::cout << "typedefs of int&&:" << "\n";
    std::cout << "A: " << std::is_same<int&&, A>::value << "\n";
    std::cout << "B: " << std::is_same<int&&, B>::value << "\n";
}
```

## Binding

|                | lvalue | const lvalue | rvalue | const rvalue |
| ---            | ---    | ---          | ---    | ---          |
| template `T&&` | yes    | yes          | yes    | yes          |
| `T&`           | yes    |              |        |              |
| `T const&`     | yes    | yes          | yes    | yes          |
| `T&&`          |        |              | yes    |              |

Notes:

* `T const&` binds to **everything**
* template `T&&` binds to **everything**;

    ``` cpp
    template <typename T> void foo(T&& a);
    ```

Examples:

``` cpp
#include <iostream>

auto print(const std::string& a) -> void {
    std::cout << a << "\n";
}

auto goo() -> std::string const {
   return "C++";
}

auto main() -> int {
    std::string j = "C++";
    std::string const& k = "C++";
    print("C++"); // rvalue
    print(goo()); // rvalue
    print(j);     // lvalue
    print(k);     // const lvalue
}
```

``` cpp
#include <iostream>

template <typename T>
auto print(T&& a) -> void {
    std::cout << a << "\n";
}

auto goo() -> int const {
   return "C++";
}

auto main() -> int {
    int j = 1;
    int const& k = 1;
    print(1);     // rvalue, foo(int&&)
    print(goo()); // rvalue, foo(const int&&)
    print(j);     // lvalue, foo(int&)
    print(k);     // const lvalue, foo(int const&)
}
```

## Forwarding Functions

### Motivation for Forwarding Functions

**Attempt 1**: Take in a value. What's wrong with this code?

``` cpp
template <typename T>
auto wrapper(T value) {
    return fn(value);
}
```

It won't work if we pass in a non-copyable type and won't be efficient if we pass in a type that is expensive to copy.

**Attempt 2**: Take in a const reference

``` cpp
template <typename T>
auto wrapper(const T& value) {
    return fn(value);
}
```

But it creates a new problem; if the wrapper function needs to modify the value, the code won't compile. If we pass in a rvalue, the wrapper function will treat it as an lvalue.  

``` cpp
// Calls fn(x)
// should call fn(std::move(x))
wrapper(std::move(x));
```

**Attempt 3:** Take in a mutable reference. What's wrong with this?

``` cpp
template <typename T>
auto wrapper(T& value) {
    return fn(value);
}
```

If we pass in a const object, the code won't compile since it will lose its "const"-ness.  
If we pass in a rvalue, it will be treated as a lvalue

#### Interlude: Reference Collapsing

An rvalue reference to an rvalue reference becomes ("collapses into") an rvalue reference  
All other reference (i.e. all combinations involving a lvalue reference) collapses into an lvalue reference.

``` txt
T& &   -> T&
T&& &  -> T&
T& &&  -> T&
T&& && -> T&&
```

### Motivation for Forwarding Functions ctd

**Attempt 4**: Forwarding references. What's wrong with this?

``` cpp
template <typename T>
auto wrapper(T&& value) {
    return fn(value);
}
```

For an lvalue, it calls `fn(i)`

``` cpp
// Instantiation generated
auto wrapper<int&>((int&)&& value) {
    return fn(value);
}

// Collapses to
auto wrapper<int&>(int& value) {
    return fn(value);
}

int i;
wrapper(i);
```

For an rvalue, it also calls `fn(i)`. The parameter is an rvalue, but inside the function, `value` is an lvalue.

``` cpp
// Instantiation generated
auto wrapper<int&&>((int&&)&& value) {
    return fn(value);
}

// Collapses to
auto wrapper<int&&>(int&& value) {
    return fn(value);
}

int i;
wrapper(std::move(i));
```

What we wanted to generate was:

``` cpp
auto wrapper<int&>(int& value) {
    return fn(static_cast<int&>(value));
}

auto wrapper<int&&>(int&& value) {
    return fn(static_cast<int&&>(value));
}
```

It turns out there's a function for this already:

``` cpp
template <typename T>
auto wrapper<T&& value> {
    return fn(std::forward<T>(value));
    // Equivalent to (but don't do this, std::forward is easier and self-explanatory)
    // return fn(static_cast<T>(value));
}

wrapper(std::move(x));
```

**`std::forward`** returns:

* a reference to value for lvalues
* `std::move(value)` for rvalues

``` cpp
// This is approximately std::forward
template <typename T>
auto forward(T& value) -> T& {
    return static_cast<T&>(value);
}

template <typename T>
auto forward(T&& value) -> T&& {
    return static_cast<T&&>(value);
}
```

### `std::forward` and Variadic Templates

Often you need to call a function you know nothing about.

* It may have any amount of parameters
* Each parameter may be a different unknown type
* Each parameter may be an lvalue or rvalue

``` cpp
template <typename... Args>
auto wrapper(Args&&... args) {
    // Note that the ... is outside the forward call, and not right next to args
    // This is because we want to call
    // fn(forward(arg1), forward(arg2), ...);
    // and not
    // fn(forward(arg1, arg2, ...));
    return fn(std::forward<Args>(args)...);
}
```

### Uses of `std::forward`

The only real uses for `std::forward` is when you want to wrap a function. This could be because:

* You want to do something else before or after (e.g. `std::make_unique`/`std::make_shared` need to wrap it in the `unique_ptr`/`shared_ptr` variable)
* You want to do something slightly different (e.g. `std::vector::emplace` uses uninitialised memory construction)
* You want to add an extra parameter (e.g. always call a function with the last parameter as 1). This isn't usually very useful though, because it can be achieved with `std::bind` or lambda functions
