# Generic Programming

## Concepts

> *"Somewhat more formally, a **concept** is a description of requirements on one or more types stated in terms of the existence and properties of procedures, type attributes, and type functions defined on the types."* - Elements of Programming, by Alexander Stepanov and Paul McJones

### Requirements

A **constraint** is a syntax requirement. The compiler can check these.

An **axiom** is a semantic requirement. The compiler can't check these, but librarians may assume that they hold true forever. Precondition checks can sometimes check at runtime.  
Axioms are usually provided via comments in-tandem with the constraints.

**Complexity requirements** don't have a fancy name, and can't be checked by the implementation, nor the library author. A sophisticated [benchmarking tool](https://github.com/google/benchmark/#asymptotic-complexity) might be able to do so.

We say that we *satisfy* the requirements if all the constraints evaluate as true. We *model* the concept if, and only if, we satisfy the requirements and meed the axioms and complexity requirements.

### Concepts in C++

``` cpp
template<typename T>
concept type = true;

template<typename T>
requires type<T>
void hello(T) {
    std::cout << "Permissive, base-case\n";
}

int main() {
    hello(0);
    hello(0.0);
    hello("base-case");
}
```

is equivalent to:

``` cpp
template<typename T>
concept type = true;

template<type T> // same as using `typename` right now
void hello(T) {
    std::cout << "Permissive, base-case\n";
}

int main() {
    hello(0);
    hello(0.0);
    hello("base-case");
}
```

``` cpp
template<typename T>
concept integral = std::is_integral_v<T>;

template<integral T>
void hello(T) {
    std::cout << "Integral case\n";
}

int main() {
    hello(0);
    //hello(0.0);
    //hello("base-case");
}
```

``` cpp
template<typename T>
concept floating_point = std::is_floating_point_v<T>;

void hello(floating_point auto f) {
    std::cout << "Floating-point case\n";
}

int main() {
    // hello(0);
    hello(0.0);
    // hello("base case");
}
```

``` cpp
void hello(integral auto)       { std::cout << "Integral case\n";       }
void hello(floating_point auto) { std::cout << "Floating-point case\n"; }
void hello(auto)                { std::cout << "Base-case\n";           }

int main() {
    hello(0);           // prints "Integral case"
    hello(0.0);         // prints "Floating-point case"
    hello("base case"); // prints "Base-case"
}
```

### Refining and Weakening Concepts

A concept C2 ***refines*** a concept C1, if whenever C2 is modelled, C1 is also modelled.  
A concept C2 ***weakens*** a concept C1, if its requirements are a proper subset of C1

``` cpp
template<typename T>
concept integral = std::is_integral_v<T>;

void hello(integral auto) { std::cout << "Integral case\n"; }

template<typename T>
concept signed_integral = integral<T> and std::is_signed_v<T>;

void hello(signed_integral auto) { std::cout << "Signed integral case\n"; }

int main() {
    hello(0); // prints "Signed integral case"
    hello(0U); // prints "Integral case"
}
```

### Library Concepts

C++20 introduced concepts as a language feature, along with three families of highly usable concepts.

Due to many reasons they are available in GCC 10 and recent versions of MSVC but not Clang with `libc++`.

There is a 1:1 mapping between concepts in range-v1 and what's in the standard library.

| Concepts family | Relevant header | `CMakeLists` `LINK` | Namespace |
| ---             | ---             | ---                 | ---       |
| [Concepts library](https://en.cppreference.com/w/cpp/concepts)   | `<concepts/concepts.hpp>` | `concepts::concepts` | `concepts` |
| [Iterators library](https://en.cppreference.com/w/cpp/iterators) | `<range/v3/iterator.hpp>` | `range-v3`           | `ranges`   |
| [Ranges library](https://en.cppreference.com/w/cpp/ranges)       | `<range/v3/range.hpp>`    | `range-v3`           | `ranges`   |

### Why Concepts?

``` cpp
struct equal_to {
    template<typename T, typename U>
    auto operator()(T const& t, U const& u) const -> bool {
        return t == u;
    }
};

auto main() -> void {
    equal_to{}(0, 0);                           // okay, returns true
    equal_to{}(std::vector{0}, std::vector{1}); // okay, returns false
    equal_to{}(0, 0.0);                         // okay, returns true
    equal_to{}(0.0, 0);                         // okay, returns true
    equal_to{}(0, std::vector<int>{});          // error: `int == vector` not defined
}
```

``` cpp
struct equal_to {
    template<typename T, std::equality_comparable_with<T> U>
    // U must be comparable with T
    auto operator()(T const& t, U const& u) const -> bool {
        return t == u;
    }
};

auto main() -> void {
    equal_to{}(0, 0);                           // okay, returns true
    equal_to{}(std::vector{0}, std::vector{1}); // okay, returns false
    equal_to{}(0, 0.0);                         // okay, returns true
    equal_to{}(0.0, 0);                         // okay, returns true
    equal_to{}(0, std::vector<int>{});          // still error, but makes more sense
}
```

### Regularity

``` cpp
template<typename T>
concept movable = std::is_object_v<T>
              and std::move_constructible<T>
              and std::assignable_from<T&, T>
              and std::swappable<T>;

template<typename T>
concept copyable = std::copy_constructible<T>
               and std::movable<T>
               and std::assignable_from<T&, T const&>;

template<typename T>
concept semiregular = std::copyable<T> and std::default_initializable<T>;
```

## Iterators

Iterators let us write a generic find:

``` cpp
{
    auto const v = std::deque<int>{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto result = ranges::find(v.begin(), v.end(), 6);
    if (result != v.end()) {
        std::cout << "Found 6\n";
    } else {
        std::cout << "Didn't find 6.\n";
    }
}
```

### Iterator Invalidation

An **iterator** is an abstract notion of a **pointer**. When we modify the container, iterators may be invalidated. Using an invalid iterator is undefined.

Invalidation for [`push_back`](https://en.cppreference.com/w/cpp/container/vector/push_back):  
Think about the way a vector is stored. "If the new `size()` is greater than `capacity()` then all iterators and references (including the past-the-end iterator) are invalidated. Otherwise only the past-the-end iterator is invalidated."

``` cpp
auto v = std::vector<int>{1, 2, 3, 4, 5};

// Copy all 2s
for (auto it = v.begin(); it != v.end(); ++i) {
    if (*it == 2) {
        v.push_back(2);
    }
}
```

Invalidation for [`erase`](https://en.cppreference.com/w/cpp/container/vector/erase):  
"Invalidates iterators and references at or after the point of erase, including the `end()` iterator". For this reason, erase returns a new iterator.

``` cpp
auto v = std::vector<int>{1, 2, 3, 4, 5};

// Erase all even numbers
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);
    } else {
        ++it;
    }
}
```

Containers generally don't invalidate when you modify values, but they may invalidate when removing are adding elements. `std::vector` invalidates everything when adding elements. `std::unordered_(map|set)` invalidates all iterators when adding elements. `std::map` and `std::set` don't invalidated iterators upon insertion.

### Iterator Adaptors

There are two types of iterator adaptors;

* a wrapper around a type to ***grant that that iterator properties***
* a wrapper around an iterator type to ***grant additional or different iterator properties***

Examples: `std::reverse_iterator` is an iterator adaptor that transforms an existing iterator's `operator++` to mean "move backward" and `operator--` to mean "move forward".

``` cpp
std::reverse_iterator<std::vector<int>::iterator>
```

### Readable iterators

| Operation      | Array-like                | Node-based    | Iterator    |
| ---            | ---                       | ---           | ---         |
| Iteration type | `gsl_lite::index`         | `node*`       | unspecified |
| Read element   | `v[i]`                    | `i->value`    | `*i`        |
| Successor      | `j = i + n < ranges::distance(v) ? i + n : ranges::distance(v)` | `j = i->successor(n)` | `ranges::next(i, s, n)` |
| Advance fwd    | `++i`                     | `i = i->next` | `++i`       |
| Comparison     | `i < ranges::distance(v)` | `i != nullptr`| `i != s`    |

#### Indirectly Readable

A type `I` models the concept `std::indirectly_readable` if:

1. These types exist
    * `std::iter_value_t<I>` - a type we can create an lvalue from
    * `std::iter_reference_t<I>` - the type `I::operator*` returns
    * `std::iter_rvalue_reference_t<I>` - the type `ranges::iter_move(i)` returns  
    `ranges::iter_move(i)` is approximately `std::move(*i)`
2. These type pairs share a "relationship"
    * `std::iter_reference_t<I>` and `std::iter_value_t<I>&`
    * `std::iter_reference_t<I>` and `std::iter_rvalue_reference_t<I>`
    * `std::iter_rvalue_reference_t<I>` and `std::iter_value_t<I> const&`
3. Given an object `i` of type `I`, `*i` outputs the same thing when called with the same input

Generating `iter_` types:

``` cpp
template<typename T>
class linked_list {
public:
    class iterator;
private:
    struct node {
        T value;
        std::unique_ptr<T> next;
        T* prev;
    };
    std::unique_ptr<node> n;
};
```

``` cpp
template<typename T>
class linked_list<T>::iterator {
public:
    using value_type = T; // std::iter_value_t<iterator> is value_type

    auto operator*() const noexcept -> value_type const& {
        // iter_reference_t<iterator> is value_type&
        // iter_rvalue_reference<iterator> is value_type&&
        return pointee_->value;
    }
private:
    node* pointee_;

    friend class linked_list<T>;

    explicit iterator(node* pointee)
    : pointee_(pointee) {}
};

// static_assert is just an assert run at compile time
static_assert(std::indirectly_readable<linked_list<int>::iterator>);
```

See more on `indirectly_readable` at [cppreference](https://en.cppreference.com/w/cpp/iterator/indirectly_readable)  
See more on `iter_` at [cppreference](https://en.cppreference.com/w/cpp/iterator/iter_t)

#### Weakly Incrementable

A type `I` models the concept `std::weakly_incrementable` if:

1. `I` models `std::default_initializable` and `std::moveable`
2. `std::iter_difference_t<I>` exists and is a signed integer

Let `i` be an object of type `I`

1. `++i` is valid and returns a reference to itself
2. `i++` is valid and has the same domain as `++i`
3. `++i` and `i++` both advance `i`, with constant time complexity

Generating `std::iter_difference_t<I>`:

``` cpp
template<typename T>
class linked_list<T>::iterator {
public:
    using value_type = T;
    using difference_type = std::ptrdiff_t; // std::iter_difference_t<iterator> is difference_type

    iterator() = default;

    auto operator*() const noexcept -> value_type const& { /* ... */ }

    auto operator++() -> iterator& {
        pointee_ = pointee_->next.get();
        return *this;
    }

    auto operator++(int) -> void { ++*this; }
private:
    //...
};

static_assert(std::weakly_incrementable<linked_list<int>::iterator>);
```

See more on `weakly_incrementable` at [cppreference](https://en.cppreference.com/w/cpp/iterator/weakly_incrementable)

#### Iterator Basis

`std::input_or_output_iterator` is the root concept for all six iterator categories.

A type `I` models the concept `std::input_or_output_iterator` if:

1. `I` models `std::weakly_incrementable`
2. `*i` is a valid expression and returns a reference to an object

See more on `input_or_output_iterator` at [cppreference](https://en.cppreference.com/w/cpp/iterator/input_or_output_iterator)

#### Input Iterators

`std::input_iterator` describes the requirement for an iterator that can be read from.

A type `I` models the concept `std::input_iterator` if:

1. `I` models `std::input_or_output_iterator`
2. `I` models `std::indirectly_readable`
3. `I::iterator_catefgory` is a type alias derived from `std::input_iterator_tag`

Input iterators let us write a generic find:

``` cpp
template<std::input_iterator I, typename T>
// std::indirect_binary_predicate checks ranges::equal_to{}(*first, *&value) is possible
// This is how we check *first == value is valid
requires std::indirect_binary_predicate<ranges::equal_to, I, T const*>
auto find(I first, I last, T const& value) -> I {
    for (; first != last; ++first) {
        if (*first == value) {
            return first;
        }
    }
    return last;
}
```

Modelling `std::input_iterator`:

``` cpp
template<typename T>
class linked_list<T>::iterator {
public:
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using iterator_category = std::input_iterator_tag;

    iterator() = default;

    auto operator*() const noexcept -> value_type const& { ... }

    auto operator++() -> iterator& { ... }
    auto operator++(int) -> void { ++*this; }
private:
    //...
};

static_assert(std::input_iterator<linked_list<int>::iterator>);
```

See more on `std::input_iterator` at [cppreference](https://en.cppreference.com/w/cpp/iterator/input_iterator)

#### Readable Iterators TL;DR

![readable iterators tldr](../imgs/07-2-94_readable-iterators-tldr.png)
