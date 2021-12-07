# Always applicable impls

Like the `Drop` trait, negative impls are currently limited to being "always applicable", which means that they must apply to *all* instances of a given type. As an example, consider the following type `Foo`:

```rust
struct Foo<T: Eq> { t: T }
```

It is legal to implement a negative trait for `Foo`, so long as it applies to all `T: Eq`:

```rust
impl<T: Eq> !Clone for Foo<T> { }
```

It is not legal to implement a negative trait that only applies to some `T`:

```rust
impl<T: Eq + Ord> !Clone for Foo<T> { }
```

## Why require always applicable?

This rule is inspired by limitations of the current implementation, particularly when it comes to auto traits. In particular, the way that auto traits are currently implemented, negative impls always apply to all instances of the type. Therefore, this impl for example would actually mean that `Foo<T>` is never `Send`:

```rust
impl<T: Eq + Ord> !Send for Foo<T> { }
```

Under the "always applicable" rule, this impl is simply illegal. Once the implementation for auto traits has improved, we could lift this rule.