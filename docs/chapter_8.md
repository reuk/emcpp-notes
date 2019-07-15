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

For any argument type, the parameter `in` must be constructed and then moved, which will always
incur one more move than approaches which pass by (forwarding/lvalue/rvalue) reference.
