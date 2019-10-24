# C++17 in Detail: Chapter 9 - `std::any`

## Overview
* Alternative to using `void*` (with a discriminator) to store objects of any type
   * This method is error prone:
       * have to handle lifetime of object
       * and prevent casting to an incorrect type
            
* Uses:
   * In libraries when you want to pass objects around, but you don’t know what they are
   * Parsing files  
   * Messaging passing
   * Bindings with a scripting language
   * Implementing an interpreter
   * User Interface - controls might hold anything
   * Entities in an editor
        
* std::variant is preferable however if it is possible to fix types
    


## Construction:
* default initialisation:
```
std::any a;

assert (! a.has_value());
```
  

* initialisation with an object:
```
std::any a2 { 10 }; // int

std::any a3 { MyType { 10, 11 } };
```
  

* `in_place`:
```
std::any a4 { std::in_place_type<MyType>, 10, 11 };

std::any a5 { std::in_place_type<std::string>, “Hello World” };
```
  

* `make_any`

```
std::any a6 = std::make_any<std::string> { "Hello World” };
```

## Changing the value:
    

* assignment operator:
```
std::any a;

a = MyType (10, 11);

a = std::string ("Hello”);
```
  

* `emplace`
```
a.emplace<float> (100.5f);

a.emplace<std::vector<int>> ({ 10, 11, 12, 13 });

a.emplace<MyType> (10, 11);
```

## Accessing:
    
```
std::any var = 10;

// read access (just copies)
auto a = std::any_cast<int> (var);

// read/write access through a reference:
std::any_cast<int&> (var) = 11;

// read/write through a pointer:
int* ptr = std::any_cast<int> (&var);
*ptr = 12;
```

* Pointer will return nullptr if not the correct type
   * Otherwise Will throw `std::bad_any_cast` if calling on an incorrect type
    
```
struct MyType
{
    int a, b;
    MyType (int x, int y) : a (x), b (y) { }
    void print() { std::cout << a << ", " << b << '\n'; }
};

int main() 
{
    std::any var = std::make_any<MyType> (10, 10);

    try
    {
        std::any_cast<MyType&> (var).print();
        std::any_cast<MyType&> (var).a = 11; // read/write
        std::any_cast<MyType&> (var).print();
        std::any_cast<int> (var); // throw!
    }
    catch (const std::bad_any_cast& e)
    {
        std::cout << e.what() << '\n';
    }


    int* p = std::any_cast<int> (&var);

    std::cout << (p ? "contains an int... \n" : "doesn't contain an int...\n");

  
    if (MyType* pt = std::any_cast<MyType> (&var); pt)
    {
        pt->a = 12;
        std::any_cast<MyType&> (var).print();
    }
}

  

OUTPUT:

10, 10
11, 10
bad any_cast
doesn't contain an int...
12, 10
```

* Performance and Memory Considerations
   * Dynamic allocation probably!
      * If and when is Implementation specific
      * Size of any object varies between compilers due to SBO (small buffer optimisation)
