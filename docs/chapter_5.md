# Rvalue references, move semantics, and perfect forwarding

```
void f (Widget&& w); // w is an lvalue
```

The rvalue ref is used for selecting a suitable overload of the function.
Inside the function, the parameter is always an lvalue (because it's named).

## Understand `std::move` and `std::forward`

These are just shorthands for specific casts.

- `move` casts to rvalue unconditionally
- `forward` casts to rvalue under certain conditions

Neither function does anything other than casting. They both compile to nothing.

Casting to rvalue using `move` will *usually* cause the compiler to select
an overload of the function being called which takes an rvalue argument (if
available). However, this doesn't always work.

Calling `move` on a const lvalue will result in a const rvalue, which will
match const lvalue overloads better than non-const rvalue overloads, probably
resulting in an unexpected copy-constructor call.

```
Foo (const std::string str)
    : value { std::move (str) } {} // oops, copy constructor call!
...
std::string value;
```

Don't declare objects `const` if you want to move from them (including objects
returned from functions).

`forward` is a conditional cast:

```
void process (const Widget&);
void process (Widget&&);

template <typename T>
void logAndProcess (T&& t) // here t is an lvalue, but T may be an lvalue or rvalue ref type
{
    process (t); // always selects the lvalue overload
    process (std::move (t)); // always selects the rvalue overload
    process (std::forward<T> (t)); // preserves the type of t by reconstructing it from T
}
```

Although `forward` can be used in more situations, it requires providing a
template argument, which is a potential source of error, so `move` should be
preferred in situations where an unconditional rvalue cast is required.

### Guidelines

- Don't declare objects `const` if you want to move from them (including objects
  returned from functions).

- Use `move` in situations where you need an unconditional rvalue cast

- Use `forward` if you want to preserve the lvalue/rvalue-ness of the argument

## Distinguish universal references from rvalue references

When using `&&` in the context of template parameters in the form `T&&` or
`auto&&` deduction, we are declaring a forwarding reference which can bind to
either rvalues or lvalues.

In other contexts (`ConcreteType&&`, `std::vector<T>&&`, `const T&&`), the
expression is a plain rvalue reference, which can only bind to rvalues.

In particular, in a template class, the following is a plain rvalue reference,
because no type deduction is happening at the point that the function is
instantiated:

```
template <typename T>
class MyClass
{
  void push_back (T&&); // No type deduction, not a forwarding reference!

  template <typename... Ts>
  void emplace_back (Ts&&...); // Types are deduced, forwarding references
};
```

`auto&&` universal references are useful for declaring forwarding reference
arguments to generic lambdas:

```
auto lambda = [] (auto&& arg) // This is a forwarding reference
{
  process (std::forward<decltype (arg)> (arg));
};
```

### Guidelines

- Ensure that you use the correct form (`T&&` or `auto&&`) for forwarding
  references.

- ???

## Use `std::move` on rvalue references, `std::forward` on forwarding references

An rvalue reference can only bind to an rvalue, which is a candidate for
moving. Rvalue references should be unconditionally cast to rvalues when
forwarding them to other functions:

```
void sink (ConcreteType&& ct) // can only be called on rvalues
{
  collection.add (std::move (ct));
}
```

A forwarding reference can bind to both lvalues and rvalues, but unexpectedly
moving-from an lvalue would be surprising! Forwarding references should be
conditionally cast to rvalues (using `std::forward`), depending on whether or
not the reference is bound to an rvalue.

```
// Imagine code like this:
ConcreteType ct;
process (ct);             // unsafe to move
process (ConcreteType{}); // safe to move

template <typename T>
void process (T&& t) // can be called on lvalues or rvalues
{
  // BAD, the user won't expect lvalues to be moved-from
  collection.add (std::move (t));
}

template <typename T>
void process (T&& t) // can be called on lvalues or rvalues
{
  // GOOD, `t` will only be moved-from if it the argument was an rvalue
  collection.add (std::forward<T> (t));
}
```

In cases where type conversions may take place (e.g. converting from a string
literal to a `string`, or from a lambda to a `function`), using forwarding
references will likely be cheaper than explicitly naming types and materialising
temporaries:

```
// BAD Passing a plain lambda here will create a temporary `std::function`
void setCallback (std::function<void()>&& f)
{
  // This line will invoke the move assignment operator
  callback = std::move (f);
  // The destructor of the temporary will be called here
}

// GOOD No temporary generated
template <typename Callback>
void setCallback (Callback&& f)
{
  // This line directly invokes the assignment operator which takes any
  // function-like argument
  callback = std::forward<Callback> (f);
  // No temporary to destroy here
}
```

If a forwarded argument is used multiple times in a function, ensure that only
the final use of the argument is `move`d or `forward`ed.

```
template <typename Contents>
void setContents (Contents&& c)
{
  hash = computeHash (c); // Does not consume c
  contents = std::forward<Contents> (c); // Consumes c
}
```

The compiler will employ Return Value Optimisation (RVO) in cases where:

- The type of a local object being returned is the same as the function return
  type (no type conversions).
- A local object is returned.

In such cases, it is harmful to explicitly move/forward the result of a
function:

```
Widget createWidget()
{
  Widget w;
  ...
  // BAD, disables RVO because this returns a reference to a local object,
  // not the object itself
  return std::move (w);
}
```

The compiler is required to treat the object being returned as an rvalue
*anyway* if the conditions for RVO are met.

One case where the conditions for RVO may not be met is modifying and then
returning an argument:

```
template <typename T>
T process (T&& t)
{
  t.adjust();
  // RVO cannot be applied because `t` is not a local object, we need to
  // explicitly avoid a copy here in the case that `t` was bound to an rvalue
  return std::forward<T> (t);
}
```

### Guidelines

- Apply `std::move` to the final use of rvalue references
- Apply `std::forward` to the final use of forwarding references
- Never `return forward()` or `return move()` when returning objects that would
  otherwise be eligible for RVO (that is, local objects with a type matching
  the function return type)

## Avoid overloading on forwarding references

Functions taking forwarding references will match almost any kind of argument.
If one overload of a function has a forwarding reference parameter, it will
generally be a better match than other overloads which would require implicit
conversions, even widening conversions (e.g. `short` to `int`).

Constructors taking forwarding references are especially dangerous, because
such a constructor may be a better match than copy/move constructors, leading
to surprising compilation failures when attempting to copy/move instances.

### Guidelines

- Don't write overload sets where one overload has a forwarding reference
  parameter.
  - Particularly, prefer not to write constructors which take a forwarding
    reference argument.

## Familiarise yourself with alternatives to overloading on universal references

Some options:

- Use different names, don't overload
  - Doesn't work for constructors!
  - Doesn't work well for generic code
- Pass by const ref
  - Inefficient when sinking arguments
- Pass by value
  - If you know you'll always consume the argument it's a good option
- Use tag dispatch (or `if constexpr`)
  - An outer function which takes a forwarding reference, which dispatches
    to inner functions that are differentiated using tag types or plain
    if statements:
    ```
    template <typename T>
    void dispatch (T&& t)
    {
      // wow this is way nicer than tag dispatch
      if constexpr (std::is_integral_v<std::remove_reference_t<T>>)
      {
        // do something for integral arguments
        integralSink (std::forward<T> (t);
      }
      else
      {
        // do something different
        otherSink (std::forward<T> (t);
      }
    }
    ```
  - Still doesn't work for constructors, because of the copy/move constructor
    issue!
- Use SFINAE, maybe via `enable_if`
  - This does actually work for constructors, but it's kind of complicated
    - In particular, the `enable_if` expression will need to disable the
      template constructor in cases where the template constructor would end up
      incorrectly being a better match than the copy/move constructors (or any
      other constructor that should be preferred).
  - Take a look at the `APVTS::ParameterLayout` constructors for a real-world
    usage

Note that the tag/if-constexpr/sfinae approaches could lead to some really
incomprehensible error messages, so they're probably only worth it in
super-generic or performance-critical cases. In those cases, some well-placed
`static_asserts` could help make error messages a bit more comprehensible.

### Guidelines

- Keep simple things simple: avoid overloading functions taking forwarding
  references where possible

- Prefer `if constexpr` to tag dispatch

- Only use universal reference parameters if the efficiency/genericity
  advantages outweigh the usability disadvantages

## Understand reference collapsing

In some contexts, the compiler might end up producing a reference to a
reference:

```
template <typename T>
void func (T&& t);

Widget w;
func (w); // pass an lvalue

// T is deduced to be `Widget&`, giving this signature
void func (Widget& && t);
```

> If either reference is an lvalue reference, the result is an lvalue ref.
  Otherwise, the result is an rvalue ref.

So the example above would collapse to:

```
void func (Widget& t);
```

Remember that templates with forwarding reference parameters will deduce the
template type to be an lvalue ref if the argument is an lvalue, or a *plain
value* (non-ref type) if the argument is an rvalue.

`std::forward` works by checking the type of its template argument. It will
only cast its argument to an rvalue if the template argument has a
non-reference type. Its implementation relies on reference collapsing.

template <typename T>
auto forward (std::remove_reference_t<T> arg)
{
    return static_cast<T&&> (arg);
}

Reference collapsing happens in a few contexts:

- Template type deduction
- `auto` type deduction
- `typedef` evaluation
- `decltype` usage

### Guidelines

- ??? Not sure this is really too important day-to-day?

## Assume that move operations are not present, not cheap, and not used

Not all types support move operations (e.g. types with explicit copy ops or
destructors that haven't been updated with move ops).

Objects that are moveable may still be expensive to move. Moving a `std::array`
requires moving each individual element, so it will be expensive for larger
arrays.

Sometimes an object's move operations cannot be used because they might
throw, violating some exception safety guarantee.

*In generic code*, you cannot be certain that all types involved will be
noexcept-moveable, so care should be taken that code performs acceptably
whether or not moveable types are used. This might involve use of perfect
forwarding, switching implementations depending on move-noexceptness using `if
constexpr` etc.

For code that uses concrete types, this is less of a concern, because it's
easy to find out whether those types offer cheap move semantics.

### Guidelines

- In generic code, don't assume that move operations are present and cheap.

## Familiarise yourself with perfect forwarding failure cases

The goal of perfect forwarding is to pass a function's arguments to another
function, preserving their types, rvalue/lvalue-ness, and const/volatile-ness.
This is achieved using forwarding references:

```
template <typename... Ts>
void fwd (Ts&&... ts)
{
  inner (std::forward<Ts> (ts)...);
}
```

Perfect forwarding fails when:

- Compilers cannot deduce a type for one of the outer function's parameters
- Compilers deduce the 'wrong' type for the outer function's parameters, so
  that either
  - Instantiating the function with those types creates code that won't
    compile, or
  - A call to the inner function using the outer function's derived types would
    behave differently to calling the inner function directly.

Unfortunately, these things do happen...

**Braced initialisers**: supposing 'inner' has the signature
```
void inner (const std::vector<int>&);
```
the expression `fwd ({1, 2, 3})` will fail to compile, because the compiler is
unable to deduce a type for raw braced initialisers when they are used as a
template function argument. (This happens a surprising amount when converting
code from using raw constructor calls to using `make_unique`.)

**0 or NULL as null pointers**: 0/NULL are integer, not pointer types, so they
cannot be perfect-forwarded as pointer arguments. Use `nullptr` instead!

**Declaration-only integral static const data members**: This one really is an
edge case. In a class like this:
```
class Widget {
public:
  static const int num = 4;
};
```
Uses of `Widget::num` will be 'const-propagated'; the compiler will insert the
integral value directly each time it is used, which means that `num` doesn't
need to have an address of its own.

If we were to call `inner (Widget::num)`, this would be transformed to
`inner (4)`. However, if we call `fwd (Widget::num)`, the code will compile but
it won't link (on some implementations)!  `fwd` passes `num` by reference,
which means that `num` needs to exist at an address. In this case, we would
also need a single definition of `num` in addition to the declaration in the
class.
```
const int Widget::num;
```

**Overloaded function names and template names**: If you pass the name of a
function which is part of an overload set to `fwd`, the compiler won't be able
to deduce its type (because there are multiple functions with different types
which share that name). The same goes if passing a template function to `fwd`
(the compiler can't deduce the template arguments of the passed function).

**Bitfields**: A non-const reference shall not be bound to a bit-field (because
individual bits cannot be directly addressed on hardware).
```
struct Options {
  std::uint8_t foo: 1,
               bar: 1,
               baz: 1;
};
...
Options options;
fwd (options.foo); // error!
fwd (static_cast<std::uint8_t> (options.foo)); // works, because it copies the field
```

### Guidelines

- Be aware that the following kinds of arguments can lead to perfect forwarding
  failures:
  - Braced initialisers
  - 0/NULL
  - Declaration-only integral `static const` data members
  - template/overloaded function names
  - bitfields

