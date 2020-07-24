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
    template <typename T>
    void foo(T&& a);
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

What's wrong with this code?

``` cpp
template <typename T>
auto wrapper(T value) {
    return fn(value);
}
```

TODO

This solves our previous problem:

``` cpp
template <typename T>
auto wrapper(const T& value) {
    return fn(value);
}
```

But it creates a new problem; the code won't work if `fn` takes in rvalues.  
We could make a separate rvalue definition and try template `T&&`, which binds to everything correctly:

``` cpp
template <typename T>
auto wrapper(const T& value) {
    return fn(value);
}

// Calls fn(x)
// Should call fn(std::move(x))
wrapper(std::move(x));
```

This solves our previous problem, but we still need to come up with a function that matches the pseudocode.

``` cpp
template <typename T>
auto wrapper(T&& value) {
    // psuedocode
    return fn(value is lvalue ? value : std::move(value));
}

wrapper(std::move(x));
```

### `std::forward`

**`std::forward`** returns:

* a reference to value for lvalues
`std::move(value)` for rvalues

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

``` cpp
template <typename T>
auto wrapper(T&& value) -> void {
    return fn(std::forward<T>(value));
}

wrapper(std::move(x));
```

### `std::forward` and Variadic Templates

Often you need to call a function you know nothing about. It may have any amount of parameters; each parameter may be a different unknown type, each parameter may be an lvalue or rvalue.

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
