# Language Clarification

## Stricter Expression Evaluation Order

In C++14, the following code can leak if an exception is thrown at a specific point:

```
callFunction (std::unique_ptr<T> (new T), std::unique_ptr<U> (new U));
```

The compiler may choose (in C++14) to order the evaluation of the expression like so:
1. `new T`
2. `new U`
3. `std::unique_ptr<T> (...)`
4. `std::unique_ptr<U> (...)`

If the constructor of `U` were to throw, the new `T` instance would leak, because it wouldn't
yet be captured in a RAII wrapper. To counteract this surprising behaviour, it was recommended
to use `make_` functions to ensure that all resources were immediately managed by RAII wrappers.
However, for `unique_ptr`s with custom destructors (for example) this wasn't always easy.

As of C++17, any function parameter is fully evaluated before the next is started. Complex nested
expressions can't be interleaved as they may have been previously.

```
foo (a (b), c (d));
```

Here, `a (b)` and `c (d)` will always be evaluated in one go, although the relative ordering of
these full expressions is unspecified.

For chained function calls, expressions will always be evaluated left-to-right, where everything
left of a `.` will be evaluated before evaluating the rhs.

```
a (b).c (d);
```

Now, `a (b)` will always be computed before `d`.

Sequencing on other operators has also been more strictly specified, with user-defined comma
operators, `&&`, `||` now offering left-to-right sequencing which is particularly useful when
writing fold expressions (another new C++17 feature).

As always, if the sequencing of an expression is important/unclear, split the expression into
multiple statements with explicit sequencing.

### Guaranteed Copy Elision

Guaranteeing copy elision is useful for:
- Returning non-moveable types from functions, and avoiding output parameters
- Improving performance

Return Value Optimisation was already allowed (but not guaranteed) under certain circumstances in
C++14, when
- The type being returned exactly matched the function's return type.
- The object being returned was a local object (not global, or passed in as an argument).

C++17 now guarantees copy elision in some circumstances. Now, even types without copy/move
constructors can be returned from functions. Copy elision happens when:
- An object is initialised from a prvalue: `auto t = T{};` (facilitates Almost-Always-Auto style)
- A function call returns a prvalue.

A prvalue is an expression whose evaluation initialises an object, where the expression itself has
no name and its address cannot be taken.

Note that Named Return Value Optimsation is still not guaranteed, so code that relies on copy
elision for named locals objects may not be portable. However, eliding copies for prvalues *is*
portable.

### Dynamic Memory Allocation for Over-Aligned Data

Prior to C++17, `new` wasn't guaranteed to honor over-alignment for allocated types (memory was only
guaranteed to have the minimum necessary alignment for the type, so that all instances of
fundamental types making up the object had the correct alignment).

If your type is marked with `alignas` then `operator new` will now automatically use the requested
alignment for the type.

```
#include <cstdint>
#include <vector>

int main()
{
    class alignas (128) Overaligned {};
    const auto vec = std::vector<Overaligned> (100);
    return reinterpret_cast<std::intptr_t> (vec.data()) % alignof (Overaligned);
}
```

The above now builds to something like:
```
main:                                   # @main
        pushq   %rbx
        movl    $12800, %edi            # imm = 0x3200
        movl    $128, %esi
        callq   operator new(unsigned long, std::align_val_t) # oh hey, an alignment parameter!
        movl    %eax, %ebx
        andl    $127, %ebx
        testq   %rax, %rax
        je      .LBB0_2
        movl    $128, %esi
        movq    %rax, %rdi
        callq   operator delete(void*, std::align_val_t)
.LBB0_2:
        movl    %ebx, %eax
        popq    %rbx
        retq
```

### Exception Specifications in the Type System

Pointers to `noexcept` functions can be converted to non-`noexcept` function pointers, but the
reverse is not allowed. Virtually overriding a `noexcept` function is only possible with another
`noexcept` function, and `noexcept` functions are allowed to override non-`noexcept` functions too.

Apparently, having this information at the type level allows the compiler to generate faster code,
although I'm not sure why. Just remember to mark functions `noexcept` where possible!

Note that `noexcept` doesn't form part of the function *signature* (like the return type), so
functions differing only in their `noexcept`ness cannot be overloaded. (This is not completely clear
in the book IMO.)
