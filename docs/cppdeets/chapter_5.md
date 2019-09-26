[Presentation link](https://docs.google.com/presentation/d/1OFtCBpyHdSd-y0xM-76W4QcnyJflUOe9nAZ7XoQRJ-A/edit?usp=sharing)


# Templates

## Fold Expressions

Reduce parameter packs over a binary operator.

Pre C++17 we would write:

```
template<typename T1, typename... T> 
auto SumCpp11 (T1 s, T... ts) 
{
	return s + SumCpp11 (ts...); 
}

auto SumCpp11() { return 0; }
```

Since C++17 we can write (notice the use of `()` parentheses):

```
template<typename ...Args> 
auto sum (Args ...args) 
{ 
    return (args + ... + 0);
}

// or simply...
template<typename ...Args> 
auto sum (Args ...args) 
{ 
    return (args + ...);
}
```


### Fold types

Expression         | Name              | Expansion
------------------ | ----------------- | -------------------------------------
(... op e)         | unary left fold   | ((e1 op e2) op ...) op eN
(init op ... op e) | binary left fold  | (((init op e1) op e2) op ...) op eN
(e op ...)         | unary right fold  | e1 op (... op (eN-1 op eN))
(e op ... op init) | binary right fold | e1 op (... op (eN-1 op (eN op init)))
   

For binary folds, both ops must be the same

Left vs right fold comparison:

```
// right fold
template<typename ...Args>  
auto sum (Args ...args)  { return (args + ...);  }

// left fold
template<typename ...Args>  
auto sum2 (Args ...args) { return (... + args);  }


auto value   = sum  (1, 2, 3, 4); // expands to 1 + (2 + (3 + 4))
auto value2  = sum2 (1, 2, 3, 4); // expands to ((1 + 2) + 3) + 4
```

**Note:** folds over comma always evaluate from left to right, despite () parentheses!: 

```
// This will evaluate in the following order: exp1, exp2, exp3, exp4!
// exp1 == expression1 etc.
exp1 , (exp2 , (exp3 , exp4)); 
```

### Examples

Example 1 (fold over comma):

```
template<typename T, typename... Args>
void push_back_vec (std::vector<T>& v, Args&&... args) 
{
	(v.push_back (std::forward<Args> (args)), ...);
}

std::vector<float> vf;
push_back_vec (vf, 10.5f, 0.7f, 1.1f, 0.89f); 
// vector content: { 10.5f, 0.7f, 1.1f, 0.89f } (same result would be with the left fold!)

```

Example 2 (binary fold):

```
template<typename ...Args>
void foldPrint (Args&&... args) 
{
	(std::cout << ... << std::forward<Args> (args)) << '\n';
}

foldPrint ("hello", 10, 20, 30); // will print "hello102030"
```

Example 3 (another fold over comma):

```
template<typename ...Args>
void foldPrintWithSpaces (Args&&... args) 
{
	auto printWithSpace = [](const auto& v) { std::cout << v << ' '; };

	(... , printWithSpace (std::forward<Args> (args)));
}

foldPrintWithSpaces ("hello", 10, 20, 30);  // will print "hello 10 20 30 "
```

### Allowed operators

`+ - * / % ^ & | = < > << >> += - = *= /= %= ^= &= |= <<= >>= == != <= >= && || , .* ->*`


### Default values for empty parameter packs

Operator     | Default value
------------ | ----------------
`&&`         | true
`||`         | false
`,`          | void()
any other    | ill-formed code

Note that calling `sum()` or `sum2()` from above examples with no params would not compile (ill-formed code).


## Class Template Argument Deduction (CTAD)

Until C++17 one had to write:

```
std::pair<int, float> myPair {10, 7.f};  // or auto myPair = std::make_pair {10, 7.f};
```

Since C++17 one can write:

```
std::pair myPair {10, 7.f};
auto otherPair = std::pair {10, 7.f};
auto myPairPtr = new std::pair {10, 7.f};
```


Until C++17 one had to write:

```
std::shared_timed_mutex mut;
std::lock_guard<std::shared_timed_mutex> lck (mut);

std::array<int, 3> arr {1, 2, 3};
```

Since C++17 one can write:

```
std::shared_timed_mutex mut;
std::lock_guard lck (mut);

std::array arr {1, 2, 3};
```

Partial deduction not possible:

```
std::tuple t (1, 2, 3);              // OK: deduction 
std::tuple<int,int,int> t (1, 2, 3); // OK: all arguments are provided 
std::tuple<int> t (1, 2, 3);         // Error: partial deduction
```


Many `std::make_*` functions are now not required, but remember that some may be doing other useful stuff apart from template argument deduction (like `std::make_shared` which ensures that allocated data and control block are in a contiguous memory region).

CTAD works thanks to deduction guides...

### Deduction guides

User defined deduction guide example:

```
template<class T> struct S { S(T); };

S(const char*) -> S<std::string>;

S s {"hello"}; // deduced as S<std::string> rather than S<const char*>
```

User defined deduction guide example 2:

```
template<class T> struct Container 
{
	Container (T t) {}
	template<class Iter> Container (Iter beg, Iter end);
};

template<class Iter>
Container (Iter b, Iter e) -> Container<typename std::iterator_traits<Iter>::value_type>;

Container c (7);                         // OK: deduces T=int using an implicitly-generated guide
std::vector<double> v = {/* ... */};
auto d = Container (v.begin(), v.end()); // OK: deduces T=double
Container e {5, 6};                      // Error: there is no std::iterator_traits<int>::value_type
```

In most cases the compiler will auto generate them for each ctor (incl. copy/move) of the primary class template, but it will not work if the class is specialised or partially specialised.

While compiler may support CTAD, its corresponding STL implementation may still lack deduction guides for some STL types.


Compiler generated deduction guide example:

```
// Notice a fold over ... operator.
template<typename T, typename... U> array (T, U...) -> array<enable_if_t<(is_same_v<T, U> && ...), T>, 1 + sizeof... (U)>;
```


### CTAD Limitations (until C++20)

It doesn’t work with template aggregate types:

```
template<class T>
struct Point { T x; T y; }; 

Point p {3.0, 4.0};           // Error: cannot deduce 
Point<double> p {3.0, 4.0};   // OK
```

It doesn't work with template aliases:

```
template typename<T>
using VecT = std::vector<T>;

VecT v {2, 5};        // Error: cannot deduce
VecT<int> v {2, 5};   // OK
```

It doesn’t work with inherited constructors.


## `if constexpr`

Compile time `if` statement, allowing to discard chunks of code at compilation stage.

```
template <typename T> 
auto getValue (T t) 
{
	if constexpr (std::is_pointer_v<T>) 
		return *t;
	else  				// else is required!
		return t;
}
```

`if constexpr` often allows for cleaner code:

```
// pre C++17, SFINAE approach
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T> simpleTypeInfo (T t) 
{
	std::cout << "foo<integral T> " << t << '\n';
	return t; 
}

template <typename T>
std::enable_if_t<! std::is_integral_v<T>, T> simpleTypeInfo (T t) 
{
	std::cout << "not integral \n";
	return t; 
}
```

```
// pre C++17, tag dispatch approach
template <typename T>
T simpleTypeInfoTagImpl (T t, std::true_type) 
{
    	std::cout << "foo<integral T> " << t << '\n';
	return t; 
}

template <typename T>
T simpleTypeInfoTagImpl (T t, std::false_type) 
{
	std::cout << "not integral \n";
	return t; 
}

template <typename T>
T simpleTypeInfoTag (T t) 
{
	return simpleTypeInfoTagImpl (t, std::is_integral<T>{}); 
}
```

```
// since C++17, if constexpr approach
template <typename T> T simpleTypeInfo (T t) 
{
	if constexpr (std::is_integral_v<T>)
		std::cout << "foo<integral T> " << t << '\n';
	else 
		std::cout << "not integral \n"; 

	return t; 
}
```


## Declaring Non-Type Template Parameters With `auto`

Example 1:

```
// Pre C++17
template <typename Type, Type value> 
constexpr Type TConstant = value;
 
constexpr auto const MySuperConst = TConstant<int, 100>;

// C++17
template <auto value> 
constexpr auto TConstant = value; 

constexpr auto const MySuperConst = TConstant<100>;
```

Example 2:

```
template <auto ... vs> 
struct HeterogenousValueList {}; 

using MyList = HeterogenousValueList<'a', 100, 'b'>;
```


## Other Changes

### Allow `typename` in template template parameter (previously only `class` keyword was allowed there):

```
template<typename T> class MyArray {};

// two type template parameters and one template template parameter:
template<typename K, typename V, template<typename> typename C = MyArray>
class Map
{
    C<K> key;
    C<V> value;
};
```


### Variable Templates for traits (follow additions in C++14), examples:

```
std::is_integral<T>::value  // pre C++17
std::is_integral_v<T>       // C++17

std::is_class<T>::value     // pre C++17
std::is_class_v<T>          // C++17
```


### Can use parameter pack expansions in `using` declarations:

```
template<class... Ts> 
struct Overloaded : Ts... 
{ 
	using Ts::operator()...;
};
```

`Overloaded` will inherit all overloads of `operator()` from all its base classes.


### Logical Operation Metafunctions

```
template<class... B> struct conjunction; - logical AND 
template<class... B> struct disjunction; - logical OR 
template<class B> struct negation;       - logical negation

template<typename... Ts> std::enable_if_t<std::conjunction_v<std::is_same<int, Ts>...> > 
printIntegers (Ts ... args) 
{
	(std::cout << ... << args) << '\n';  // fold over << operator
}
```


### `std::void_t` Transformation Trait

```
template <class...> 
using void_t = void;

void compute (int &) { } // example function 

template <typename T, typename = void>
struct is_compute_available : std::false_type {};
 
template <typename T>
struct is_compute_available<T,
  std::void_t<decltype (compute (std::declval<T>())) >> : std::true_type {};

static_assert (is_compute_available<int&>::value); 
static_assert (! is_compute_available<double&>::value);
```
