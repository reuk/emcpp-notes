# `auto`

## Prefer auto to explicit type declarations

- Auto forces initialisation.
  ```
  int x; // oops, uninitialised
  ```

- Auto is often shorter than other type names, especially inner/dependent classnames.
- Auto can represent unutterable types (e.g. lambda types).
- Using auto to divine the static type of an expression for us will often be cheaper (smaller,
  faster) and more correct than converting the expression to some other type: Initialising a
  std::function from a lambda will be bigger than the plain lambda, and introduces virtual call
  overhead, and thwarts inlining.
- Auto is often more correct than explicitly naming a type: using `auto` rather than `unsigned` to
  capture the result of vector::size avoids type conversions that in turn may introduce errors.
  - Also when iterating collections with subtle element types (pairs with const members).

## Use the explicitly typed initializer idiom when auto deduces undesired types

Sometimes using the *exact* type of some expression introduces errors.  In the case of `auto b =
getVecOfBool()[i];`, `b` is not a bool, it's some object which contains a reference back to the
vector. If the vector goes out of scope before we read from `b`, then we'll encounter UB.

Libraries that rely on expression templates (I think Eigen does this) also do not interact well with
`auto` - they need types to be mentioned explicitly in order to concretise/evaluate the expression.

The solution: put the type on the RHS. Meyers suggests using `static_cast` but that looks pretty
ugly. A decent alternative from Herb Sutter's blog post on Almost Always Auto is just to do
```
auto x = MyType { some.complex.expression() };
```

Putting the type on the RHS is preferable to the LHS, because you can still use `auto` to force
initialisation, and it looks more uniform/consistent.
