- Feature Name: `min_impl_trait_in_fn_trait_bound_return`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Permit `impl Trait` in return position of `Fn(…)`-like trait bounds, expanding
to a trait bound on the associated type.

# Motivation
[motivation]: #motivation

Consider
[`slice::sort_by_key`](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by_key)
from the standard library:

```rust
impl<T> [T] {
    pub fn sort_by_key<K, F>(&mut self, mut f: F)
        where F: FnMut(&T) -> K, K: Ord
    {
        self.sort_by(|a, b| f(a).cmp(&f(b)))
    }
}
```

`K` cannot depend on `'a`, which means that uses of this method one might expect
to work, do not:

```rust
// Example from https://github.com/rust-lang/rust/issues/34162

struct Client(String);

impl Client {
    fn key(&self) -> &str {
        &self.0
    }
}

fn sort_clients(clients: &mut [Client]) {
    // error: lifetime may not live long enough
    clients.sort_by_key(|c| c.key());

    // This works fine:
    clients.sort_by(|a, b| a.key().cmp(&b.key()));
}
```

It should be possible to write a version of `sort_by_key` that lacks this
restriction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

It is now possible, in edition 2024 and later, to use `impl Trait` in the return
type of `Fn`-like trait bounds. These capture all generic parameters in scope,
including higher-ranked lifetimes. However, `use<…>` is not yet supported in
this position.

With the help of this feature, we could fix the issue with `sort_by_key` by
rewriting it like so:

```rust
impl<T> [T] {
    pub fn sort_by_key<F>(&mut self, mut f: F)
        where F: for<'a> FnMut(&'a T) -> impl Ord,
    {
        self.sort_by(|a, b| f(a).cmp(&f(b)))
    }
}
```

We can also make the higher-ranked lifetime implicit:

```rust
impl<T> [T] {
    pub fn sort_by_key<F>(&mut self, mut f: F)
        where F: FnMut(&T) -> impl Ord,
    {
        self.sort_by(|a, b| f(a).cmp(&f(b)))
    }
}
```

(Actually making this change in the standard library is not part of this RFC.
Removing the `K` generic parameter is a breaking change, and would have to be
done over an edition.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In edition 2024 and above, it is now permissible to use `impl Trait` syntax in
the return type position of `Fn`-like bounds. (The `Fn`-like traits are `Fn`,
`FnMut`, `FnOnce`, `AsyncFn`, `AsyncFnMut`, and `AsyncFnOnce`.). This desugars
to a trait bound on the `Output` associated type. For example, `T: Fn() -> impl
Send` desugars to `T: Fn<(), Output: Send>`, and `T: Fn(&()) -> impl Send`
desugars to `T: for<'a> Fn<(&'a (),), Output: Send>`.

The new syntax is accepted in bounds on generic parameters of functions and
methods:

```rust
impl<T> [T] {
    pub fn sort_by_key<B, F>(&mut self, mut f: F)
        where F: FnMut(&T) -> impl Ord,
    {
        // ...
    }
}
```

It is also accepted in bounds in other contexts:

```rust
trait Foo<F>
where
     F: FnMut(&T) -> impl Ord,
{}

struct Bar<F: FnMut(&T) -> impl Ord>(F);

// ... etc.
```

This syntax is also accepted in argument-position `impl Fn()`:

```rust
impl<T> [T] {
    pub fn sort_by_key<B, F>(&mut self, mut f: impl FnMut(&T) -> impl Ord)
        where F: FnMut(&T) -> impl Ord,
    {
        // ...
    }
}
```

`use<…>` is not currently supported in this position; that is a future
extension. Support for editions 2021 and below is also left as a future
extension.

Also not yet supported is nesting the `impl Trait` inside a generic type, e.g.
`impl FnMut(&T) -> Option<impl Ord>`.

When several traits are combined using `+`, parentheses are required to
disambiguate. `T: Fn() -> impl Send + Sync` is not accepted, but  `T: Fn() ->
(impl Send + Sync)` and `T: (Fn() -> impl Send) + Sync` are.

# Drawbacks
[drawbacks]: #drawbacks

## Doesn’t immediately allow us to fix stable functions in the standard library

As mentioned previously, this change is not on its own sufficient to fix
`std::slice::sort_by_key`. However, there are several standard library methods
which are still unstable, and could therefore benefit immediately:
`cmp::minmax_by_key()` and `slice::partition_dedup_by_key()`. And, of course,
this feature would also benefit libraries in the ecosystem.

## Higher-ranked lifetime capturing behavior is inconsistent with `impl Trait` in argument-position associated type bounds

In argument-position `impl for<'a> Foo<'a, Item = impl Send>`, the `impl Send`
does not capture the higher-ranked `'a` lifetime. This RFC proposes to deviate
from that behavior for `Fn` bounds in return types.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Only edition 2024, no `use<…>` support

These restrictions are meant to ensure this feature desugars to a simple trait
bound on an associated type, keeping the implementation simple. These
restrictions can be lifted in the future. (The restriction to edition 2024 is
meant to ensure consistency with the RPIT lifetime capture rules of each
edition.)

## Versus a different syntax

This RFC introduces no new syntax that the compiler doesn’t already know how to
parse. Additionally, it’s consistent with RPIT.

# Prior art
[prior-art]: #prior-art

None known.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Closure type inference

The trickiest part of implementing this RFC will be likely be ensuring that
closure signatures are properly inferred. For the higher-ranked case, this may
have to be delayed until after the MVP, requiring users to rely on
[`closure_lifetime_binder`](https://rust-lang.github.io/rfcs/3216-closure-lifetime-binder.html).

# Future possibilities
[future-possibilities]: #future-possibilities

- Support for `use<…>`
- Support in editions prior to 2024
- Support for nesting inside generic types
- Support in the return types of `fn(...) -> ...` function pointer types (remove [E0562](https://doc.rust-lang.org/nightly/error_codes/E0562.html))
- Support in the return types of `dyn Fn(...) -> ...` trait object types (remove [E0657](https://doc.rust-lang.org/nightly/error_codes/E0657.html))
