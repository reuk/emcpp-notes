# Chapter 10: ```std::string_view```

A `std::string_view` pointer to a contiguous character sequence and the length (or end pointer).

It avoids unnecessary heap allocation by not taking ownership.

Much like `juce::StringRef` but with added functionality.

## Example: Temporary Copies

Even with copy elision the following example does 3 string copies.

```cpp
std::string fromFirstOccurrenceOf (const std::string& content,
                                   const std::string& target)
{
    return content.substr (content.find (target));
}

std::string text {"Hello Amazing Programming Environment"};

auto subString = fromFirstOccurrenceOf (text, "Prog");
std::cout << subString << '\n';
```

With ```std::string_view``` only one allocation takes place.

```cpp
std::string_view fromFirstOccurrenceOf (std::string_view content,
                                        std::string_view target)
{
    return content.substr (content.find (target));
}

std::string text {"Hello Amazing Programming Environment"};

auto subString = fromFirstOccurrenceOf (text, "Prog");
std::cout << subString << '\n';
```

## Usage

Different character widths are available through template specializations of `std::basic_string_view`. As with `std::string` and `std::basic_string`.

```cpp
using std::string_view    = std::basic_string_view<char>
using std::wstring_view   = std::basic_string_view<wchar_t>
using std::u8string_view  = std::basic_string_view<char8_t> // (C++20)
using std::u16string_view = std::basic_string_view<char16_t>
using std::u32string_view = std::basic_string_view<char32_t>
```

`juce::StringRef` uses the selected character encoding type to determine the width.

### Construction

```cpp
constexpr basic_string_view() noexcept;
constexpr basic_string_view (const basic_string_view& other) noexcept = default;
constexpr basic_string_view (const CharT* s, size_type count);
constexpr basic_string_view (const CharT* s);
```

Also by using a conversion from `std::string`. Or by using the `""sv` literal (in `std::literals`).

```cpp
using namespace std::literals;
constexpr auto text = "Heya, World!"sv;
```

### Methods

Almost the same as all non-mutable string operations.

All of the methods (except for `copy()`, `operator<<`, and `std::hash` specialisation) are `constexpr`.

Some interesting methods `remove_prefix()`, `remove_suffix()`, `starts_with()`, `ends_with()`, iterators...

`substr()` is **O(1)** instead of **O(n)** for the `std::string` equivalent.

## Risks

### Null-Termination

`"\0"` is not necessarily present in `std::string_view`.

Problem with **all** C-string functions. If a function accepts only a `const char*` parameter, it's probably not safe to pass a `std::string_view`, e.g. `std::basic_ostream::operator<<`.

### References and Temporary Objects

Since `std::string_view` doesn't own the memory, the objects lifetime should never exceed the lifetime of the string-owning object.

Crucial when:

* Returning a `std::string_view` from a function
* Storing a `std::string_view` in objects and containers

### Undefined Behaviour

If the view is empty, you'll get undefined behaviour with `operator[]`, `front()`, `back()`, `data()`.

So when uncertain, check with `empty()`.

### Reference Lifetime Extension

When replacing a `const std::string&` with a `std::string_view` lifetime extension is lost.

```cpp
std::string createTemporary() { return {}; }

std::string_view   nonExtended = createTemporary();
const std::string& extended    = createTemporary();
```

[TotW 101](https://abseil.io/tips/101), [ToW 107](https://abseil.io/tips/107)

## `std::string` from a `std::string_view`

```cpp
struct UserName
{
    std::string name;
    UserName (const std::string& name) : name (name) {}
};

UserName u1 {"John With Very Long Name"};       // from const char*

std::string s2 {"Marc With Very Long Name"};    // from lvalue
UserName u2 { s2 };

td::string s3 {"Marc With Very Long Name"};     // from rvalue
UserName u3 { std::move (s3) };

std::string GetString() { return "some string..."; }
UserName u4 { GetString() };                    // from return
```

| input parameter   | `const std::string&` | `std::string_view` | `std::string` & move  |
|-------------------|----------------------|--------------------|-----------------------|
| `const char*`     | 2 allocations        | **1 allocation**   | 1 allocation + move   |
| `const char*` SSO | 2 copies             | **1 copy**         | 2 copies              |
| lvalue            | **1 allocation**     | **1 allocation**   | 1 allocation + move   |
| lvalue SSO        | **1 copy**           | **1 copy**         | 2 copies              |
| rvalue            | 1 allocation         | 1 allocation       | **2 moves**           |
| rvalue SSO        | **1 copy**           | **1 copy**         | 2 copies              |

## Main points

* Non-owning view of contiguous sequence of characters.
* Contains most of `std::string` operations that don't modify the underlying characters.
* Might not be Null-Terminated.
* Good for viewing splices of a string.
* Most operations are marked `constexpr`.
* Looks like a constant reference but doesn't extend lifetime of returned temporary.

## Guidelines

* Consider for read-only functions otherwise favour pass-by-value and move.
* Use for `constexpr` string analysis.
* A function that returns a `std::string_view` should probably take a `std::string_view`.
* Don't use `std::string_view` with functions that only take a `const char*`.
