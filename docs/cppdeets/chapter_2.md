# Removed or Fixed Language Features

## Removed Elements

The following items have all been removed. We'd get build errors if we were using any of these
under C++17, so we can be fairly certain that we're not accidentally using them.

- The register did nothing since C++11, now it's been removed.
- `bool++` was deprecated in C++98, `++bool` was deprecated in C++03. Both have been removed in
  C++17.
- Old-style exception specifications were removed as they were too impractical. `throw()` still
  remains, but and means the same as `noexcept(true)`. `noexcept` is should be preferred.
  - `throw()` isn't directly used in our repo, but is used by a couple of dependencies.
- No more trigraphs.

## Fixes

### Direct List Initialisation with `auto`

C++17 updates the type deduction rules for direct list initialisation.
```
auto item { 1.0f };     // direct initialisation, deduces type `float` since C++17
auto list = { 1.0f };   // copy initialisation, deduces type `std::initializer_list<float>`
```

- For direct initialisation from a braced-init-list with a single element, auto deduction will
  deduce the type of that element.
- For direct initialisation from a list with more than one element, auto deduction will be
  ill-formed.

```
auto x {1, 2}; // this will cause a compile error as of C++17
```

### `static_assert` with No Message

Frequently, we don't need to describe static assertions as their conditions are descriptive enough.

### Different `begin` and `end` Types in Range-Based For Loop

Allows an `end` iterator to have a different type to the `begin` iterator, which is useful
for implementing sentinels. Makes possible the implementation of C++20 ranges.
