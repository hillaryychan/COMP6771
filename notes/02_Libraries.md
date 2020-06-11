# Libraries

A **library** is a collection of implementations of behaviour, written in terms of a language, that has a well-defined interface by which behaviour is invoked.

Why do we want to use libraries?  
***"Every line of code you don't write is bug free"***

Code in in a popular library is ofter:

* well-documented
* well-tested
* well-reviewed
* has lots of feedback

In COMP6771, we use the following extremely popular libraries

* C++ standard library
* Abseil
* Catch2 (test framework)
* {fmt}
* gsl-lite
* range-v3

## Catch2 Library Recap

``` cpp
#include <catch2/catch.hpp>

TEST_CASE("empty vectors") { // Opens up a context for testing
    auto v = std::vector<int>();

    REQUIRE(v.empty()); // Aborts the test case (not the program) on failure.

    SECTION("check we can insert elements to the back") { // Opens a sub-context, where everything in
        v.push_back(5);                                   // the outer scope runs for *each* SECTION.
        REQUIRE(ranges::distance(v) == 1);

        CHECK(v[0] == 1); // Gives a meaningful message on failure, but doesn't abort.
        CHECK(v.back() == v[0]);

        SECTION("check we can insert elements to the front") {
            auto const result = v.insert(v.begin(), -1);
            REQUIRE(ranges::distance(v) == 2);
            CHECK(result == v.begin());
            CHECK(v[0] == -1);
            CHECK(v.front() == v[0]);

            CHECK(v[1] == 0);
            CHECK(v.back() == v[1]);
        }

        SECTION("check we can remove elements from the back") {
            v.pop_back();
            CHECK(v.empty()); // remember that each section inherits an independent context from its parent scope
        }
    }
}
```

## Containers

Containers are abstractions of common data structures. They are objects that you can "put"other objects "into".

Container operations vary in time and space complexity. Their performance has a basis in physics (see [here](TODO)).

`std::vector` is always the default container (see [here](TODO))

The complexity of common container operations:

![container operations](../imgs/2-5_container-operations.png)

### Sequence Containers

**Sequence containers** organise a finite set of objects into a strict **linear** arrangement.

| Container           | Arrangement             |
| ---                 | ---                     |
| `std::vector`       | Dynamically-sized array |
| `std::array`        | Fixed-sized array       |
| `std::deque`        | Double-ended queue      |
| `std::forward_list` | Singly-linked list      |
| `std::std::list`    | Doubly-linked list      |

### Unordered Associative Containers

**Unordered associative containers** provide **fast** retrieval of data based on keys. The keys are **hashed**.

| Container             | Arrangement                                       |
| ---                   | ---                                               |
| `std::unordered_set`  | A collection of unique keys                       |
| `absl::flat_hash_set` | A collection of unique keys                       |
| `std::unordered_map`  | Associative array that map unique keys to a value |
| `absl::flat_hash_map` | Associative array that map unique keys to a value |

The Abseil flat-hash containers offer significant performance benefits over the `std::` containers. They are interface-compatible.

### Associative Containers

**Associative containers** provide **fast** retrieval of data based on keys. The keys are **sorted**.

| Container       | Arrangement                                            |
| ---             | ---                                                    |
| `std::set`      | A collection of unique keys                            |
| `std:multiset`  | A collection of keys                                   |
| `std::map`      | Associative array that map unique keys to a values     |
| `set::multimap` | Associative array where one key may map to many values |

These containers are mostly interface-compatible with unordered associative containers.

## String Processing

### User-defined Literals

Normally when we initialise a string we use:

``` cpp
auto const greeting = std::string("hello, world");
```

**User-defined literals** (UDLs) can remove the need to write `std::string` all the time:

``` cpp
using namespace std::string_literals;
auto const greeting = "hello, world"s;
```

We should put `using`-directives in the smallest scope possible

``` cpp
#include <string>

auto main() -> int {
    using namespace std::string_literals;
    auto const greeting = "hello, world"s;
}
```

### String Concatenation

``` cpp
#include <absl/strings/str_cat.h>
#include <string>

auto const greeting = absl:StrCat("hello", "world", "!");
```

### String Formatting

``` cpp
#include <fmt/format.h>
#include <iostream>
#include <string>

auto const message1 = fmt::format("The meaning of life is {}", 42);
std::cout << message1 << '\n';

auto const message1 = fmt::format("pi has the value {}", 3.1415);
std::cout << message1 << '\n';

auto const message3 = fmt::format("life={}, pi={}", 42, 3.1415);
std::cout << message3 << '\n';

auto const message4 = fmt::format("life={life}, pi={pi}",
                                  fmt::arg("pi", 3.1415),
                                  fmt::arg("life", 42));
```

## `std::vector` Revisited

### Initialising Vectors

``` cpp
auto some_ints = std::vector<int>{0, 1, 2, 3, 2, 5};
REQUIRE(ranges::distance(some_ints) == 6);

// Querying a vector
CHECK(some_ints[0] == 0);
CHECK(some_ints[1] == 1);
CHECK(some_ints[2] == 2);
CHECK(some_ints[3] == 3);
CHECK(some_ints[4] == 2);
CHECK(some_ints[5] == 5);
```

### Adding Elements To a Vector

`std::vector` grows as we add elements

``` cpp
auto some_ints = std::vector<int>{0, 1, 2, 3, 2, 5};
REQUIRE(ranges::distance(some_ints) == 6);

some_ints.push_back(42);
REQUIRE(ranges::distance(some_ints) == 7);

CHECK(some_ints[6] == 42);
```

### Removing Elements From a Vector

`std::vector` shrinks as we remove elements

``` cpp
auto some_ints = std::vector<int>{0, 1, 2, 3, 2, 5};
REQUIRE(ranges::distance(some_ints) == 6);

some_ints.pop_back();
REQUIRE(ranges::distance(some_ints) == 5);
CHECK(some_ints.back() == 2);

// erase any occurrence of 2
std::erase(some_ints, 2);
REQUIRE(ranges::distance(some_ints) == 4);
CHECK(some_ints[2] == 3);
```

``` cpp
auto some_ints = std::vector<int>{0, 1, 2, 3, 2, 5};
REQUIRE(ranges::distance(some_ints) == 6);

some_ints.clear(); // removes *all* elements
CHECK(some_ints.empty());

auto const no_elements = std::vector<int>{};
REQUIRE(no_elements.empty());

CHECK(some_elements == no_elements);
```

### More Vector Initialisation

I want a vector with five zeros

``` cpp
auto all_default = std::vector<double>(5);
REQUIRE(ranges::distance(all_default) == 5);

CHECK(all_default[0] == 0.0);
CHECK(all_default[1] == 0.0);
CHECK(all_default[2] == 0.0);
CHECK(all_default[3] == 0.0);
CHECK(all_default[4] == 0.0);
```

I want a vector with three identical values

``` cpp
auto const initial_value = std::string("some words go here!");
auto all_same = std::vector<std::string>(3, intial_value);
REQUIRE(ranges::distance(all_same) == 3);

CHECK(all_same[0] == initial_value);
CHECK(all_same[1] == initial_value);
CHECK(all_same[2] == initial_value);

all_same[1] = "other words";
CHECK(all_same[0] != all_same[1]);
CHECK(all_same.front() == all_same.back());
```

## Stacks and Queues

``` cpp
enum class colour { red, green, blue, yellow };
enum class value { number, draw_two, draw_four, reverse, skip };

struct car {
    colour colour;
    value value;

    friend auto operator==(card, card) -> bool = default;
};

auto const red_number = card{colour::red, value::number};
auto const blue_number = card{colour::blue, value::number};
auto const green_draw_two = card{colour::green, value::draw_two};
auto const blue_skip = card{colour::blue, value::skip};
auto const yellow_draw_four = card{colour::yellow, value::draw_four};
```

### Stacks

#### Initialising and Adding Elements To a Stack

``` cpp
#include <stack>

auto deck = std::stack<card>();
REQUIRE(deck.empty());

deck.push(red_number);
deck.push(green_draw_two);
deck.push(green_draw_two);
deck.push(yellow_draw_four);
deck.push(blue_number);
```

#### Removing Elements From a Stack

``` cpp
CHECK(deck.top() == blue_number); // access top element
deck.pop();                       // remove top element

CHECK(deck.top() == yellow_draw_four);
deck.pop();
```

#### Comparing Two Stacks

``` cpp
auto const more_cards = deck;
REQURE(more_cards == deck);

deck.pop();
CHECK(more_cards != deck);
```

### Queues

#### Initialising and Adding Element To a Queue

``` cpp
#include <queue>

auto deck = std::queue<card>();
REQUIRE(deck.empty());

deck.push(red_number);
deck.push(green_draw_two);
deck.push(green_draw_two);
deck.push(yellow_draw_four);
deck.push(blue_number);
```

#### Removing Elements From a Queue

``` cpp
CHECK(deck.front() == red_number); // access first element
deck.pop();                        // remove first element

CHECK(deck.front() == green_draw_two);
deck.pop();
```

#### Comparing Two Queues

``` cpp
auto const more_cards = deck;
REQURE(more_cards == deck);

deck.pop();
CHECK(more_cards != deck);
```

## Iterators

> A range is an ordered sequence of elements with a designated start and rule for finishing

**Iterators** are a common theme between containers, algorithms and ranges. They provide and intermediate representation for which containers an be used by algorithms.

![iterators](../imgs/2-10_iterators.jpg)

Algorithms go through containers using iterators and return an iterator as a result.

Comparing iterators with other traversal methods:

| Operation      | Array-like                  | Node-based     | Iterator      |
| ---            | ---                         | ---            | ---           |
| Iteration type | `gsl_lite::index`           | `node*`        | `unspecified` |
| Read element   | `v[i]`                      | `i->value`     | `*i`          |
| Successor      | `j = i + n < ranges::distance(v) ? i + n : ranges::distance(v)` | `j = i->successor(n)` | `ranges::next(i, s, n')` |
| Predecessor    | `j = i - n < 0 ? 0 : i - n` | `j = i->precessor(n)` | `ranges::prev(i, s, n)` |
| Advance fwd    | `++i`                       | `i = i->next`  | `++i`         |
| Advance bwd    | `--i`                       | `i = i->prev`  | `--i`         |
| Comparison     | `i < ranges::distance(v)`   | `i != nullptr` | `i != s`      |

### Ranges

Generating a hand of cards

``` cpp
auto hand = std::vector<card>{
    red_number,
    blue_number,
    green_draw_two,
    blue_number,
    blue_skip,
    yellow_draw_four,
    blue_number,
    blue_number,
    blue_skip,
};
```

#### Counting Cards

``` cpp
#include <range/v3/algorithm.hpp>

CHECK(ranges::count(hand, red_number) == 1);
CHECK(ranges::count(hand, blue_number) == 4);
CHECK(ranges::count(hand, blue_skip) == 2);
```

#### Finding a Card

``` cpp
auto card_to_play = ranges::find(hand, blue_number);
REQUIRE(card_to_play != hand.end());
CHECK(*card_to_play == blue_number);
```

`card_to_play` is an iterator at position `1` with value `blue_number`

![ranges::find](../imgs/2-11_ranges-find.png)

``` cpp
auto const green_draw_four = card{colour::green, value::draw_four};
auto card_to_play = ranges::find(hand, green_draw_four);

REQUIRE(card_to_play == hand.cend());
```

`card_to_play` is an iterator at position `10` with no value since the card can't be fount

![ranges::find](../imgs/2-11_ranges-find2.png)

#### Erasing a Single Specific Card

``` cpp
auto card_to_play = ranges::find(hand, blue_number);
REQUIRE(card_to_play != hand.cend());
CHECK(*card_to_play == blue_number);

card_to_play = hand.erase(card_to_play);
REQUIRE(card_to_play != hand.cend());
CHECK(*card_to_play = green_draw_two);
```

`card_to_play` is an iterator at position `1` and value `green_draw_two` since `erase()` returns the iterator following the last removed element

![find::erase](../imgs/2-11_find-erase.png)

#### Getting a Range of Values

``` cpp
ranges::sort(hand);
REQUIRE(ranges::is_sorted(hand));

auto [first, last] = ranges::equal_range(hand, blue_number);
REQUIRE(first != last);
CHECK(ranges::distance(first, last) == 4);

CHECK(ranges::all_of(first, last, [blue_number](card const x) {  // uses lambda expressions
    return x == blue_number;
}));
```

`ranges::equal_range` returns iterators of the beginning and end that match the given value  
See more on lambda expressions [here](#lambda-expressions)

![ranges::equal_range](../imgs/2-11_ranges-equal-range.png)

#### Lambda Expressions

This can be done via **lambda expressions**

``` cpp
// count_if counts the number of elements that satisfy a given predicate
auto const blue_cards = ranges::count_if(hand, [](card const c) {
    return c.colour = colour::blue
});

auto const expected_blue_cards = 6;
CHECK(blue_cards == expected blue_cards)
```

Lambda unary predicate:

``` cpp
[](card const c) {
    return c.colour == colour::blue;
}
```

Explicit return type:

``` cpp
[](card const c) -> bool {
    return c.colour == colour::blue;
}
```

Multi-argument lambda:

``` cpp
auto const blue_then_yellow = [](card const x, card const y) {
    return x.colour == colour::blue and y.colour == colour::yellow;
};

auto const blue_card = ranges::adjacent_find(hand, blue_then_yellow);
REQUIRE(blue_card != hand.end());
CHECK(*blue_card == blue_skip);

auto const yellow_card = ranges::next(blue_card);
CHECK(*yellow_card == yellow_draw_four);
```

##### Lambda with value capture

Lambda allows **captures**, which is a list that defines ***outside variablaes*** that are accessible from within the lambda function body.

``` cpp
[blue_number](card const x) {
    return x == blue_number;
}
```

We can capture by **reference**:

``` cpp
[&cards_swapped](card const c) {
    // ... stuff
}

```

Example: two  players can swap a card of the same value (but for a different colour)

``` cpp
auto note_swaps(std::map<card, int>& cards_swapped, card const c) -> void {
    auto result = cards_swapped.find(c);
    if (result == cards_swapped.end()) {
        cards_swapped.emplace(c, 1);
        return;
    }

    ++result->second;
}

{
    auto cards_swapped = std::map<card, int>{};
    ranges::transform(hand, hand.begin(), [&cards_swapped](card const c) {
        if (c.colour != colour::blue) {
            return c;
        }

        note_swaps(cards_swapped, c);
        return card{colour::green, c.value};
    });

    CHECK(ranges::none_of(hand, [](card const c) {
        return c.colour == colour::blue;
    }));
}
{
    REQUIRE(cards_swapped.contains(blue_number));
    CHECK(cards_swapped.at(blue_number) == 4);
    auto const green_number = card{colour::green, value::number};
    CHECK(ranges::count(hand, green_number) == 4);
}
{
    REQUIRE(cards_swapped.contains(blue_skip));
    CHECK(cards_swapped.at(blue_skip) == 2);
    auto const green_skip = card{colour::green, value::skip};
    CHECK(ranges::count(hand, green_skip) == 2);
}
```

##### Lambda and Library Function Objects

``` cpp
#include <range/v3/functional.hpp>

ranges::equal{}
// is roughly equivalent to
[](auto const& x, auto const& y) {
    return x == y;
}
```

``` cpp
#include <range/v3/functional.hpp>

ranges::not_equal_to{}
// is roughly equivalent to
[](auto const& x, auto const& y) {
    return x != y;
}
```

``` cpp
#include <range/v3/functional.hpp>

ranges::plus{}
// is roughly equivalent to
[](auto const& x, auto const& y) {
    return x + y;
}
```

``` cpp
#include <range/v3/functional.hpp>

ranges::multiplies{}
// is roughly equivalent to
[](auto const& x, auto const& y) {
    return x * y;
}
```