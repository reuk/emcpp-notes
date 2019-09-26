[Presentation link](https://docs.google.com/presentation/d/1OFtCBpyHdSd-y0xM-76W4QcnyJflUOe9nAZ7XoQRJ-A/edit?usp=sharing)

# General Language Features

## Structured Bindings

Motivation - accessing unnamed members, old syntax:

```
std::pair<int, bool> InsertElement (int el) { ... }
auto ret = InsertElement (...); // need to access members as ret.first, ret.second
```

Could be more expressive as:

```
int index { 0 };
bool inserted { false };
std::tie (index, inserted) = InsertElement (10);
```

Better still, less code:

```
auto [index, inserted] = InsertElement (10);
```

Syntax:

```
auto [a, b, c, ...] = expression;
[[maybe_unused]] auto [a, b, c, ...] = expression;
```

Modifiers: const, &, &&

```
const auto [a, b, c, ...] = expression;
auto& [a, b, c, ...] = expression;
const auto& [a, b, c, ...] = expression;
auto&& [a, b, c, ...] = expression;
```

`a, b, c, ...` bind to a hidden object that is a copy or reference to the object obtained in expression.

Binding by value (`auto`, `const auto`) gives us “read” access (reading a copy):

```
std::pair myPair (0, 1.0f);  // Class Template Argument Deduction!
auto [x, y] = myPair;
x = 7;                       // modifies our copy, but myPair stays untouched
const auto [x, y] = myPair;
//x = 7;                     // illegal, x is const
```

Binding by reference gives us “write” access:

```
std::pair myPair (0, 1.0f); 
auto& [x, y] = myPair;
x = 7;      // myPair is now <7, 1.0f>
```

Can be conveniently used with `std::map`:

```
std::map<int, const char*> myMap;

// C++14:
for (const auto& elem : myMap) 
    // elem.first - is the key, elem.second - is the value

// C++17:
for (const auto& [key, val] : myMap) 
    // use key/value directly
```

Binding to an array:

```
double myArray[3] = { 1.0, 2.0, 3.0 }; 
auto [a, b, c] = myArray;

// The number of identifiers must match the number of elements in the array.
// In this example the whole array is copied into a temporary object (since we are not binding by reference above), could use a reference instead.
```

Binding to non-static data members:

```
struct Point 
{ 
    double x; 
    double y;
};

Point getStartPoint() { return { 0.0, 0.0 }; }

const auto [x, y] = getStartPoint();

// Does not require PODs, but the number of identifiers must equal the number of non-static data members, the members must be accessible (can be private if accessed from friend functions).
```

Providing binding support for custom types:

Need to define `get<N>()`, `std::tuple_size` and `std::tuple_element` specializations:

```
struct UserEntry 
{
	std::string getName() const { return name; }
	unsigned getAge()     const { return age; }
private:
	std::string name;
	unsigned age { 0 };
	size_t cacheEntry { 0 }; // not exposed
};

template <size_t Index> auto get (const UserEntry& u) 
{ 
	if constexpr (Index == 0) return u.getName(); 
	if constexpr (Index == 1) return u.getAge();
}

//or:
//template<> std::string get<0> (const UserEntry &u) { return u.getName(); } 
//template<> unsigned    get<1> (const UserEntry &u) { return u.getAge(); }

namespace std 
{
    template <> struct tuple_size<UserEntry> : integral_constant<size_t, 2> {};

    template <> struct tuple_element<0, UserEntry> { using type = std::string; };
    template <> struct tuple_element<1, UserEntry> { using type = unsigned; };
}

UserEntry u = ...;
auto [name, age] = u;
```


## Init statement for `if` and `switch`

```
// Pre C++17
auto val = GetValue(); 
if (condition (val))
	// on true
else
  	// on failure
```

```
// C++17
if (auto val = GetValue(); condition (val)) 
	// on true
else
	// on false
```

Can be conveniently combined with structured bindings:

```
if (auto [iter, succeeded] = mymap.insert (value); succeeded)
{ 
	use (iter); // OK
	// ...
} // iter and succeeded are destroyed here
```


## `inline` Variables

Allow to declare and define member variables in one place.

Semantics as for `inline` functions:

- Can be defined, identically, in multiple translation units.
- Must be defined in every translation unit where they are used.
- The behaviour of the program must be as if there was exactly one variable.

Example:

```
struct MyClass 
{
    inline static const int sValue = 777;
};
```

Unlike `constexpr` variables, `inline` variables don’t have to be initialized with a constant expression:

```
class MyClass 
{
	static inline int seed = rand();
};

// All of MyClass::seed instances will have the same value (generated at runtime)!
```

## Capturing `[*this]` in Lambda Expressions


Capturing plain `this` can cause a dangling pointer issue:

```
struct Baz 
{ 
	auto foo() { return [this] { std::cout << s << '\n'; }; }
	std::string s;
};

int main() 
{
	auto f1 = Baz{"ala"}.foo(); 
	f1();	// the lambda will now reference a dangling pointer to this!
}
```

Capturing an object by `*this` will capture a copy of the whole object:

```
struct Baz 
{ 
	auto foo() { return [*this] { std::cout << s << '\n'; }; }
	std::string s;
};

int main() 
{
	auto f1 = Baz{"ala"}.foo(); 
	f1();	// safe now!
}
```

C++14 workaround:

```
struct Baz 
{ 
	auto foo() { return [self = *this] { std::cout << self.s << '\n'; }; }
	std::string s;
};

int main() 
{
	auto f1 = Baz{"ala"}.foo(); 
	f1();	// safe now!
}
```

## Other Features

### `constexpr` Lambda Expressions

Example:

```
constexpr auto SquareLambda = [] (int n) { return n * n; }; 
static_assert (SquareLambda(3) == 9);
```

For lambda to be `constexpr`, the body should not invoke any code that is not `constexpr` (such as dynamic memory allocations or `throw` calls)

### Nested namespaces

Old syntax:

```
namespace foo { namespace bar {
	// stuff...
} }
```

Can now be compacted to:

```
namespace foo::bar {
	// stuff...
}
```

### `__has_include` preprocessor expression

Can be used to check if a given header exists.

This feature was already implemented by multiple compilers before.

Can give misleading results as a compiler can provide a stub, empty implementation.

See `roli_optional.h` for an example.

Be careful not to write this though as it as compiler not supporting `__has_include` will fail to compile this line:

```
#if defined __has_include && __has_include (<charconv>)
```
