# Resource Management

In C++, an object is a region of memory associated with a type. Unlike some other languages (Java), basic types such as `int` and `bool` are objects.

For the most part, C++ objects are designed to be intuitive to use.  
Some special things we can do with objects are:

* Create
* Destroy
* Copy
* Move

When writing a class, always consider the ***"rule of 5"***:

* Destructor
* Move assignment
* Move constructor
* Copy assignment
* Copy constructor

Here's a look at `std::vector<int>` under the hood:

``` cpp
class my_vec {
    // Constructor
    my_vec(int size): data_{new int[size]}, size_{size}, capacity_{size} {}

    // Copy constructor
    my_vec(my_vec const&) = default;
    // Copy assignment
    my_vec& operator=(my_vec const&) = default;

    // Move constructor
    my_vec(my_vec&&) noexcept = default;
    // Move assignment

    my_vec& operator=(my_vec&&) noexcept = default;
    // Destructor
    ~my_vec() = default;

    int* data_;
    int size_;
    int capacity_;
}

auto main() -> int {
    // Call constructor.
    auto vec_short = my_vec(2);
    auto vec_long = my_vec(9);
    // Doesn't do anything
    auto& vec_ref = vec_long;
    // Calls copy constructor.
    auto vec_short2 = vec_short;
    // Calls copy assignment.
    vec_short2 = vec_long;
    // Calls move constructor.
    auto vec_long2 = std::move(vec_long);
    // Calls move assignment
    vec_long2 = std::move(vec_short);
}
```

Though you should always consider it, you should rarely have to write it.

If all data members have one of these defined, then the class should automatically define this for you. But, this may not always be what you want. C++ follows the principle of "only pay for what you use", so zeroing out data for an `int` is extra work. Hence, moving an `int` actually just copies it. This is the same for some other basic types

## Destructors

**Destructors** are called when the object goes out of scope, however, this does not occur for reference objects (because they don't own anything). They are implicitly `noexcept`.

In the code above, when `vec_short` goes out of scope, destructors are called on each member.

Destructing a pointer type does nothing and can result in memory leaks. This solve this we would manually have to free memory in the destructor.

``` cpp
my_vec::~my_vec() {
    delete[] data_;
}
```

## Copy Constructors

By default, the synthesised copy constructor does a **memberwise copy**. If an attribute is a pointer, that pointer will point to the same thing as the original.

``` cpp
class my_vec {
    // Constructor
    my_vec(int size): data_{new int[size]}, size_{size}, capacity_{size} {

    // Copy constructor
    my_vec(my_vec const&) = default;
    // Copy assignment
    my_vec& operator=(my_vec const&) = default;

    // Move constructor
    my_vec(my_vec&&) noexcept = default;
    // Move assignment
    my_vec& operator=(my_vec&&) noexcept = default;

    // Destructor
    ~my_vec() = default;

    int* data_;
    int size_;
    int capacity_;
}

auto main() -> int {
    auto vec_short = my_vec(2);
    auto vec_short2 = vec_short;
}
```

Some consequences of this are that any modification to `vec_short` will also change `vec_short2`. We will perform a double free. To fix this we allocate new memory and copy the contents of the original data to the newly allocated memory.

``` cpp
my_vec::my_vec(my_vec const& orig)
: data_{new int[orig.size_]}
, size_{orig.size_}
, capacity_{orig.size_} {
    // Should work if we also define .begin() and .end(), an exercise for the reader.
    ranges::copy(orig.data_, data_);
}
```

## Copy Assignment

Assignment is the same as construction, except that there is already a constructed objected in your destination.

``` cpp
class my_vec {
    // Constructor
    my_vec(int size): data_{new int[size]}, size_{size}, capacity_{size} {

    // Copy constructor
    my_vec(my_vec const&) = default;
    // Copy assignment
    my_vec& operator=(my_vec const&) = default;

    // Move constructor
    my_vec(my_vec&&) noexcept = default;
    // Move assignment
    my_vec& operator=(my_vec&&) noexcept = default;

    // Destructor
    ~my_vec() = default;

    int* data_;
    int size_;
    int capacity_;
}

auto main() -> {
    auto vec_short = my_vec(2);
    auto vec_long = my_vec(9);
    vec_long = vec_short;
}
```

You need to clean up the destination first. The copy-and-swap idiom makes this trivial.

``` cpp
my_vec& my_vec::operator=(my_vec const& orig) {
    // create a copy of what we want to copy (i.e. orig) then
    // swap the values in the new vector and the current vector
    // the copy will contain the previous values and
    // will be destroyed when out of scope (i.e this function)
    return my_vec(orig).swap(*this);
}

my_vec& my_vec::swap(my_vec& other) {
    std::swap(data_, other.data_);
    std::swap(size_, other.size_);
    std::swap(capacity_, other.capacity_);
}

// Alternate implementation, may not be as performant.
my_vec& my_vec::operator=(my_vec const& orig) {
    my_vec copy = orig;
    std::swap(copy, *this);
    return *this;
}
```

## References

There are multiple types of references.

### Lvalue References

Lvalue references look like `T&`.  
Lvalue references to `const` look like `T& const`

A lvalue reference denotes an object whose resource cannot be reused. This includes most objects (e.g. variable, variable[0]).  
Once the lvalue reference goes out of scope, it may still be needed.

### Rvalue References

Rvalue references look like `T&&`.  

An rvalue reference denotes an object whose resources can be reused. E.g. Temporaries (`my_vec` object in `f(my_vec())`). When someone passes it to you, they don't care about it once you're done with it.

``` cpp
void f(my_vec&& x);
```

It is essentially saying "the object that x binds to is YOURS. Do whatever you like with it, no one will care anyway".  
It is like giving a copy to `f` without actually making a copy.

``` cpp
auto inner(std::string&& value) -> void {
    value[0] = 'H';
    std::cout << value << '\n';
}

auto outer(std::string&& value) -> void {
    inner(value); // This fails? Why?
    std::cout << value << '\n';
}

auto main() -> int {
    outer("hello"); // This works fine.
    auto s = std::string("hello");
    outer(s); // This fails because s is an lvalue.
}
```

An rvalue reference formal parameter means that the value was disposable from the caller of the function.  
If `outer` modified the value, no one would care/notice since the caller (`main`) has promised that it won't be used anymore.  
If `inner` modified the value, `outer` would care/notice since the caller (`outer`) has never made such a promise. An rvalue reference parameter is an lvalue inside the function.

### Reference Binding

Note that:

* `T&&` binds to rvalues only
* `T&` binds to lvalues only
* **`T const&` binds to BOTH lvalues and rvalues**

## std::move

`std::move` looks something like this:

``` cpp
T&& move(T& value) {
    return static_cast<T&&>(value);
}
```

It converts its argument into an rvalue. Calling `move` is essentially saying "I don't care about this anymore". All this does is allow the compiler to use rvalue reference overloads.

``` cpp
auto inner(std::string&& value) -> void {
    value[0] = 'H';
    std::cout << value << '\n';
}

auto outer(std::string&& value) -> void {
    inner(std::move(value));
    // Value is now in a valid but unspecified state.
    // Although this isn't a compiler error, this is bad code.
    // Don't access variables that were moved from, except to reconstruct them.
    std::cout << value << '\n';
}

auto main() -> int {
    outer("hello"); // This works fine.
    auto s = std::string("hello");
    outer(s); // This fails because s is an lvalue.
}
```

## Moving Objects

Always declare your moves as `noexcept`. Failing to do so can make your code slower.

``` cpp
class T {
    T(T&&) noexcept;
    T& operator=(T&&) noexcept;
};
```

Unless otherwise specified, objects that have been moved from are in a value but unspecified state.

Moving is an optimisation of copying. The only difference is that when moving, the moved-from object is mutable. Not all types can take advantage of this.  
If moving an `int`, mutating the moved-from `int` is extra work.
If moving a `vector`, mutating the moved-from `vector` potentially saves a lot of work.

Moved-from objects must be placed in a valid state.  
Moved-from containers **usually** contain the default-constructed value.  
Moved-from types that are cheap to copy are **usually** unmodified.  
Although this is the only requirement, individual types may add their own constraints

Compiler generated move constructor/assignment performs memberwise moves

### Move Constructor

``` cpp
class my_vec {
    // Constructor
    my_vec(int size): data_{new int[size]}, size_{size}, capacity_{size} {

    // Copy constructor
    my_vec(my_vec const&) = default;
    // Copy assignment
    my_vec& operator=(my_vec const&) = default;

    // Move constructor
    my_vec(my_vec&&) noexcept;
    // Move assignment
    my_vec& operator=(my_vec&&) noexcept = default;

    // Destructor
    ~my_vec() = default;

    int* data_;
    int size_;
    int capacity_;
}

my_vec::my_vec(my_vec&& orig) noexcept
: data_{std::exchange(orig.data_, nullptr)}
, size_{std::exchange(orig.size_, 0)}
, capacity_{std::exchange(orig.size_, 0)} {

auto main() -> int {
    auto vec_short = my_vec(2);
    auto vec_short2 = std::move(vec_short);
}
```

### Move Assignment

``` cpp
class my_vec {
    // Constructor
    my_vec(int size): data_{new int[size]}, size_{size}, capacity_{size} {

    // Copy constructor
    my_vec(my_vec const&) = default;
    // Copy assignment
    my_vec& operator=(my_vec const&) = default;

    // Move constructor
    my_vec(my_vec&&) noexcept = default;
    // Move assignment
    my_vec& operator=(my_vec&&) noexcept;

    // Destructor
    ~my_vec() = default;

    int* data_;
    int size_;
    int capacity_;
}

my_vec& my_vec::operator=(my_vec&& orig) noexcept {
    // The easiest way to write a move assignment is generally to do
    // memberwise swaps, then clean up the orig object.
    // Doing so may mean some redundant code, but it means you don't
    // need to deal with mixed state between objects.
    ranges::swap(data_, orig.data_);
    ranges::swap(size_, orig.size_);
    ranges::swap(capacity_, orig.capacity_);
    // The following line may or may not be necessary, depending on
    // if you decide to add additional constraints to your moved-from object.
    orig.clear();
    return *this;
}

void my_vec::clear() noexcept {
    delete[] data_;
    data_ = nullptr;
    size_ = 0;
    capacity = 0;
}

auto main() -> int {
    auto vec_short = my_vec(2);
    auto vec_long = my_vec(9);
    vec_long = std::move(vec_short);
}
```

## Passing References To Be Copied

Consider the following code:

``` cpp
struct S {
    // modernize-pass-by-value error here
    S(std::string const& x) : x_{x} {}
    std::string x_;
};

auto str = std::string("hello world");
auto a = S(str);
auto b = S(std::move(str));
```

When we construct `a`:

* we create a `const` reference
* we copy it to `x_`

When we construct `b`:

* we create a `const` reference
* since we have a `const` reference, we cannot move from it
* we copy it into `x_`

Now consider the following code:

``` cpp
struct S {
    // modernize-pass-by-value error no longer here
    S(std::string x) : x_{std::move(x)} {}
    std::string x_;
};

auto str = std::string("hello world");
auto a = S(str);
auto b = S(std::move(str));
```

When we construct `a`:

* we create a temporary object by copying
* we then move that temporary copy

When we construct `b`:

* we create a temporary object by moving (since the argument is an rvalue)
* we then move that temporary copy

It turns out that moving from temporary objects is something the compiler can trivially optimise. This should be the same performance for lvalues, but allowing move instead of copying for rvalues

## Deleted Copies and Moves

### Explicitly Deleted Copies and Movees

We may not want a types to be copyable/moveable. If so we can declare `fn() = delete`

``` cpp
class T {
    T(const T&) = delete;
    T(T&&) = delete;
    T& operator=(const T&) = delete;
    T& operator=(T&&) = delete;
};
```

### Implicitly Deleted Copies and Moves

Under certain conditions, the compiler will not generate copies and moves.

The implicitly defined copy constructor calls the copy constructor memberwise. If one of its members doesn't have a copy constructor, the compiler can't generate one for you. The same applies for copy assignment, move constructor, move assignment.

Under certain conditions, the compiler will not ***automatically*** generate copy/move assignment/constructors. E.g. If you have manually defined a destructor, the copy constructor isn't generated.

If you define one of the rule of five, you should explicitly delete, default or define all five. If the default behaviour isn't sufficient for one of them, it likely isn't sufficient for others.  
Explicitly doing this tells the reader of your code that you have carefully considered this. This also means you don't need to remember all the rules about "if I write X, then Y is generated"

## Resource Acquisition Is Initialisation

**Resource Acquisition Is Initialisation (RAII)** is a concept where we encapsulate resources inside objects; we acquire the resource in the constructor, and release the resource in the destructor.

Every resource should be owned by either

* another resource (e.g. [smart pointer](TODO), data member)
* the stack
* a nameless temporary variable

## Object Lifetimes

To create safe object lifetimes in C++, we always attach the lifetime of one object to that of something else.

Names objects:

* a **variable** in a **function** is tied to its *scope*
* a **data member** is tied to the lifetime of the **class instance**
* an **element in a `std::vector`** is tied to the lifetime of the vector

Unnamed objects:

* A **heap object** should be tied to the lifetime of whatever object created it
* Examples of bad programming practice:
    * an *owning raw pointer* is tied to nothing
    * A *C-tyle array* is tied to nothing

Recommend viewing: [Leak freedom in C++... by Default](https://www.youtube.com/watch?v=JfmTagWcqoE)

### Object Lifetime with References

We need to be very careful when returning references.  
The ***object must always outlive the reference***  
This is undefined behaviour -  if you're unlucky, you code might even work.

Lesson: **do not return references to variables local to the function returning**

``` cpp
auto okay(int& i) -> int& {
    // pass in an int reference and return an int reference
    // only binds to lvalues
    return i;
}

auto okay(int& i) -> int const& {
    // pass in an int reference and make it not modifiable
    return i;
}

auto questionable(int const& i) -> int const& {
    // can bind to r values
    // if you pass in a temporary, you will return a reference to it
    // but the temporary when be destroyed on returns (out of scope)
    return i;
}

auto not_okay(int i) -> int& {
    // pass in a copy, return a reference
    // copy will be destroyed on return (out of scope)
    return i;
}

auto not_okay() -> int& {
    // reference to local variable returned
    // will be destroyed on return (out of scope)
    auto i = 0;
    return i;
}
```
