# Moving to Modern C++

## Distinguish between () and {} when creating objects

C++11 introduces *uniform initialisation* which uses {}. Braced initialisation gives us new ways to
initialise objects, such as constructing vectors directly with the desired contents, and specifying
initial values for class data members.

### Good bits

Braced initialisation disallows narrowing conversions, whereas parens do not. Not that much
difference if you already use compiler flags to disallow narrowing conversions.

```
double x { 0.0 };
int y { x }; // error!
```

Braced init allows us to work around the 'most vexing parse'.

```
Widget w(); // looks like we're declaring a function
Widget w{}; // looks like a default constructor call
```

### Bad bits

Using braces will always prefer a constructor taking an `initializer_list`, if one is available. Be
careful around vectors and other std collections! Note that 'empty braces' still means a normal
default constructor call.

Also, be careful when adding/removing overloads which take initializer_lists, as you might silently
break dependent code.

If you're writing a variadic `make` function, consider documenting whether you use parens or braces
to construct the inner object, as the two may have considerably different semantics.

## Prefer nullptr to 0 and NULL

nullptr is a pointer type (kinda, it's a nullptr_t). 0 and NULL are integer types.

If you use 0 or NULL, you're in danger of calling overloads taking integral types, where you
probably meant to call an overload taking a ptr type.

nullptr never converts to an integer, so you're in no danger of accidentally calling an overload
taking an integral argument.

Even so, you should still avoid overloading ints and pointers!

## Prefer alias declarations to typedefs

`using` looks much nicer than `typedef` when aliasing function types.

`using` can be templated, which is much more concise than writing template metafunctions.
```
template <typename T>
using MyList = std::list<T, MyAlloc<T>>;
// vs
template <typename T>
struct MyList {
    typedef std::list<T, MyAlloc<T>> type;
};
```

Using these template aliases is nicer too, because you don't need to access inner types, or use
`typename`:
```
auto x = MyList<T>{};
// vs
auto x = typename MyList<T>::type{};
```

The stdlib has a few new metafunctions with aliases for type conversions/operations in the
`<type_traits>` header.

## Prefer scoped enums to unscoped enums

Scoped enums (enum classes) reduce namespace pollution, because the enumerator names don't leak
into the enclosing namespace, unlike unscoped enums.

Scoped enums are more strongly typed, and don't implicitly convert to integers. They can also be
forward-declared.

## Prefer deleted functions to private undefined ones

Deleted functions warn you during compilation if the function is used. Preferable to other
approaches (making private and leaving undefined), which tend to warn during link instead.

Deleting functions works everywhere (not just special member functions). e.g. you can delete
`operator new` on a class to disallow heap allocation for that type...

## Declare overriding functions `override`

Lots of conditions must be met in order for a derived class function to
override (*not overload!*) a base class function:

- The base class function must be virtual
- The base + derived function names must be the same
- The parameter types must be identical
- The constness of the functions must be identical
- The return types and exception specs must be compatible
  - Overriding functions are allowed to get 'more noexcept'
    ```
    // Error
    struct Base { virtual void foo() noexcept = 0; };
    struct Derived : Base { void foo() override {} };

    // OK
    struct Base { virtual void foo() = 0; };
    struct Derived : Base { void foo() noexcept override {} };
    ```
  - Return types may be covariant
    ```
    // OK
    struct Base { virtual Base* clone() const; };
    struct Derived : Base { Derived* clone() const override; };
    ```
- The *reference qualifiers* must be identical
  - A feature that allows member functions to be restricted so that they can
    only be called on lvalues or rvalues. Works on non-virtual functions too!
    ```
    struct Base {
      void foo() &;   // May only be called on an lvalue
      void foo() &&;  // May only be called on an rvalue
    };
    ```
  - Similar to the way that non-`const` member functions cannot be called on
    a `const` instance
  - Mostly useful when transferring ownership of a data member out of a
    temporary object

It's very easy to change one of these properties in a base class e.g. while
refactoring, leading to breakage when derived class functions no longer
override base class functions.

Declaring a derived class function `override` will make the compiler check that
the function is in fact overriding a base class function.

### New rules

No `override` necessary on overriding destructors

## Prefer `const_iterator`s to `iterator`s

The stdlib used to have poor const_iterator support. It was difficult to
obtain const_iterators from mutable containers, and most container member
functions only accepted iterator arguments instead of const_iterators.

`iterator` and `const_iterator` are fundamentally different types, and it's
generally unsafe to cast between them.

In modern C++, there's a few ways to get const iterators:
```
std::vector<int> values;
auto a = values.cbegin();
auto b = std::cbegin (values);
auto c = std::as_const (values).begin();
```

In generic code, prefer non-member versions of `begin`, `end` etc. over member
varieties, as more types of collection are able to support the non-member
versions (e.g. C arrays).

For bonus points, you can do this, which will use specialised versions of
`cbegin/cend` from the container's type's namespace if available, and will
fall back to the `std` functions if no custom versions are present.
```
using std::cbegin;
using std::cend;
auto it = std::find(cbegin(container), cend(container), value);
```

### New rules

Use functions accepting ranges rather than iterator pairs where possible.
Prefer non-member utility functions over member functions in generic code.
Using member functions is fine if you know the concrete type!

## Declare functions `noexcept` if they won't emit exceptions

Old-style exception specifications are deprecated in C++11. They're replaced by
`noexcept`.

`noexcept` becomes part of a function's interface, a bit like `const`. Calling
code is able to query the noexceptness of an expression in a similar way to
decltype:

```
void may_throw();
void no_throw() noexcept;

static_assert(! noexcept(may_throw()));
static_assert(noexcept(no_throw()));
```

If a `noexcept` function emits an exception, the program is terminated. The
stack is not guaranteed to be unwound before termination, which gives the
optimiser fewer constraints, which means generated code will likely be faster.

Some standard library collections (like vector) offer a 'strong exception
safety' guarantee on some member functions, so that the state of the collection
remains valid and known in the event of a thrown exception. If move operations
on the container element type are not `noexcept`, the collection may decide to
copy rather than move elements in order to maintain the exception guarantee.
This will result in slow code with lots of copies! It's critical that custom
move operations are `noexcept` whenever possible for this reason.

Note that removing `noexcept` from a function may break client code that
depends on the exception specification of the function. It's important to be
certain that the function will never ever need to throw an exception.

### New rules

If you write a custom move (or swap) op, make it `noexcept` if at all possible.
If a function doesn't (and never could) throw an exception, mark it noexcept.
If you're writing realtime-compatible functions, aim to make them noexcept.
Consider some keyword/macro for realtime-compatible functions.

## Use `constexpr` wherever possible

`constexpr` values are values which are known during compilation. They are
implicitly `const`, and can be used in situations where a plain `const`
variable wouldn't be sufficient (e.g. as an array size).

`constexpr` functions produce compile-time constants when they are called with
compile-time constants. If they're called with values that are only known at
runtime, the function will produce runtime values.

User-defined types can also be used in `constexpr` contexts, if they have a
`constexpr` constructor. In C++14, even non-const member functions can be
`constexpr` (examples of this in the iterators module!).

By using `constexpr` wherever possible, you maximise the range of situations in
which your objects and functions may be used. Similar to `noexcept`, clients
may come to depend on the constexprness of your functions, so think carefully!

### New rules

In practice, the implementations of constexpr/non-constexpr functions might need to be considerably
different in order to get the required performance of each scenario. Apply careful thought!

Prefer constexpr variables to other compile-time constructs (enums, defines, etc.)

constexpr variables inside a function should probably be `static` (?)

## Make `const` member functions thread safe

Having two threads perform a read operation (calling a `const` member function)
without synchronisation should ideally be safe. However, if said const member
is lazily evaluated and does in fact modify a `mutable` member, there will be a
data race if two threads call the function at the same time. In such
situations, the race might be fixed using normal thread safety techniques
(e.g. using atomics, adding+locking a mutex).

Obviously, be careful if you have multiple data members that must be updated
'simultaneously'. Using atomics is unlikely to work in this situation, as
atomic updates may be interleaved in undesirable ways, duplicating work or
breaking object invariants.

If you can *guarantee* that a const member will never be called in a concurrent
context, you likely don't need synchronisation.

If your const member doesn't modify the state of the object, you likely don't
need synchronisation.

### New rules

Be careful with mutable data members, make sure you have synchronisation if you're
modifying mutable members from const functions.

If for some reason you've elected to make your const member thread-unsafe, make sure
that's clear!

## Understand special member function generation

That's the
- Default constructor
- Destructor
- Copy constructor
- Copy assignment operator
- Move constructor
- Move assignment operator

These functions are only generated if needed. They are implicitly public and
`inline`. They're nonvirtual, unless the generated function is a destructor
for a class whose base has a virtual destructor.

Declaring either of the move ops will prevent the compiler from generating the
other, so declare both or neither.

Move ops won't be generated if the class declares a copy op. Similarly, copy
ops won't be generated if the class declares a move op.

Move ops won't be generated if the class declares a destructor. That means that
if you add a virtual (defaulted) destructor on a base class, that class will
no longer be moveable.

So, it's normally best to declare either *all* or *none* of special
members. Remember to declare the move ops noexcept!

### New rules

Write lots of default members hooooray
If you're writing a move-only type, it's fine not to declare copy ops
If you're writing a listener base class, the type shouldn't be moveable (consider explicitly
deleting the move ops to show intent)
