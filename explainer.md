# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

## Negative impls: a promise NOT to do something

By now you are probably familiar with trait impls in Rust:

```rust
// In version 1.0 of the crate `uint`:
trait UnsignedInt {
    fn increment(&mut self);
}

impl UnsignedInt for u16 { 
    fn increment(&mut self) {
        *self += 1;
    }
}

impl UnsignedInt for u32 { 
    fn increment(&mut self) {
        *self += 1;
    }
}

impl UnsignedInt for u64 { 
    fn increment(&mut self) {
        *self += 1;
    }
}
```

### Impls are forever

Of course, a trait implementation provides the definition of a trait's methods, but it also provides a semver promise: future releases of the same crate cannot remove an implementation without a major version bump.

### Minor versions can add new impls

It is, however, legal for crates to add a **new** implementation in a minor version (with some caveats[^blanket]). So, if we had the trait above, one could add an impl for `u8`:

```rust
// In version 1.1 of the crate `uint`:

impl UnsignedInt for u8 {
    fn increment(&mut self) {
        *self += 1;
    }
}
```

[^blanket]: It is not permitted to add a *blanket impl* in a minor version. See [RFC 2451](https://rust-lang.github.io/rfcs/2451-re-rebalancing-coherence.html) for more details.

### Implication: Downstream crates cannot rely on something *not* being implemented

A consequence of the fact that crates are permitted to add new impls is that Rust code cannot rely on impls *not* being implemented, since impls that don't exist today may come to exist in the future. The primary place that this comes up today is with coherence. Consider this example:

```rust
// In another crate `foo` that depends on `uint`:

trait FooTrait { }
impl<T: uint::UnsignedInt> FooTrait for T { }
impl FooTrait for u128 { }
```

At the moment, `uint` does not implement `UnsignedInt` for `u128`, so one could imagine that this code would compile. However, if we were to allow that, then the crate `foo` might be broken by some future release of `uint` which adds an impl for `u128`. In particular, that would cause there to be two overlapping impls for `FooTrait`, and this is not allowed. That is a problem, since adding a new impl is meant to be a non-breaking change. For this reason, Rust rejects the impls above, unless they appear within the crate `uint` itself (the idea is that, within the crate, if you were to add the new impl, you would get a compilation error locally).

In the `foo` example, Rust's restriction makes sense. It seems *highly likely* that adding not having `u128` implement `UnsignedInt` was just an oversight. But other examples are less clear. Imagine this crate, `bar`:

```rust
// In yet another crate `bar` that depends on `uint`:

trait BarTrait { }
impl<T: uint::UnsignedInt> BarTrait for T { }
impl BarTrait for String { }
```

Again, the compiler rejects this code, this time because it is concerned that some future release of `uint` might add an impl of `UnsignedInt` for `String` -- but that seems very unlikely to happen, since string is not an integer of any kind. (Although not *impossible* -- maybe we'd like to be able to treat `format!("123")` as an integer?) How are we to distinguish these two cases?

### Enter: Negative impls

The idea of negative impls is to make it possible for a crate to promise *not* to implement something. Given this extension, the `uint` crate could add an impl like the following:

```rust
// In uint v1.2:

impl !UnsignedInt for String { }
```

Once added, removing a negative impl is a breaking change, just as with a positive impl. Effectively, the negative impl is a promise by `uint` never to implement `UnsignedInt` for `String`. Given this negative impl, it is then fine for `bar` to add impls that rely on `String` not implementing `UnsignedInt`, so that example would compile.


