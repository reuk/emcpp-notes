# Chapter 11: String Conversion

C++17 adds functions for fast low-level string conversion.

* `std::from_chars` for converting from strings into numbers, integers and floating points.
* `std::to_chars` for converting numbers into a string

## Current Issues

* **Safety:** Potential for buffer overruns
* **Format parsing:** Required for C-style APIs
* **Exceptions:** Performance hit and limited information
* **Whitespaces:** May ignore whitespaces
* **Locale:** Potentially unnecessary overhead for internationalization.

## Solution

* Non-throwing
* Error reporting of the conversion outcome
* Non-allocating
* Memory safety
* No string formatting of numbers
* No locale support

## `std::from_chars`

```cpp
std::from_chars_result from_chars (const char* first,
                                   const char* last,
                                   TYPE& value,
                                   int base = 10);
struct from_chars_result
{
    const char* ptr;
    std::errc ec;
};
```

### Conversion Information

* **Success**: `std::from_chars_result::ec` is value-initialized.
* **Invalid conversion**: `std::from_chars_result::ec == std::errc::invalid_argument`. `value` is unmodified.
* **Out of range**: `std::from_chars_result::ec == std::errc::result_out_of_range`. `value` is unmodified.

```cpp
std::from_chars_result from_chars (const char* first,
                                   const char* last,
                                   TYPE& value,
                                   int base = 10);

struct from_chars_result {
    const char* ptr;
    std::errc ec;
};
```

```cpp
std::string_view numbers {"1234"};
int value = 0;
const auto r = std::from_chars (numbers.data(),
                                numbers.data() + numbers.size(),
                                value);
if      (r.ec == std::errc())                    { std::cout << value << "\n"; }
else if (r.ec == std::errc::invalid_argument)    { std::cout << "invalid\n"; }
else if (r.ec == std::errc::result_out_of_range) { std::cout << "out of range\n"; }
```

```bash
input: 1234             value: 1234
input: 12345678901234   value: out of range
input: asdfa            value: invalid
```

## `std::to_chars`

### Conversion Information

* **Success**: `std::from_chars_result::ec` is value-initialized.
* **Error**: `ptr == first` and `ex == std::errc::invalid_argument`.
* **Out of range**: `ec == std::errc::value_too_large`. `first` & `last` in unspecified state.

```cpp
std::to_chars_result to_chars (const char* first,
                               const char* last,
                               TYPE& value,
                               int base = 10);

struct to_chars_result {
    const char* ptr;
    std::errc ec;
};
```

```cpp
std::string text {"xxxxxx"};
const int value = 1234;
const auto r = std::to_chars (text.data(),
                              text.data() + text.size(),
                              value);

if (r.ec == std::errc())
    std::cout << text << "\n";
else
    std::cout << "too large\n";
```

```bash
input: 1234         value: 1234xx
input: 12345678     value: too large
```

## Benchmark: From Characters

![from_char](from_char.png)

## Benchmark: To Characters

![to_char](to_char.png)

## Guidelines

* Wrap functions for better expressiveness
* Consider when conversion is a bottleneck