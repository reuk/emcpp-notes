# Searchers & String Matching

`std::search` in C++14 has O(m*n) complexity (m is pattern length, n is input length), so it's
often not fast enough to be useful. C++17 introduces overloads with amortized linear complexity.

## Overview

There are lots of searching algorithms with different time/space performance trade-offs. Faster
algorithms might build lookup tables to reduce execution time at the expense of memory usage.

C++17 introduces `search` overloads which accept a Searcher object which encapsulates the searching
algorithm to use. It also adds execution policy overloads, to run searches in a parallel way.

## Available Algorithms

m: length of pattern
n: length of 'text' (the input to search)
k: size of alphabet

- `default_searcher`
  - O(m*n) complexity
  - No space overhead
  - Requires forward iterators
- `boyer_moore_searcher` and `boyer_moore_horspool_searcher`
  - O(n/m) best, O(m*n) worst complexity
  - O(k) space overhead
  - Require random access iterators

`boyer_moore` is slower to initialise than `boyer_moore_horsepool` and uses two lookup tables.
Proper profiling in context is probably required to determine which algorithm has the right
trade-offs for a particular situation.

Searchers and execution policies cannot be used simultaneously.

## Using Searchers

Each searcher is constructed from the search string. Preprocessing will happen on construction,
so the searcher can be constructed once and used multiple times, with minimal overhead.

```cpp
// No allocation, still supports begin/end
const auto testString = std::string_view { "Hello world!" };
const auto toFind = std::string_view { "world" };

const auto it = std::search (testString.cbegin(),
                             testString.cend(),
                             std::boyer_moore_searcher (toFind.cbegin(), toFind.cend()));

if (it != testString.cend())
    ...
```
