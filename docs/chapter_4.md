# Smart Pointers

Raw pointers are bad (especially owning raw pointers).

- The declaration doesn't indicate whether it points to an element or an array
- The declaration doesn't indicate whether the pointer is owning or non-owning
- Doesn't indicate whether owning pointers should be deallocated with `delete`
  or `delete[]` or `free` or some other mechanism
- Difficult to avoid double-deletes, dangles, and leaks with owning raw
  pointers

One way to counteract these shortcomings is with smart pointers. The smart
pointers provided in the stl are `unique_ptr`, `shared_ptr`, and `weak_ptr`
(`auto_ptr` is deprecated).

## Use `unique_ptr` for exclusive-ownership resource management

`unique_ptr` typically introduces no overhead over (correct) use of a
single-ownership raw pointer.

A common use-case is as a return type for factory functions which might return
an instance of any number of derived types which share a common base class.
Well-suited to this use-case because it converts easily to a `shared_ptr`
(converting in the opposite direction is tricker).  Another use is to hold an
implementation class as part of the Pimpl idiom.

A non-null `unique_ptr` exclusively owns what it points-to. `unique_ptr` is a
move-only type. By default, a non-null `unique_ptr` will destroy what it
points-to by calling `delete`, although it is possible to customise this
behaviour.

If custom deletion behaviour is necessary, a functor can be passed to the
`unique_ptr` instance upon construction, which will be called when the time has
come to destroy the owned resources.

```
auto doCustomDelete = [] (auto* ptr) { /*...*/ };
auto ptr = std::unique_ptr<MyType, decltype (doCustomDelete)> { new MyType{},
                                                                doCustomDelete };
```

> Side note: returning a `unique_ptr` with a custom deleter from a function
  seems like a good use for `auto` return type deduction. If the deleter is a
  function-local lambda, it won't have a type which is visible _or_ utterable.
  e.g.
  ```
  auto makeInstance() {
    auto doCustomDelete = [] (auto* ptr) { /*...*/ };
    auto ptr = std::unique_ptr<MyType, decltype (doCustomDelete)> { new MyType{},
                                                                    doCustomDelete };
  }
  ```

Note that if a custom deleter functor is used, it must be stored alongside the
managed pointer.

- When using a function pointer as the deleter, the pointer itself will be
  stored alongside the managed pointer, so the `unique_ptr` object itself will
  be the size of two pointers.
- If instead a function object is used as the deleter, the increase in size
  will be equal to the size of the function object. Notably, if the function
  object has no state, then no space overhead will be incurred.  This is
  accomplished using the EBO from `tuple` in the implementations I've seen.

Therefore, in many cases, a lambda/function-object will be preferable to a
plain function pointer, even if the function object just forwards through to a
normal function.

`unique_ptr` also has a specialisation for arrays, which allows indexing and
disallows dereferencing via `*` and `->`. Probably only useful for interfacing
with C APIs which return heap-allocated arrays.

### Guidelines

- Prefer `unique_ptr` over `shared_ptr` in interfaces where possible.
  Conversions from `unique_ptr` to `shared_ptr` are easier than converting in
  the opposite direction.
- Prefer function objects or lambdas over plain function pointers as custom
  deleter functions for `unique_ptr` as they introduce less space overhead.
- Avoid creating `unique_ptr` instances from raw pointers unless you really
  need a custom deleter.

## Use `shared_ptr` for shared-ownership resource management

Many `shared_ptr` instances may point to the same memory. There is no special
owner. Instead, when the final `shared_ptr` pointing to an object stops
pointing there, the object is destroyed. This is achieved by storing a
reference count for each object, and destroying the object when the count
reaches zero.

`shared_ptr` is more expensive than `unique_ptr`:

- `shared_ptr` is ~twice the size of a raw pointer, because it needs to store
  a pointer to the control block too
- That control block has to be allocated, as well as the managed object
- Refcount modifications must be atomic, as `shared_ptr`s may be shared across
  threads

Like `unique_ptr`, `shared_ptr` supports custom deleters, but the deleter
does not affect the type or size of the object. The deleter is stored in the
control block, alongside other book-keeping info like the refcount, weak count,
and custom allocator.

If the same raw pointer is passed to the constructors of two `shared_ptr`
instances, both instances will set up a new control block for that pointer,
leading to UB, double-deletes, much pain. Prefer to use `make_shared`, or if
you *must* use a `shared_ptr` constructor directly (to supply a custom
deleter), put the `new` expression directly in the constructor invocation.

```
// No chance for others to capture the new Foo, delete it, etc.
std::shared_ptr<Foo> ptr { new Foo, customDeleter };
```

If you want to share ownership of an object from within a member function of
that object, use `std::enable_shared_from_this`.

```
std::vector<std::shared_ptr<Foo>> collection;

void Foo::storeInGlobalCollection() {
  // Bad, will create a duplicate control block for this object
  collection.emplace_back (this);
}

void Foo::storeInGlobalCollection() {
  // Good, will use the existing control block
  collection.emplace_back (shared_from_this());
}
```

### Guidelines

- Think carefully before using `shared_ptr`. It's more costly than
  `unique_ptr`, (bigger, uses atomic ops) and it can't be converted to a
  `unique_ptr` later. Once `shared_ptr` has been introduced, it's difficult to
  change it to a more restricted ownership strategy.
- Avoid creating `shared_ptr` instances from raw pointers, including `this`.

## Use `weak_ptr` for shared-ownership pointers that can dangle

Weak pointers are created from `shared_ptr` instances. They have a `lock`
mechanism to check whether the originally-pointed-to object is still alive, and
if so, to access that object.

```
auto original = std::make_shared<Foo>();
auto weak = std::weak_ptr<Foo> (original);
...
// `locked` is a `shared_ptr`
// If the object managed by `original` is still alive, `locked` will also point
// to that object.
// Otherwise, `locked` will be null
auto locked = weak.lock();
```

> How does this compare to juce's WeakReference? I suspect that `weak_ptr`
  is safer, because `lock` extends the lifetime of the pointed-to object, so
  dangling/lifetime issues are less likely. Just a hunch though.

### Guidelines

- Use `weak_ptr` for shared-ownership pointers that can dangle.
    - Especially in multithreaded contexts

## Prefer `make_unique` and `make_shared` to direct `new`

Using `new` requires you to repeat the typename, while the `make` functions do
not.

The `make` functions are also exception-safe whereas `new` is not. Depending
on evaluation order, an exception may be thrown before the `new`ed object is
captured in a smart pointer, leading to leaks.

```
// May run `new Foo`, then `throwException`, then the `unique_ptr` constructor
doSomething (std::unique_ptr<Foo> (new Foo), throwException());
```

`make_shared` is able to allocate a single block containing both the managed
object and the control block, which will generally be more efficient than doing
two separate allocations.

Unfortunately, the `make` functions

- don't accept custom deleters
- do not allow calling `initializer_list` constructors concisely
  - although an explicit `initializer_list` can be created and passed to the
    function if necessary

Additionally,

- `make_shared` doesn't work as expected with class-specific `new/delete`
  because it needs to allocate extra space for the control block
- the memory allocated by `make_shared` may live for longer than the managed
  object, if there are weak pointers still using the control block, which may
  be unacceptable if the managed object is very large


### Guidelines

- Prefer `make` functions unless you have a good reason not to (e.g. you need
  to use a custom deleter)
- If you cannot use a `make` function for some reason, ensure that the smart
  pointer constructor is separated into its own statement, to improve exception
  safety.

## When using Pimpl, define special member functions in the implementation file

The full definition of the type managed by a `unique_ptr` must be known at the
point that the `unique_ptr` destructor is instantiated. If you don't declare a
destructor, the compiler will generate an inline one at a point where the impl
class hasn't yet been defined, which will trigger a compiler error. To fix
this, put a destructor declaration in the header, and use a defaulted
destructor definition in the implementation file. The same goes for move ops.

### Guidelines

- Put (defaulted) definitions of special members of Pimpl classes in the
  implementation file.
