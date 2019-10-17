# `std::optional`

Sometimes we need a special sentinel value for an object that specifies a special 'null' state for
that object. Sometimes, special values like `-1`, `nullptr`, or `std::numeric_limits<int>::max()`
can be used (see `string::find`, which returns `npos` on failure). When using this technique, care
must be taken not to accidentally use the special value in a context where a 'non-sentinel' value
was expected.

An alternative approach is to add a bool flag to the object that specifies whether or not the
object is valid, which is the approach taken by `optional`. This approach has the advantage of
retaining value semantics, whereas using a nullable `unique_ptr` (for example) would require
pointer semantics and heap allocation.

## Use cases

- Indicating that a computation was unsuccessful, but this is not an error:
    - Searching for a particular item in a collection might return `nullopt` if the value
      wasn't present.
    - Computing a location to display something onscreen might return `nullopt` if the entity lies
      outside the viewport.
- Performing lazy-loading.
    - We can leave 'space' in an object for a member object that can be initialised on demand, but
      which doesn't have to be created when the outer object is constructed.
- Controlling lifetimes of RAII object.
    - Perhaps we have some RAII type like a `future` or a `Connection` that we want to start and
      end at will. We can store an instance of that type in an `optional`, and `emplace` or
      `reset` the instance to trigger the constructor/destructor behaviour on demand.
- Passing optional parameters.
    - Some functions might have extra functionality that they can perform if they are given extra
      parameters.

## Constructing optionals

Optionals can be:
- Initialised as empty
- Created directly with a value, using deduction guides or `operator=`
- Created directly from multiple parameters using `std::in_place`
- Moved/copied from another optional

### `in_place` construction

This is mostly useful for constructing non-copyable non-moveable types into an optional,
especially in contexts where `auto` deduction cannot be used (class data member declarations).

```
struct Inner
{
    Inner (std::string s, int i) {}

    Inner (const Inner&) = delete;
    Inner (Inner&&) noexcept = delete;

    Inner& operator= (const Inner&) = delete;
    Inner& operator= (Inner&&) noexcept = delete;
};

struct Outer
{
    // If we used `make_optional<Inner>` here instead, we'd be repeating the type `Inner`
    // unnecessarily
    std::optional<Inner> optionalInner { std::in_place, "foo", 42 };
};
```

In other contexts, C++17 guarantees RVO, so it should be possible to use `make_optional` even
with non-copyable non-moveable types:

```
auto optionalInner = std::make_optional<Inner> ("foo", 42);
// The book says that non-copyable non-moveable types require `in_place` but this seems to build...
auto opt = std::make_optional<std::mutex>();
```

### Returning optional values

Optionals default-construct as empty, and can be implicitly converted from an instance of their
wrapped type, so it's often simplest to do something like this:

```
std::optional<int> computeValue (Input input)
{
    if (input.valid())
        return input.getIntValue(); // Mandatory RVO applies here, don't wrap this in `{}`!

    return {};
}
```

Note that wrapping the returned value with `{}` will force a copy, so make sure to omit the
braces if you want to avoid unnecessary temporaries.

### Accessing optionals

- `operator*` and `operator->` act as you would expect, UB if there's no value
- `value()` throws if there's no value
- `value_or(def)` returns the wrapped value if available, or `def` otherwise

Access can be quite clean with initialised if statements:

```
if (auto maybeValue = compute(); maybeValue != std::nullopt)
    // do something with *maybeValue
```

### Other features

The contained object can be replaced or destroyed using `emplace`, `reset`, `swap`, and `operator=`.

Optionals can be compared using `operator<` etc., and `nullopt` will always compare less-than a
'real' value. (Think carefully before actually using this functionality!)

### Warnings

Optional objects may double the required memory for a wrapped object, due to alignment rules.
For single optionals, this probably doesn't matter, but if you have lots of optionals in a class,
or are using very large optional objects, there may be more efficient alternatives. Profile before
writing anything custom, though!
