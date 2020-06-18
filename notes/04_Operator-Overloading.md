# Operator Overloading

We can overload operators and specify how we want them to behave.  
It allows us to use currently understood semantics (all of the operators) and gives us a common and simple interface to define class methods.

``` cpp
#include <iostream>
class point {
public:
    point(int x, int y)
    : x_{x}
    , y_{y} {};

    friend point operator+(point const& lhs, point const& rhs) {
        return point(lhs.x_ + rhs.x_, lhs.y_ + rhs.y_);
    }
    friend std::ostream& operator<<(std::ostream& os, point const& p) {
        os << "(" << p.x_ << "," << p.y_ << ")";
        return os;
    }

private:
    int x_;
    int y_;
};

auto main() -> int {
    point p1{1, 2};
    point p2{2, 3};
    std::cout << p1 + p2 << "\n";
}
```

## Friend

A class may declare `friend` functions or classes.

Those functions/classes are non-member functions that may access private parts of the class.  
This is, in general, a ***bad idea***, but there are a few cases where it may be required:

* Non-member operator overloads
* Related classes:
    * A `Window` class might have `WindowManager` as a friend
    * A `TreeNode` class might have a `Tree` as a friend
    * Container could have `iterator_t<Container>` as a friend - though a nested class may be more appropriate

Use friends when:

* The data should not be available to everyone
* There is a piece of code very related to this particular class.

***In general, we prefer to define friends directly in the class they relate to.***

## Operator Overloads

C++ supports a rich set of operator overloads.

All operator overloads must have at least one operant of its type.

Advantages:

* Reuse existing code semantics
* No verbosity required for simple operations

Disadvantages:

* Lack of context on operations

**Only create an overload if you type has a single, obvious meaning to an operator**.

Operator overload design:

![operator overload design](../imgs/4-6_operator-overload-design.png)

Use members when the operation is called in the context of a particular instance  
Use friends when the operation is called without any particular instance (even if they don't require access to private details)

### Overload: I/O

This is equivalent to `.toString()` method in Java  
Scope to overload for different types of output and input streams

``` cpp
#include <istream>
#include <ostream>
class point {
public:
    point(int x, int y)
    : x_{x}
    , y_{y} {};
    friend std::ostream& operator<<(std::ostream& os, const point& type) {
        os << "(" << p.x_ << "," << p.y_ << ")";
        return os;
    }
    friend std::istream& operator>>(std::istream& is, point& type) {
        // To be done in tutorials
    }
private:
    int x_;
    int y_;
};

auto main() -> int {
    point p(1, 2);
    std::cout << p << '\n';
}
```

### Overload: Compound Assignment

Sometimes particular methods might not have any real meaning, and they should be omitted (in this case, what does dividing two points together mean).

Each class can have any number of `operator+=` operators, but there can only be one `operator+=(X)` where X is a type.  That's why in this case we have two multiplier compound assignment operators

``` cpp
class point {
public:
    point(int x, int y)
    : x_{x}
    , y_{y} {};

    point& operator+=(point const& p);
    point& operator-=(point const& p);
    point& operator*=(point const& p);
    point& operator/=(point const& p);
    point& operator*=(int i);

private:
    int x_;
    int y_;
};

point& point::operator+=(point const& p) {
    x_ += p.x_;
    y_ += p.y_;
    return *this;
}

point& operator+=(point const& p) { /* what do we put here? */}
point& operator-=(point const& p) { /* what do we put here? */}
point& operator*=(point const& p) { /* what do we put here? */}
point& operator/=(point const& p) { /* what do we put here? */}
point& operator*=(int i) { /* what do we put here? */}
```

#### Operator Pairings

Many operators should be grouped together. This table should help you work out which are minimal set of operators to overload for any particular operator.

| If you overload      | Then you should also overload                     |
| ---                  | ---                                               |
| `operator OP=(T, U)` | `operator OP(T, U)`                               |
| `operator+(T, U)`    | `operator+(U, T)`                                 |
| `operator-(T, U)`    | `operator+(T, U)`, `operator+(T)`, `operator-(T)` |
| `operator/(T, U)`    | `operator*(T, U)`                                 |
| `operator%(T, U)`    | `operator/(T, U)`                                 |
| `operator++()`       | `operator++(int)`                                 |
| `operator--()`       | `operator++()`, `operator--(int)`                 |
| `operator->()`       | `operator*()`                                     |
| `operator+(T)`       | `operator-(T)`                                    |
