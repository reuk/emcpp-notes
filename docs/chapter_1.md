# Intro

The idea is just to provide some motivation to read the chapter, and internalise it a bit. By
talking about the chapter contents, hopefully they’ll stick a bit better… for that reason, it’d be
good if everyone can ask a question or make a point (even if you haven’t read the chapter!).

- How much reading is reasonable for a fortnight?
- Is there anything that people especially want to get out of these sessions?

# Type deduction

This is basically just going to be the 'Things to Remember' from the chapter.

## Template type deduction

During template type deduction, arguments that are references are deduced as non-references
(although the arguments themselves retain their reference-ness).

```
template <typename T> void f (T& arg) {}

auto x = int { 0 };       f (x);  // T is int
const auto cx = x;        f (cx); // T is const int
const auto& rx = x;       f (rx); // T is const int
```

Forwarding references treat lvalue references specially. For lvalue references, T and the parameter
itself retain their ref-ness. For rvalues, the normal rules apply, so T's ref-ness is removed.

```
template <typename T> void f (T&& arg) {}

auto x = int { 0 };     f (x);  // T is int&
const auto cx = x;      f (cx); // T is const int&
const auto& rx = x;     f (rx); // T is const int&
                        f (0);  // T is int
```

By-value parameters will have their ref-ness, const-ness and volatile-ness removed.

```
template <typename T> void f (T arg) {}
```

Arrays and function arguments decay to pointers, unless they're used to initialise references.

## Auto type deduction

Auto type deduction is the same as template type deduction, with the exception of
std::initialiser_list. If you're going to use brace initialisation with auto, it's important to put
the type on the rhs!

```
auto x = { 0 }; // initializer_list, oh no!
```

auto parameters or return types use the template rules. They won't deduce an initialiser_list type
in those cases.

## Understand decltype

Decltype yields the type of a variable or expression. There's a gotcha where wrapping a variable in
parens means it's no longer a name expression, which might affect its ref-ness.

```
int x = 0;
decltype ((x)) // int&
```

`decltype(auto)` will apply the decltype deduction rules when deducing the type.  That is, it will
retain the constenss and ref-ness of returned values. Be careful not to return lvalue references to
temporaries when using `decltype(auto)`.

```
return (some_expression)
```

## Know how to debug types

Deduced types can be viewed by forcing a template instantiation error involving the type in
question, or by using an IDE (support seems questionable though) or boost TypeIndex.

```
template <typename T> struct Error;
Error<decltype(some_expression)> e;
```

## Talking points

Is a knowledge of prvalues, xvalues, glvalues, rvalues etc. useful? I know they exist, but I don't
really know the definitions, and I'm not sure whether I should.

Is it helpful to specifically say `auto*` when initialising a variable with the result of a function
which returns a pointer? If I was writing a template function which expected to only operate on
pointers, I probably wouldn't put a pointer in the parameter list, but maybe that's a mistake.
