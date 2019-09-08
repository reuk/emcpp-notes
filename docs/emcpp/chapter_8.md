# Tweaks

## Consider pass by value for copyable parameters that are cheap to move and always copied.

Sometimes we want to 'sink' a function argument when we call a function:

```
class Widget {
public:
    // Both of these functions are going to make a copy of the in parameter

    // This function makes a copy via the copy constructor
    void add (const std::string& in) { strings.push_back (in); }

    // This function makes a copy via the move constructor
    void add (std::string&& in) { strings.push_back (std::move (in)); }
private:
    std::vector<std::string> strings;
};
```

Writing two copies of a function which both do essentially the same thing is awkward to maintain and
increases the size of the resulting binary.

We could write a single function taking a forwarding reference instead, but that has drawbacks like
- requiring the definition to be in a header
- creating versions of the function for arguments that aren't `std::strings` at all (but are
  convertible)
- failing to document the requirements on the argument type in the interface

Another option is to write a single function which accepts its argument by value:

```
void add (std::string in) { strings.push_back (std::move (in)); }
```

This version allows us to hide the function implementation, and ensures we only have a single
version of the function in the resulting binary. Also, lvalues are copied and rvalues are moved,
which is the behaviour we want. However, this approach has some performance implications.

- For any argument type, the parameter `in` must be constructed and then moved, which will always
  incur one more move than approaches which pass by (forwarding/lvalue/rvalue) reference.
  If you pass an object by value several layers up the stack, that could be a lot of extra
  moves.
- For some types (like `string`), calling a copy constructor might be able to re-use previously
  allocated memory, so passing by ref and then doing a copy assign will be faster than passing
  by value (i.e. constructing a whole new object, incurring allocation overhead) and then moving
  that object.

When is passing by value acceptable?

- When profiling has shown that the overhead of the additional moves (and other performance
  implications) is acceptable. This will likely be true for small objects (~128 bytes) which
  are trivially copyable.
- When types are copyable (i.e. not move-only). For move-only types, we only need a single
  overload taking an rvalue, so passing by value doesn't save us any duplicated code
  (although there are arguments that this style makes it harder to reason about the state and
  lifetimes of `move`d objects).
- When it's cheap to move the objects. Pass-by-value will introduce one or more additional moves,
  which we obviously want to be fast. Passing a `std::array` of move-only objects will still be
  slow.
- When we unconditionally want to make copies of the objects. If there's a chance the function
  doesn't need its own copy, passing-by-value will incur the cost of making a copy regardless.

### Guidelines

- In some cases, passing objects by value may be almost as efficient as passing by reference, and
  has the benefit that both the source code and the generated binary can be smaller. However,
  pass-by-value has potential to add significant overhead, so weigh the maintenance cost against the
  performance overheads before deciding to use pass-by-value.
- Be aware that 'copy-then-move' may be more expensive than a direct copy-assignment (via a const
  ref).

## Consider emplacement instead of insertion.

Emplacement functions like `emplace_back` allow us to avoid the creation and destruction of
temporaries when constructing objects into containers. They *should* always be at least as efficient
as their `push` variants, and sometimes will be more efficient. Unfortunately, in reality this isn't
always the case, so profiling will be required if performance is a concern. However, in the
following cases, emplacement is all but guaranteed to be faster:
- The new object is being constructed into an 'empty' location in the container (i.e. isn't
  overwriting an existing object, which might happen if `emplace`ing at an existing position)
- The argument types are different to the container type. In this scenario, no temporary will be
  created, so emplacement will likely be faster.
- The container is unlikely to reject the new value as a duplicate. The emplacement function will
  generally create a whole new node to emplace, which will be wasted work if that node is found
  to be a duplicate and discarded.

Emplacement has a couple of gotchas:
- Emplacing into a `vector<shared_ptr<T>>` by passing in a `new T` might leak, because the container
  might find out it's unable to reserve space for the new element and throw, before the pointer
  is captured in a `shared_ptr` instance. If you're using raw pointers, it's important to
  *immediately* capture them in resource-management classes. Threading raw pointers through
  function calls is a bad idea, always pass resource-managing wrappers instead.
- Emplacement functions allow calls to explicit constructors, whereas insertion functions do not,
  so they may lead to unexpected conversions. Be careful to pass the correct types to `emplace`
  functions (especially when refactoring existing code!).

### Guidelines

- Emplacement functions *should* always be at least as efficient as their insertion counterparts,
  and may be more efficient. Consider using when performance is a concern, and the danger of
  introducing unwanted type conversions is deemed acceptable.
