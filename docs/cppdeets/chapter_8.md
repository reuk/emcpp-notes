# C++17 in Detail: Chapter 8 - `std::variant`

## Overview

* Type safe union - You can store objects of different types - with proper lifetime guarantee.
* Similar to a c-style union, and in most cases much superior
* Can be comlpex objects (e.g. `std::vector<std::string>`)
* Useful for implementing certain design patterns e.g. visitor pattern, pattern matching and runtime polymorphism for unrelated hierarchies
* Alternative to boost::variant and juce::var (but you can specify the types the variant holds)

Things you can do:
* Construct an object that can hold an int, float and a string. If it is uninitialised then it will default to calling the default constructor of the first type (note that this type must have one!)
```
#include <variant>
std::variant<int, float, string> intFloatString;
```

* Use a Visitor to do something to the object no matter what type it is:

```
struct PrintVisitor 
{
    void operator() (int i) { cout << "int: " << i << '\n'; }
    void operator() (float f) { cout << "float: " << f << '\n'; }
    void operator() (const string& s) { cout << "str: " << s << '\n'; }
};

std::visit(PrintVisitor{}, intFloatString);


OUTPUT:
int: 0
```

* Set the value to a float/string and print the index of the type
```
intFloatString = 100.0f;
cout << "index = " << intFloatString.index() << endl;

intFloatString = "hello super world";
cout << "index = " << intFloatString.index() << endl;

OUTPUT
index = 1
index = 2
```

* Use `getIf<Type>` to get a pointer to an object of Type. Returns nullptr if the variant doesn’t currently store an object of this type
```
if (const auto intPtr = get_if<int> (&intFloatString))
    cout << "int: " << *intPtr << '\n';
else if (const auto floatPtr = get_if<float> (&intFloatString))
    cout << "float: " << *floatPtr << '\n’;
    
OUTPUT:

```
* Use `get<Type>` to retrieve a reference to an object of `Type`. This will throw a `bad_variant_access` exception if it doesn’t contain one of those types:
```
try 
{
    auto f = std::get<float>(intFloatString);
    cout << "float! " << f << '\n';
}
catch (std::bad_variant_access&) 
{
    cout << "our variant doesn't hold float at this moment...\n";
}
Output:
our variant doesn't hold float at this moment...
```

* Use` holds_alternative<int> ()`
 ```
if (holds_alternative<int> (intFloatString))
    cout << "the variant holds an int!\n";
else if (holds_alternative<float> (intFloatString))
    cout << "the variant holds a float\n";
else if (holds_alternative<string> (intFloatString))
    cout << "the variant holds a string\n”;

OUTPUT:
the variant holds a string
```

* Other things to note:
    * No extra heap allocation occurs
    * Constructors/Destructors of non-trivial classes are called when switching 
    

## Creation 
* Default construction - as before
```
std::variant<int, float> intFloat;
std::cout << intFloat.index() << ", val: " << std::get<int> (intFloat) << '\n’;

Output:
0, val: 0
```

* Monostate for default initialisation

```
class NotSimple 
{ 
public:
    NotSimple(int, float) {}
};

std::variant<NotSimple, int> cannotInit; // error
std::variant<std::monostate, NotSimple, int> okInit; 
std::cout << okInit.index() << '\n’;

Output:
0
```
* Passing a value:
```
std::variant<int, float, std::string> intFloatString { 10.5f }; 
std::cout << intFloatString.index() << ", value " << std::get<float>(intFloatString) << '\n’;

Output:
1
```
* Watch out for ambiguity:
 ```
std::variant<int, float, std::string> intFloatString { 10.5 }; //Compiler error
```
* In place
    * Can be used to resolve ambiguity
    * And for efficient creation of complex types (similar to std::optional)
    * `std::in_place_type` and `std::in_place_index`
```
std::variant<long, float, std::string> longFloatString 
{ 
        std::in_place_index<1>, 7.6 // double!
};

// could also be:
variant<long, float, std::string> longFloatString 
{
        std::in_place_type<float>, 7.6 // double!
};

std::cout << longFloatString.index() << ", value "
<< std::get<float>(longFloatString) << '\n’;

// in_place for complex types
std::variant<std::vector<int>, std::string> vecStr 
{ 
        std::in_place_index<0>, { 0, 1, 2, 3 }
};

std::cout << vecStr.index() << ", vector size " 
        << std::get<std::vector<int>> (vecStr).size() << '\n';

Output:
1, value 7.6
0, vector size 4
```
* Copy initialisation
```
std::variant<int, float> intFloatSecond { intFloat }; 
std::cout << intFloatSecond.index() << ", value "
          << std::get<int>(intFloatSecond) << '\n’;

Output:
0, value 0
```

* Unwanted Type Narrowing and Conversion
    * First implentation used regular c++ expressions which produced some odd results. Fixed in GCC 10.0
    * What does the following lines code produce?
```
std::variant<bool, string> v = “Hello”;
std::variant<float, optional<double>> x = 10.05;
std::variant<float, char> v = 0;
std::variant<float, long> v = 0;
```
|Expression| Before | After |
|--|--|--|
|`std::variant<bool, string> v = "Hello"` | `bool` | `string` |
|`std::variant<float, optional<double>> x = 10.05` | `float` | `optional` |
|`std::variant<float, char> v = 0` | ill-formed | ill-formed |
|` std::variant<float, long> v = 0` | ill-formed | `long` |



    
Changing the Values
* Four types (well 5 types really):
    * assignment operator
```
std::variant<int, float, std::string> intFloatString { "Hello" };
intFloatString = 10; // we're now an int
```
* emplace (use index as  template parameter)
```
intFloatString.emplace<2> (std::string ("Hello")); // we're now string again
```
*  using the reference returned from get
```
std::get<std::string> (intFloatString) += std::string (" World”);
```
* Using getIf to change a pointer to the object
```
intFloatString = 10.1f;
if (auto pFloat = std::get_if<float> (&intFloatString); pFloat)
    *pFloat *= 2.0f;
 ```
*  Using a visitor (can’t change type but can change the value of the current type)



## Object Lifetime
* `std::variant` manages object lifetime (unlinke c-style union)
    * calls the appropriate constructors and destructors when changing types:
 ```

{
    std::variant<std::string, int> v { "Hello A Quite Long String" }; // v allocates some memory for the string
    v = 10; // we call destructor for the string!
}
// no memory leak

```
* Or using custom classes
```
class MyType 
{ 
public:
    MyType() { std::cout << "MyType::MyType\n"; }
    ~MyType() { std::cout << "MyType::~MyType\n"; } 
};

class OtherType {
 public:
    OtherType() { std::cout << "OtherType::OtherType\n"; }
    OtherType (const OtherType&) { std::cout << "OtherType::OtherType copy ctor\n"; }
    ~OtherType() { std::cout << "OtherType::~OtherType\n"; } 
};

int main() {
    std::variant<MyType, OtherType> v; 
    v = OtherType();
return 0; 
}

OUTPUT
MyType::MyType
OtherType::OtherType
MyType::~MyType
OtherType::OtherType copy ctor
OtherType::~OtherType
OtherType::~OtherType
```


## Access
* Cannot do something like this:
```
std::variant<int, float, std::string> intFloatString { "Hello" }; 
std::string s = intFloatString;
```
* You can use `get<Type>` with either `Type` or the index of the type.
* Will throw an exception if it is the wrong type.
```
std::variant<int, float, std::string> intFloatString; 
try 
{
    auto f = std::get<float> (intFloatString);
    std::cout << "float! " << f << '\n’;

    // alternative
    auto f = std::get<1> (intFloatString);
}
catch (std::bad_variant_access&) 
{
    std::cout << "our variant doesn't hold float at this moment...\n";
}
```
* Alternatively use `get_if<>`, again with either `Type` or an index.
    * Also non-member but won’t throw. 
    * Returns a pointer to `Type` or `nullptr`
    * Takes pointer to object (whereas get takes a reference)
 ```
if (const auto intPtr = std::get_if<0> (&intFloatString)
     std::cout << "int!" << *intPtr << '\n';
```

## Visitor
* Probably the most important way to access
* Use the `std::visit` function, nb multiple vars:
```
template <class Visitor, class... Variants>
constexpr visit(Visitor&& vis, Variants&&... vars);
```
* A visitor: Must be a callable object that has an overload for every type in the variant.
    * Generic lambda: simple example, as all types support `operator<<`:
```
auto PrintVisitor = [](const auto& t) { std::cout << t << '\n'; }; 

std::variant<int, float, std::string> intFloatString { "Hello" };
std::visit (PrintVisitor, intFloatString);

OUTPUT
Hello
```
* Can also modify variant:
```
auto PrintVisitor = [](const auto& t) { std::cout << t << '\n'; }; 
auto TwiceMoreVisitor = [](auto& t) { t*= 2; };

std::variant<int, float> intFloat { 20.4f }; 
std::visit (PrintVisitor, intFloat); 
std::visit (TwiceMoreVisitor, intFloat);
std::visit(PrintVisitor, intFloat);

OUTPUT
20.4
40.8
```
* Generic lambdas are simple and useful, but we can also define a structure with overloaded `operator()`
    * when we don’t want to treat all types the same,
    * or we want to use some kind of state in our visitor
```
struct MultiplyVisitor 
{
    float mFactor;

    MultiplyVisitor (float factor) : mFactor (factor) { }

    void operator() (int& i) const
    {
        i *= static_cast<int> (mFactor);
    }

    void operator()(float& f) const 
    { 
        f *= mFactor;
    }

    void operator()(std::string& ) const 
    { 
        // nothing to do here...
    } 
};

std::visit (MultiplyVisitor (0.5f), intFloat);
std::visit (PrintVisitor, intFloat);

OUTPUT
10.2
```

* Useful feature, but not currently in standard library (may be in C++20), is the overload structure:
```
template<class… Ts> struct overload : Ts… { using Ts::operator()…; };
template<class… Ts> overload(Ts…) -> overload<Ts…>;
```
* Creates a struct that inherits from the lambdas
```
std::variant<int, float, std::string> myVariant; 
std::visit(
    std::overload 
    {
        [](const int& i) { std::cout << "int: " << i; },
        [](const std::string& s) { std::cout << "string: " << s; }, 
        [](const float& f) { std::cout << "float: " << f; }
    },
    myVariant 
);
```
* Visiting multiple variants
    * Visitor must provide overloads for all combinations of variants.
    * So to visit 2 variants, each with 3 types, the visitor must provide overloads for all 9 combinations. 
    * Compiler error if not
```
std::variant<int, float, char> v1 { 's' };
std::variant<int, float, char> v2 { 10 };

std::visit(
    overload
    {
        [](int a, int b) { }, 
        [](int a, float b) { }, 
        [](int a, char b) { }, 
        [](float a, int b) { }, 
        [](float a, float b) { }, 
        [](float a, char b) { }, 
        [](char a, int b) { }, 
        [](char a, float b) { }, 
        [](char a, char b) { }
    }, v1, v2);
```
* Athough generic lambdas can be used alongside/instead:
```
struct Pizza { }; 
struct Chocolate { };
struct Salami { };
struct IceCream { };

int main()
{
    std::variant<Pizza, Chocolate, Salami, IceCream> firstIngredient{IceCream()}; 
    std::variant<Pizza, Chocolate, Salami, IceCream> secondIngredient{Chocolate()};


std::visit(overload{
    [](const Pizza& p, const Salami& s) {
        std::cout << "here you have, Pizza with Salami!\n"; 
    },
    [](const Salami& s, const Pizza& p) {
        std::cout << "here you have, Pizza with Salami!\n";
     },
    [](const Chocolate& c, const IceCream& i) {
        std::cout << "Chocolate with IceCream!\n"; 
    },
    [](const IceCream& i, const Chocolate& c) {
        std::cout << "IceCream with a bit of Chocolate!\n";
    },
    [](const auto& a, const auto& b) {
        std::cout << "invalid composition...\n"; 
    },
    }, firstIngredient, secondIngredient);
    
    return 0; 
}

OUTPUT:
IceCream with a bit of Chocolate!
```


* Operations
    * Comparisons: If objects are of the same type then it calls corresponding operator, if not then the ‘earlier’ type (lower Type index) is less than.
    * `std::move`: Can be used
    * `std::hash` is availble if all types are hasable
* Exception Safety Guarantees
```
class ThrowingClass { 
public:
    explicit ThrowingClass(int i) { if (i == 0) throw int (10); }
    operator int () { throw int(10); } 
};

int main(int argc, char** argv) { 
    std::variant<int, ThrowingClass> v;

    // change the value:
    try {
        v = ThrowingClass(0);
    }
    catch (...) {
        std::cout << "catch(...)\n";
        // we keep the old state!
        std::cout << v.valueless_by_exception() << '\n'; std::cout << std::get<int>(v) << '\n';
    }
    // inside emplace
    try {
        v.emplace<0>(ThrowingClass(10)); // calls the operator int
    }
    catch (...) {
        std::cout << "catch(...)\n";
        // the old state was destroyed, so we're not in invalid state! 
        std::cout << v.valueless_by_exception() << '\n';
    }
    return 0; 
}
```
* If an exception is thrown during destruction:
     * Assignment: Previous type will still be valid
     * Emplace(): State will be invalid (and can be checked using valueless_by_exception() member function). 
    * Calling index() on an invalid state will return variant::npos, and std::get and std::visit will throw a variant_bad_access
* Performance & Memory Considerations
    * Will need to be large enough to store largest type
    * Plus a discriminator
    * Usual alignment rules apply

* Possible use examples:
    * Mostly superior alternative to C-style unions (although there a few use cases for unions according to the cpp core guidelines)
    * Input variable that could be one of several different types (e.g. parsing a command line)
    * Output from a computation that could be one of several outcomes
    * Error handling: `std::variant<Object, ErrorCode>` so you can return an `Object` if successful, or an error code if not.
    * Finite State Machines
    * Polymorphism without vtables and inheritance (i.e. using the visitor pattern)

