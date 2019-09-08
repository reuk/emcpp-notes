# Lambdas

- A 'lambda expression' is the expression of the lambda in source code:
  ```
  auto check = [] (int val) { return 0 < val && val < 10; };
  ```
- A 'closure' is the function object that exists at runtime, which holds
  copies/references of the captured objects.
- A 'closure class' is the class from which the closure itself is intantiated.

## Avoid default capture modes

Default by-reference capture can lead to dangling references. If the lifetime
of a closure exceeds the lifetime of a variable captured by-reference, the
reference in the closure will dangle. By explicitly listing the names of
captured objects, it's easier to check that those objects have appropriate
lifetimes.

Default by-value capture *may* also lead to dangling references. If you capture
a pointer by value, other code may still invalidate that pointer, causing it to
dangle. A common case is when capturing `this` by value. Again, by explicitly
naming `this`, we are reminded to check that the closure can't outlive `this`.

### Guidelines

- Avoiding default capture modes can help remind us of object liftimes,
  although it's still important to be careful when capturing anything by
  reference or pointer (especially `this`).

## Use init capture to move objects into closures

In C++14 we can capture the result of any expression, including expressions
involving `std::move`.
```
auto vec = std::vector { 1, 2, 3, 4, 5 };
auto func = [vec = std::move (vec)] (auto item) mutable
{
  vec.push_back (item);
  return vec;
};
```
Remember that the `operator()` of a closure class is `const` by default. To
allow mutation of non-reference captured objects, it's necessary to mark the
lambda expression `mutable`.

### Guidelines

- Use generalised lambda capture to move objects into closures when it makes
  sense to do so.

## Use `decltype` on `auto&&` parameters to `std::forward` them

If you write a lambda that calls through to an inner function which treats
lvalues and rvalues differently, then the lambda should perfect-forward its
arguments.

In C++14, perfect forwarding in lambdas looks like this:
```
auto f = [] (auto&&... params)
{
  return process (std::forward<decltype (params)> (params)...);
};
```

(There's probably a bunch of code I've written that gets this wrong.)

In C++20, this gets... I don't want to say 'nicer', but it's good if you really
like a wide variety of bracket types in your expressions.
```
auto f = [] <typename... Ts> (Ts&&... ts)
{
    process (std::forward<Ts> (ts)...);
};
```

### Guidelines

- Use `std::forward<decltype (params)>` on `auto&&` parameters when you need
  perfect forwarding inside a lambda expression.

## Prefer lambdas to `std::bind`

Not going go in-depth on this because only `fxpansion-synth` and `3rd_party`
code even use `std::bind`.

### Guidelines

- Lambdas are more readable, more expressive, and potentially more efficient
  than using `std::bind`, so lambdas should be preferred.
