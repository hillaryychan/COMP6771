# Class Types

## Scope

The scope of a variable is the part of the program where it is accessible.

Scope starts at variable definition. It usually ends at the next `}`.

We should define variable as close to their first usage as possible.

``` cpp
#include <iostream>

int i = 1;
int main() {
    std::cout << i << "\n";
    if (i > 0) {
        int i = 2;
        std::cout << i << "\n";
        {
            int i = 3;
            std::cout << i << "\n";
        }
        std::cout << i << "\n";
    }
    std::cout << i << "\n";
}
```

### Object Lifetimes

An object is a piece of memory of a specific type that holds some data.  
All variables are objects. Unlike many other languages, this does not add overhead.

Object lifetime starts when it comes in scope (i.e. "constructs" the object). Each type has ***one or more constructor*** that says how to construct it.  
Object lifetime ends when it goes out of scope (i.e. "destructs" the object). Each type has a different "destructor" which tells the compiler how to destroy it.

*This is the behaviour that primitive types follow. With `class`es we tend to think about it more explicitly*

### Construction

We generally use `()` to call functions, and `{}` to construct objects.  
`()` can be used for functions, and `{}` can be used for either. There are some ***rare*** occasions these are different as it is sometimes ambiguous between a constructor and an initialise list.

Examples with `std::vector`:

``` cpp
auto main() -> {
    // Always use auto on the left for this course, but you may see this elsewhere.
    std::vector<int> v11; // Calls 0-argument constructor. Creates empty vector.

    // There's no difference between these:
    // T variable = T{arg1, arg2, ...}
    // T variable{arg1, arg2, ...}
    auto v12 = std::vector<int>{}; // No different to first
    auto v13 = std::vector<int>(); // No different to the first

    {
        auto v2 = std::vector<int>{v11.begin(), v11.end()}; // A copy of v11.
        auto v3 = std::vector<int>{v2}; // A copy of v2.
    } // v2 and v3 destructors are called here

    auto v41 = std::vector<int>{5, 2}; // Initialiser-list constructor {5, 2}
    auto v42 = std::vector<int>(5, 2); // Count + value constructor (5 * 2 => {2, 2, 2, 2, 2})
} // v11, v12, v13, v41, v41 destructors called here
```

This also works for basic types, **but the default constructor has to be called manually** otherwise it will contain what was previously in memory. This potential bug can be hard to detect due to how function stacks work (e.g. a variable may *happen* to be `0`). It can be especially problematic with pointers.

``` cpp
#include <iostream>
double f() {
    return 1.1;
}

int main() {
    // One of the reasons we do auto is to avoid uninitialised values.
    // int n; // Not initialised (memory contains previous value)

    int n21{}; // Default constructor (memory contains 0)
    auto n22 = int{}; // Default constructor (memory contains 0)
    auto n3{5};

    // Not obvious you know that f() is not an int, but the compiler lets it through.
    // int n43 = f();

    // Not obvious you know that f() is not an int, and the compiler won't let you (narrowing
    // conversion)
    // auto n41 = int{f()};

    // Good code. Clear you understand what you're doing.
    auto n42 = static_cast<int>(f());

    // std::cout << n << "\n";
    std::cout << n21 << "\n";
    std::cout << n22 << "\n";
    std::cout << n3  << "\n";
    std::cout << n42 << "\n";
}
```

Object lifetimes are useful because there are times where we ***have*** to remember things, when we shouldn't have to worry about them.  
E.g omitting `f.close()` in languages like C, Java, Python. It's not easy to spot the mistake, and compilers often don't spot it either.

## Namespaces

**Namespaces** are used to express names that *belong together*.

``` cpp
// lexicon.hpp
namespace lexicon {
    std::vector<std::string> read_lexicon(std::string const& path);

    void write_lexicon(std::vector<std::string> const&, std::string const& path);
} // namespace lexicon
```

It also prevents similar names from clashing.

``` cpp
// word_ladder.hpp
namespace word_ladder {
    absl::flat_hash_set<std::string> read_lexicon(std::string const& path);
} // namespace word_ladder

// read_lexicon.cpp
namespace word_ladder {
    absl::flat_hash_set<std::string> read_lexicon(std::string const& path) {
        // open file...
        // read file into flat_hash_set...
        // return flat_hash_set
    }
} // namespace word_ladder
```

### Nested Namespaces

We can also have nested namespaces:

``` cpp
namespace comp6771::word_ladder {
    std::vector<std::vector<std::string>>
    word_ladder(std::string const& from, std::string const& to);
} // namespace comp6771::word_ladder

namespace comp6771 {
    // ...
    namespace word_ladder {
        std::vector<std::vector<std::string>>
        word_ladder(std::string const& from, std::string const& to);
    } // namespace word_ladder
} // namespace comp6771
```

It is preferrable to have top-level and occasionally two-tier namespaces to multi-tier.  
It is okay to won multiple namespaces per project, if they logically separate things.

### Unnamed namespaces

In C you had static functions that made functions local to a file.  
C++ uses **unnamed** namespaces to achieve the same effect. Functions that you don't want in your public interface should be put into unnamed namespaces.

Unlike named namespaces, it is okay to nest unnamed namespaces

``` cpp
namespace word_ladder {
    namespace {
        bool valid_word(std::string const& word);
    } // namespace
} // namespace word_ladder
```

### Namespace Aliases

We can give a namespace a new name

``` cpp
namespace chrono = std::chrono;
namespace views = ranges::views;
```

This is often good for shortening nested namespaces

### Qualifying Function Calls

There are certain complex rules about how overload resolution works that will surprise you, so it's a best practice to **always fully-qualify** your function calls (i.e. call them w.r.t their namespace)

``` cpp
int main() {
    auto const x = 10.0;
    auto const x2 = std::pow(x, 2);

    auto const ladders = word_ladder::generate("at", "it");

    auto const x2_as_int = gsl_lite::narrow_cast<int>(x2);
}
```

Using a namespace alias counts as "fully-qualified" ***only if*** the alias was fully qualified.

You should qualify your function calls even if you're in the ***same namespace***.

``` cpp
namespace word_ladder {
    namespace {
        bool valid_word(std::string const& word);
    }

    std::vector<std::vector<std::string>> generate(std::string const& from, std::string const& to) {
        auto const result = word_ladder::valid_word(word);
    }
}
```

You should qualify your function calls even if you're in the ***same nested namespace***.

``` cpp
namespace word_ladder::something::very_long {
    namespace {
        bool valid_word(std::string const& word);
    }

    std::vector<std::vector<std::string>> generate(std::string const& from, std::string const& to) {
        auto const result = word_ladder::someting::very_long::valid_word(word);
    }
}
```

## Object Oriented Programming

A class uses data abstraction and encapsulation to define an abstract data type:

* **Interface** - the operations used by the user (an API)
* **Implementation** - the data members, the bodies of the functions in the interface and any other functions not intended for general use
* **Abstraction** - separation of interface from implementation.  
Useful as class implementation can change over time
* **Encapsulation** - enforcement of this via information hiding

Example:

``` txt
bookstore.h (interface)
bookstore.cpp (implementation)
bookstore_main.cpp (knows the interface)
```

### C++ Classes

A **class**:

* defines a new type
* is created using keywords `class` or `struct`
* may define some members (functions, data)
* contains zero or more public and private sections
* is instantiated through a constructor

A **member function**:

* must be declared inside the class
* may be defined inside the class (it is then inline by default)
* may be declared const, when it doesn't modify the data members

The **data members** should be private, representing the state of an object.

This is how we support encapsulation and information hiding in C++:

``` cpp
class foo {
public:
    // Members accessible by everyone
    foo(); // The default constructor.

protected:
    // Members accessible by members, friends, and subclasses
    // Will discuss this when we do advanced OOP in future weeks.

private:
    // Accessible only by members and friends
    void private_member_function();
    int private_data_member_;

public:
    // May define multiple sections of the same name
};
```

A simple example of C++ classes:

``` cpp
#include <iostream>
#include <string>

class person {
public:
    person(std::string const& name, int age);
    auto get_name() -> std::string const&;
    auto get_age() -> int const&;

private:
    std::string name_;
    int age_;
};

person::person(std::string const& name, int const age) {
    name_ = name;
    age_ = age;
}

auto person::get_name() -> std::string const& {
    return name_;
}

auto person::get_age() -> int const& {
    return age_;
}

auto main() -> int {
    person p("Hayden", 99);
    std::cout << p.get_name() << "\n";
}
```

### Classes and Structs in C++

A `class` and a `struct` in C++ are almost exactly the same.  
The **only** difference is that:

* All members of a `struct` are **public by default**
* All members of a `class` are **private by default**

We use structs only when we want a simple type with little or no methods and direct access to data members (as a matter of style). This is a ***semantic*** difference, not a ***technical*** one

``` cpp
class foo {
    int member_; /* default private */
}
struct foo {
    int member_; /* default public */
}
```

### Class Scope

Anything declared inside the class needs to be accessed through the scope of the class. Scope are accessed using `::` in C++.

``` cpp
// foo.h

class Foo {
public:
    // Equiv to typedef int Age
    using Age = int;

    Foo();
    Foo(std::istream& is);
    ~Foo();

    void member_function();
};
```

``` cpp
// foo.cpp
#include "foo.h"

Foo::Foo() {
}

Foo::Foo(std::istream& is) {
}

Foo::~Foo() {
}

void Foo::member_function() {
    Foo::Age age;
    // Also valid, since we are inside the Foo scope.
    Age age;
}
```

### Incomplete Types

An **incomplete type** may only be used to define pointers and references, and in function declarations (but not definitions). Because of the restriction on incomplete types, a class cannot have data members in its own type.

``` cpp
struct node {
    int data
    // Node is incomplete -  this is invalide
    // This would also make no sense. What is sizeof(Node)?
    node next;
};
```

But the following is legal, since a class is considered declared once its class has been seen:

``` cpp
struct node {
    int data
    node* next;
};
```

### Constructors

**Constructors** define how class data members are initialised.  
A constructor has the same name as the class and ***no return type***.  
Default initialisation is handled through the default constructor. Unless we define our own constructors the compiler will declare a default constructor. This is known as teh **synthesised default constructor**.

``` txt
for each data member in declaration order
    if it has an in-class initialiser
        Initialise it using the in-class initialiser
    else if it is of a built-in type (numeric, pointer, bool, char, etc.)
        do nothing (leave it as whatever was in memory before)
    else
    Initialise it using its default constructor
```

The synthesised default constructor is generated for a class only if it declared no constructors.  For each member, it calls the in-class initialiser if present, otherwise it calls the default constructor (except for trivial types like `int`).  
It cannot be generated when any data members are missing both in-class initialised and default constructors.

``` cpp
class A {
    int a_;
}

class C {
    int i_{0}; // in-class initialiser
    int j_;    // untouched memory

    A a_; // This stops the default constructor from being synthesised
    B b_;
}

class B {
    B(int b) : b_{b} {}
    int b_;
}
```

#### Constructor Initialiser List

The initialisation phase occurs before the body of the constructor is executed, regardless of whether the initialised list is supplied.

A constructor will:

1. Construct all data members **in order of member declaration** (using the same rules as those used to initialise variables)
2. Execute the body of the constructor: the code may **assign** values to the data members to override the initial values

``` cpp
#include <string>

class nodefault {
public:
    explicit nodefault(int i)
    : i_{i} {}

private:
    int i_;
};

int const b_default = 5;

class b {
    // Constructs s_ with value "Hello world"
    explicit b(int& i)
    : s_{"Hello world"} , const_{b_default} , no_default_{i} , ref_{i} {}

    // Doesn't work - constructed in order of member declaration.
    /*explicit b(int& i)
    : s_{"Hello world"} , const_{5} , ref_{i} , no_default_{ref_} {}*/

    /*explicit b(int& i) {
        // Constructs s_ with an empty string, then reassigns it to "Hello world"
        // Extra work done (but may be optimised out).
        s_ = "Hello world";

        // Fails to compile (modifying a const object).
        const_string_ = "Goodbye world";
        // Fails to compile (references *must* be initialized in the constructor).
        ref_ = i;
        // This is fine, but it can't construct it initially.
        no_default_ = nodefault{1};
    }*/

    std::string s_;
    // All of these will break compilation if you attempt to put them in the body.
    const int const_;
    nodefault no_default_;
    int& ref_;
};
```

#### Delegating Constructors

A constructor may call another constructor inside the initialiser list.

Since the other constructor must construct all the data members, do not specify anothing else in the construction initialiser list.  
The other constructor is called completely before this one.  
This is one of the few good uses for default values in C++. Default values may be used instead of overloading and delegating constructors.

``` cpp
#include <string>

class dummy {
public:
    explicit dummy(int const& i)
    : s_{"Hello world"}
    , val_{i} {}

    explicit dummy()
    : dummy(5) {}

    std::string const& get_s() {
        return s_;
    }

    int get_val() {
        return val_;
    }

private:
    std::string s_;
    const int val_;
};

auto main() -> int {
    dummy d1(5);
    dummy d2{};
}
```

### Destructors

**Destructors** are called when the object goes out of scope. They are handy when

* freeing pointers
* closing files
* unlocking mutexes (from multithreading)
* aborting database transactions

``` cpp
class MyClass {
    ~MyClass() noexcept;
};

MyClass::~May noexcept {
    // Definition here
}
```
