---
marp: true

---

# C++ 17 in Detail
## Chapter 14: Parallel STL Algorithms

---

"Free lunch is over (Herb Sutter, 2006)"
http://www.gotw.ca/publications/concurrency-ddj.htm

---

## Ways of speeding up algorithms
- split task between multiple threads of execution
- use vector instructions (SIMD), most CPUs have 128bit wide registers, newer chips have 256 or 512 bits (AVX 256, AVX 512)
- run on GPU (via 3rd party APIs like CUDA, OpenCL, OpenMP etc.)

---

## Threading Overview
- pre C++11 - threading available through 3rd party libraries or system APIs
- since C++11 - standard library support for threads, atomics, locks, `std::async` and futures
- since C++17 - ability to parallelize most of standard library algorithms

---

## High level API
- simply added `ExecutionPolicy` template parameter

```cpp
template<class ExecutionPolicy, class RandomIt, ...> 
std::algorithm_name (ExecutionPolicy&& policy, 
                     RandomIt first, RandomIt last, 
                     ...);
```

- implementation details are hidden from the user, up to the implementer
- MSVC uses Windows thread pools, GCC uses Intel TBB library, Clang does not have support for parallel algorithms yet

---

Execution policy is one of three global objects:
- `std::execution::seq` (`sequenced_policy` type) - do not parallelize
- `std::execution::par` (`parallel_policy` type) - allow parallelization
- `std::execution::par_unseq` (`parallel_unsequenced_policy` type) - allow parallelization and vectorization

The types of policies do not share the same base type.

Defined in `<execution>` header.

---

Example:

```cpp
std::vector<float> vecX = {...};                   // generate data
std::vector<float> vecY (vecX.size());
std::transform (std::execution::seq,               // or par, or par_unseq
                begin (vecX), end (vecX),          // input range 
                begin (vecY),                      // output 
                [](float x) { return x * 2.0f; }); // operation
```

- in sequential case (`seq`) we will transform element one by one
- in parallel case (`par`) each transformation can be performed by a different thread, the order of transformation and the number of threads are undefined
- in parallel unsequential case (`par_unseq`) each operation can be performed by a different thread, but each thread may e.g. perform the operation on e.g. 4 vector members at once, also each instruction of the operation may be interleaved, so e.g. the first instruction may be called for multiple elements first before the next one, which is required for vectorization to work

---

## Updated algorithms

Name                   | Name                    | Name
---------------------- | ----------------------- | -------------------------------------
adjacent_difference    | inplace_merge           | replace_copy
adjacent_find          | is_heap                 | replace_copy_if
all_of                 | is_heap_until           | replace_if
any_of                 | is_partitioned          | reverse
copy                   | is_sorted               | reverse_copy
copy_if                | is_sorted_until         | rotate
copy_n                 | lexicographical_compare | rotate_copy


---

## Updated algorithms (cont.)

Name                   | Name                    | Name
---------------------- | ----------------------- | -------------------------------------
count                  | max_element             | search
count_if               | merge                   | search_n
equal                  | min_element             | set_difference
exclusive_scan         | minmax_element          | set_intersection
fill                   | mismatch                | set_symmetric_difference
fill_n                 | move                    | set_union
find                   | none_of                 | sort

---

## Updated algorithms (cont.)

Name                   | Name                    | Name
---------------------- | ----------------------- | -------------------------------------
find_end               | nth_element             | stable_partition
find_first_of          | partial_sort            | stable_sort
find_if                | partial_sort_copy       | swap_ranges
find_if_not            | partition               | transform
for_each               | partition_copy          | transform_exclusive_scan
for_each_n             | remove                  | transform_inclusive_scan
generate               | remove_copy             | transform_reduce

---

## Updated algorithms (cont.)

Name                   | Name                    | Name
---------------------- | ----------------------- | -------------------------------------
generate_n             | remove_copy_if          | uninitialized_copy
includes               | remove_if               | uninitialized_copy_n
inclusive_scan         | replace                 | uninitialized_fill
inner_product          | unique                  | uninitialized_fill_n
.                      | unique_copy             | .

---

## New algorithms

Algorithm                 | Description             
------------------------- | ----------------------- 
for_each                  | similar to previous version of  `for_each` except returns `void`
for_each_n                | applies a function object to the first n elements of a sequence 
reduce                    | similar to accumulate, except out of order execution to allow parallelism
transform_reduce          | transforms the input elements using a unary operation, then reduces the output out of order

---

## New algorithms (cont.)

Algorithm                 | Description             
------------------------- | ----------------------- 
exclusive_scan            | parallel version of `partial_sum`, excludes the i-th input element from the i-th sum, out of order execution to allow parallelism
inclusive_scan            | parallel version of `partial_sum`, includes the i-th input element in the i-th sum, out of order execution to allow parallelism
transform_exclusive_scan  | applies a functor, then calculates exclusive scan
transform_inclusive_scan  | applies a functor, then calculates inclusive scan

---

- `reduce()` and `scan()` also have "fused" versions which perform transformation and then the reduce or scan which will be faster than calling both operations separately, as we only prepare for multithreading once
- all new algorithms provide overloads without execution policy, to run them in sequence
- `for_each()` version with execution policy returns `void` rather than a value because the order of the operations is undefined

---

- `reduce()` computes final sum by diving computation into smaller subranges and then merging the results
- by default `std::plus<>{}` is used to compute the reduction steps
- `reduce()` requires the operation to be associative: 
  `(x @ y) @ z = x @ (y @ z)`
- `reduce()` requires the operation to be commutative:
  `x @ y = y @ x`

---

- cannot thus use e.g. subtraction as operation:

```cpp
1 - (5 - 4) != (1 - 5) - 4 // subtraction is not associative
1 - 7       != 7 - 1       // subtraction is not commutative
```

- also cannot sum floating point numbers:

```cpp
// #include <limits> - for numeric_limits
std::cout.precision (std::numeric_limits<double>::max_digits10); 
std::cout << (0.1 + 0.2) + 0.3 << " != " << 0.1 + (0.2 + 0.3) << '\n';

//The output:
//0.60000000000000009 != 0.59999999999999998
```

---

- `transform_reduce()` example

```cpp
std::vector<int> v { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

auto sumTransformed = std::transform_reduce (std::execution::par,
                                             v.begin(),
                                             v.end(),
                                             0,
                                             std::plus<int>{},
                                             [](const int& i) { return i * 2; });
```

---

Example with counting elements of a vector that meet a certain criteria:

```cpp
template <typename Policy, typename Iter, typename Func>
std::size_t CountIf (Policy policy, Iter first, Iter last, Func predicate) 
{
    return std::transform_reduce (policy, first, last,
                                  std::size_t (0), 
                                  std::plus<std::size_t>{}, 
                                  [&predicate](const Iter::value_type& v) { return predicate (v) ? 1 : 0; }); 
}
```

---

This can be used like so:

```cpp
std::vector<int> v (100);
std::iota (v.begin(), v.end(), 0);

auto NumEven = CountIf (std::execution::par, v.begin(), v.end(),
                        [](int i) { return i % 2 == 0; } );

std::cout << NumEven << '\n';
```

---

or like so:

```cpp
std::map<std::string, int> CityAndPopulation 
{ 
    { "Cracow", 765000 },
    { "Warsaw", 1745000 },
    { "London", 10313307 },
    { "New York", 18593220 },
    { "San Diego", 3107034 } 
};

auto NumCitiesLargerThanMillion = CountIf (std::execution::seq, 
                                           CityAndPopulation.begin(), 
                                           CityAndPopulation.end(), 
                                           [](const std::pair<const std::string, int>& p) 
                                           { return p.second > 1000000; });

std::cout << CitiesLargerThanMillion << '\n';
```

---

## Gotchas
- it is up to the user to ensure that the algorithm is safe to be used in parallel mode - there may be no deadlocks or data races

```cpp
std::vector<int> vec (1000);
std::iota (vec.begin(), vec.end(), 0);
std::vector<int> output;
std::mutex m;

std::for_each (std::execution::par, vec.begin(), vec.end(), 
               [&output, &m, &x](int& elem) 
               {
                    if (elem % 2 == 0) 
                    { 
                        std::lock_guard guard (m);  // required!
                        output.push_back (elem);
                    }  
                });
```

---

## Gotchas (cont.)
- using many synchronisation points may decrease performance significantly
=> try avoiding sharing resources when using parallel algorithms
- because `par_unseq` instructions can be interleaved, using vector unsafe functions (like locks or memory allocation) are forbidden as they can lead to deadlocks or data races (`std::lock_guard guard(m)` call is not safe, culd be called twice in a row for instance!)

---

## Gotchas (cont.)
- handling exceptions
    - `std::bad_alloc` may be thrown if the implementation fails to allocate resources needed
    - if user provided code (functor) throws an exception, `std::terminate()` is called

---

```cpp
try 
{
    std::vector<int> v { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    std::for_each (std::execution::par, v.begin(), v.end(), 
                   [](int& i) 
                   {
                        std::cout << i << '\n'; 
                        
                        if (i == 5)
                            throw std::runtime_error ("something wrong... !");
                   }); 
}
catch (const std::bad_alloc& e) 
{
    std::cout << "Error in execution: " << e.what() << '\n';
}
catch (const std::exception& e) // will not happen
{ 
    std::cout << e.what() << '\n';
}
catch (...) // will not happen
{ 
    std::cout << "error!\n";
}
```

---
## Gotchas (cont.)
- except for `for_each()` and `for_each_n()`, implementations are allowed to take copy of the data from sequences if for the data `is_trivially_copy_constructible_v<T>` and `is_trivially_destructible_v<T>` are true

```cpp
vector<int> vec;
vector<int> other;
vector<int> external;

int* beg = vec.data(); 

std::transform (std::execution::par, 
                vec.begin(), vec.end(), other.begin(), 
                [&beg, &external](const int& elem) 
                {
                    // use pointer arithmetic
                    auto index = &elem - beg;       // elem may be a copy rather than a reference!
                    return elem * externalVec[index]; 
                }
);
```

---
## Gotchas (cont.)
- a safe approach could involve a separate container for indices

```cpp
void Process (int a, int b) { }

std::vector<int> v (100); 
std::vector<int> w (100); 
std::iota (v.begin(), v.end(), 0); 
std::iota (w.begin(), w.end(), 0);

std::vector<size_t> indices (v.size()); 
std::iota (indices.begin(), indices.end(), 0);

std::for_each (std::execution::par, 
               indices.begin(), indices.end(), 
               [&v, &w](size_t& id) 
               { Process (v[id], w[id]); });
```

---
## Gotchas (cont.)
- or a zip iterator

```cpp
void Process (int a, int b) { }

std::vector<int> v (100); 
std::vector<int> w (100); 
std::iota (v.begin(), v.end(), 0); 
std::iota (w.begin(), w.end(), 0);

vec_zipper<int, int> zipped { v, w }; 

std::for_each (std::execution::seq, 
               zipped.begin(), zipped.end(),
               [](std::pair<int&, int&>& twoElements) 
               { Process (twoElements.first, twoElements.second); });
```

---

## Performance considerations
- parallel algorithms require extra work to setup and manage threading
- if synchronisation is required, then it can seriously decrease the performance
- sharing data between the cores can cause memory access bottleneck since a common desktop CPU cores share the same memory bus, so it's best if each core works on independent chunk of data
- performance may be heavily dependent on implementation (e.g. MSVC ignores parallel vectorized policy `par_unseq` and executes it as parallel `par`)
=> benchmark performance rather than assuming before hand that the parallel implementation will be faster

---
Is sequential execution policy useful at all?
- easier debugging
- may be faster than parallel (parallel requires threading setup and cleanup which is not for free, synchronisation may be costly)
- vectorized approach may be unsafe (vectorization unsafe operation may be used like locks or allocations)

---

Should we use parallel algorithms?
- YES! Parallel algorithms can significantly improve performance (~x number of cores in best scenarios)
- BUT: they have to be used with care to avoid bugs and to ensure that they bring performance benefits we want
