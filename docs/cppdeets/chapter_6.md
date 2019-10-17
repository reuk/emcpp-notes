# Standard Attributes

## Why attributes?

Attributes are a standard way of supplying extra information to the compiler that can't be conveyed
using the core language. Some attributes are specified in the standard, but some platforms/compilers
have their own custom attributes. Attributes cannot be user-defined.

Before the standard syntax was introduced, clang/gcc would specify attributes using
`__attribute__((attr_name))`. MSVC used `__declspec(attr_name)`.

After C++11, attributes can be specified in double square brackets: `[[attr]]` or
`[[namespace::attr]]`.

## Standardised attributes

### `[[noreturn]]`

Control flow will never return to the caller of a function marked `[[noreturn]]`, so the compiler
won't warn about failure to return a value on such a path (for example).

### `[[carries_dependency]]`

Indicates that the dependency chain in release-consume `std::memory_order` propagates in and out
of the function, allowing the compiler to skip unnecessary memory fences.

### `[[deprecated]]` and `[[deprecated("reason")]]`

Code marked with this attribute will raise an appropriate warning at build time.

### `[[fallthrough]]`

Indicates that fallthrough in a `switch` is intentional. Better than a comment, but try to avoid
fallthrough completely wherever possible.

### `[[maybe_unused]]`

Suppresses compiler warnings about unused entities, much like `juce::ignoreUnused`. Nicer than
function/cast-based approaches because it can be applied directly to variable declarations and
parameters, so there's no need for a whole separate 'ignore' statement. Most commonly used with
build-mode specific code like assertions, possibly `if constexpr`.

### `[[nodiscard]]`

Can be applied to a function or type to ensure that the user does something with a returned value.
Useful for RAII types like `unique_lock`, `std::future`s returned from `std::async`, `operator new`.

If you really really do want to ignore a `[[nodiscard]]` value, mark the value `[[maybe_unused]]`.

Code gets very noisy if every function that returns a value is marked `[[nodiscard]]`, so it's
probably best to reserve this attribute for cases where discarding the value would be an outright
logic error. It's probably not necessary to use it in cases where discarding the value would just
mean doing extra work at runtime, but wouldn't affect the correctness of the program.

## C++17 Enhancements

Attributes can be applied to namespaces and enums in C++17, which wasn't previously possible.
Also, compilers are guaranteed not to emit a hard error for attributes that they don't understand.
