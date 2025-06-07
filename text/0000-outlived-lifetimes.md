- Feature Name: `outlived_lifetimes`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduce a notation, `'<…>`, for the longest lifetime outlived by all members
of a set of types and/or lifetimes.

# Motivation
[motivation]: #motivation

## Avoiding early-bound lifetimes

The signature of `Box::leak()` contains an early-bound lifetime parameter:

```rust
impl<T: ?Sized> Box<T> {
    fn leak<'a>(b: Box<T>) -> &'a mut T { … }
}
```

It would be nice if this function (and other functions with a similar signature)
could be defined without this early-bound parameter.

Similarly, in the following function, we are forced to introduce an early-bound
lifetime:

```rust
trait Trait {}

#[derive(Clone, Copy)]
struct MultiRef<'a, 'b>(*mut &'a str, *mut &'b str);

impl Trait for MultiRef<'_, '_> {}

// Doesn't work
fn foo<'a, 'b>(mr: MultiRef<'a, 'b>) -> Box<dyn Trait + '??> {
    //                                                  ^^^ ERROR what do we put here?
    Box::new(mr)
}

// Instead, we must do:
fn foo<'a: 'c, 'b: 'c, 'c>(mr: MultiRef<'a, 'b>) -> Box<dyn Trait + 'c> {
    Box::new(mr)
}
```

### Expressing certain function pointer types

This forced early binding has bad consequences. For instance, there’s no way to
turn `foo` into a function pointer without loss of generality:

```rust
trait Trait {}

struct MultiRef<'a, 'b>(*mut &'a str, *mut &'b str);

impl Trait for MultiRef<'_, '_> {}

fn foo<'a: 'c, 'b: 'c, 'c>(mr: MultiRef<'a, 'b>) -> Box<dyn Trait + 'c> {
    Box::new(mr)
}

fn call_twice(mr1: MultiRef<'_, '_>, mr2: MultiRef<'_, '_>) {
    let fnptr: fn(_) -> _ = foo;
    fnptr(mr1);
    fnptr(mr2); // ERROR lifetime may not live long enough
}
```

### Expressing certain associated types

Trait objects have the same issue as function pointers; there is no way to turn
`leak_tuple` into a `dyn Fn(…)` without loss of generality either.

## Generic self-referential structs

In current Rust, it’s difficult to properly define generic self-referential
structs, even with the help of `unsafe`. Using references imposes unnecessary
restrictions:

```rust
struct DataAndView<T> {
    x: Box<[T]>,
    element: &'static T,
           // ^^^^^^^ This forces us to add a `T: 'static` bound
           //         But what could we write instead?
}
```

To avoid these issues, it’s necessary to fall back to raw pointers, which comes
with its own problems:

```rust
struct DataAndView<T> {
    x: Box<[T]>,
    element: NonNull<T>,
}

unsafe impl<T: Send + Sync> Send for DataAndView<T> {}
unsafe impl<T: Sync> Sync for DataAndView<T> {}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, let’s review how lifetime bounds work. [From the
Reference](https://doc.rust-lang.org/reference/trait-bounds.html#r-bound.lifetime):

> ## Lifetime bounds
>
> Lifetime bounds can be applied to types or to other lifetimes.
>
> The bound `'a: 'b` is usually read as `'a` *outlives* `'b`. `'a: 'b` means
> that `'a` lasts at least as long as `'b`, so a reference `&'a ()` is valid
> whenever `&'b ()` is valid.
>
> ```rust
> fn f<'a, 'b>(x: &'a i32, mut y: &'b i32) where 'a: 'b {
>     y = x;                      // &'a i32 is a subtype of &'b i32 because 'a: 'b
>     let r: &'b &'a i32 = &&0;   // &'b &'a i32 is well formed because 'a: 'b
> }
> ```
>
> `T: 'a` means that all lifetime parameters of `T` outlive `'a`. For example,
> if `'a` is an unconstrained lifetime parameter, then `i32: 'static` and
> `&'static str: 'a` are satisfied, but `Vec<&'a ()>: 'static` is not.

The syntax `'<…>`, where `…` is a list of zero or more comma-separated types or
lifetimes, represents the longest lifetime that is outlived by all these
parameters.

For example:

- `'<>` is `'static`
- `'<'a>` is `'a`
- `'<'a, 'b>` is the intersection of `'a` and `'b`
- `'<'a, 'b, 'c>` is the intersection of `'a`, `'b`, and `'c`
- `'<i32>` is `'static`
- Given `struct Foo<'a, 'b>(&'a i32, &'b i32);`, `'<Foo<'a, 'b>>` is `'<'a, 'b>`
- Given `struct Foo<'a, 'b>(&'a i32, &'b i32);`, `'<Foo<'a, 'b>, 'c>` is `'<'a,
  'b, 'c>`

These “outlived lifetimes” can be used in the same positions as any other
lifetimes.

`Box::leak()` could be rewritten like so:

```rust
impl<T: ?Sized> Box<T> {
    fn leak(b: Box<T>) -> &'<T> mut T { … }
}
```

(Making this change for the actual `Box::leak` in the standard library would
have to be done over an edition, and is not part of this RFC.)

`foo()` from the motivation section can be rewritten like so:

```rust
fn foo<'a, 'b>(mr: MultiRef<'a, 'b>) -> Box<dyn Trait + '<'a, 'b>> {
    Box::new(mr)
}
```

This new function can be converted into a function pointer of type:

```rust
type FnPtr = for<'a, 'b> fn(MultiRef<'a, 'b>) -> Box<dyn Trait + '<'a, 'b>>;
```

As well as into a trait object of type:

```rust
type DynFn = dyn for<'a, 'b> Fn(MultiRef<'a, 'b>) -> Box<dyn Trait + '<'a, 'b>>;
```

The self-referential struct from the motivation section can now be written as:

```rust
struct DataAndView<T> {
    x: Box<[T]>,
    element: &'<T> T,
}
```

(Ideally, one or both of this struct’s fields would be marked as `unsafe`, with
the help of [RFC 3458](https://github.com/rust-lang/rfcs/pull/3458).)

## Replacement for `use<…>`

As a bound in RPIT, outlived lifetimes can be used in place of `use<…>`:

```rust
// The following are equivalent

fn foo<'a, 'b, T>(a: &'a (), b: &'b (), c: T) -> impl Sized + '<'a, T> {
    (a, c)
}

fn foo<'a, 'b, T>(a: &'a (), b: &'b (), c: T) -> impl Sized + use<'a, T> {
    (a, c)
}
```

Having been obsoleted thereby, `use<…>` syntax should be removed in the next
edition.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The syntax for an outlived lifetime is `'` followed by a [_GenericArgs_ syntax
element](https://doc.rust-lang.org/reference/paths.html#r-paths.expr.syntax).
This element can contain zero or more lifetimes or types, in any order. It
represents the unique lifetime satisfying the following two properties:

1. Outlived by all its arguments
2. Outlives all lifetimes satisfying (1)

Outlived lifetime arguments can appear anywhere other lifetime arguments can.

Outlived lifetimes may contain inferred generic parameters (`_` or `'_`) in
contexts where those are permitted. However, the compiler may emit a warning in
such cases, if the entire outlived-lifetime construct could be replaced by `'_`.

Outlived lifetimes are covariant with respect to their parameters.

## As a bound in RPIT

When used as a bound in RPIT, outlived lifetimes capture all type parameters
used inside them. In other words, they have the same semantics as `use<…>`, and
therefore entirely replace that feature. Having become obsolete, `use<…>` is
removed from the next edition.

## In type declarations

As stated previously, `'<…>` is covariant:

```rust
/// Covariant in `T`
struct Foo<T>(inner: &'<T> T);

/// Invariant in `T`
struct Foo<T>(&'<T> mut &'<T> T);

/// Contravariant in `T`
struct Foo<T>(fn(&'<T> T));
```

For the purposes of
[drop-check](https://doc.rust-lang.org/nomicon/phantom-data.html#generic-parameters-and-drop-checking),
`'<T>` acts like `PhantomData<*const T>`, not `PhantomData<T>`.

Generic parameters used only self-referentially are considered unused; this is
unchanged from current Rust:

```rust
struct Foo<'a>(&'<Foo<'a>> ()); // ERROR `'a` is unused
```

With this feature, it’s possible to define types whose lifetime parameters
cannot be uniquely inferred from their field types.

```rust
// Impossible to infer `'a` and `'b`
// from `'<'a, 'b>`
struct Foo<'a, 'b>(&'<'a, 'b> ());
```

However, this situation can already occur in Rust today. The compiler makes an
angelic choice in such cases:

```rust
trait Gatt {
    type Gat<'a>
    where
        Self: 'a;
}

impl Gatt for () {
    type Gat<'a> = ();
}

#[derive(Clone, Copy)]
struct Foo<'lt>(<() as Gatt>::Gat<'lt>);

fn requires_static(_: Foo<'static>) {}

fn requires_not_static<'a>(_: Foo<'a>, _: &'a ()) {}

fn main() {
    let foo = Foo(());

    // Angelic choice:
    // commenting out either one of these two calls allows the program to compile
    requires_static(foo);
    requires_not_static(foo, &foo.0);
}
```

That being said, *type* parameters that are used only inside `'<…>` are
considered unused, which is an error:

```rust
struct Foo<T>(&'<T> ()); // ERROR `T` is unused
```

## In implementations

[From the description of
E0207](https://doc.rust-lang.org/stable/error_codes/E0207.html):

> A type, const or lifetime parameter that is specified for `impl` is not
> constrained.
>
> Erroneous code example:
>
> ```rust
> struct Foo;
>
> impl<T: Default> Foo {
>     // error: the type parameter `T` is not constrained by the impl trait, self
>     // type, or predicates [E0207]
>     fn get(&self) -> T {
>         <T as Default>::default()
>     }
> }
> ```
>
> Any type or const parameter of an `impl` must meet at least one of the
> following criteria:
>
> - it appears in the _implementing type_ of the impl, e.g. `impl<T> Foo<T>`
> - for a trait impl, it appears in the _implemented trait_, e.g. `impl<T>
>   SomeTrait<T> for Foo`
> - it is bound as an associated type, e.g. `impl<T, U> SomeTrait for T where T:
>   AnotherTrait<AssocType=U>`

For the purposes of this analysis, using a generic parameter inside `'<…>` does
*not* constrain that parameter.

## In function pointer types

Outlived lifetimes may be used in higher-ranked function pointer types. However,
similar to the ristriction from the previous section, a higher-ranked liftetime
must appear somewhere in the argument types *ouside* an `'<…>` in order to be
considered constrained:

```rust
type Foo = for<'a,'b> fn(&<'a, 'b> ()) -> &'a (); // ERROR[E0581] `'a` is unconstrained
type Foo = for<'a> fn(&<'a> ()) -> &'a (); // ERROR[E0581] `'a` is unconstrained
type Foo = for<'a> fn(&'a ()) -> &'a (); // Ok
type Foo = for<'a,'b> fn(&'a (), &'b (), &'<'a, 'b> ()) -> &'<'a, 'b> (); // Ok
```

## In `dyn` types

The same rules apply: a higher-ranked liftetime must appear somewhere in the
trait’s generic parameters ouside* an `'<…>` in order to be considered
constrained:

```rust
type Foo = for<'a,'b> dyn Trait<&<'a, 'b> (), Assoc = &'a ()>; // ERROR[E0582] `'a` is unconstrained
type Foo = for<'a> dyn Trait<&<'a> (), Assoc = &'a ()>; // ERROR[E0582]`'a` is unconstrained
type Foo = for<'a> dyn Trait<&'a (), Assoc = &'a ()>; // Ok
type Foo = for<'a,'b> dyn Trait<&'a (), &'b (), &'<'a, 'b> (), Assoc =  &'<'a, 'b> ()>; // Ok
```

### In `impl` blocks

The same rules apply: a higher-ranked liftetime must appear somewhere in the
trait’s generic parameters ouside* an `'<…>` in order to be considered
constrained:


# Drawbacks
[drawbacks]: #drawbacks

- Adds complexity to the language and borrow checker.
- There are some cases in Rust today where Rust considers a pair of types to be
distinct, despite being mutual subtypes. For example, `for<'a, 'b> fn(&'a (),
&'b ())` is a different type from `for<'a> fn(&'a (), &'a ())`. This quirk can
lead to some very confusing borrow-check errors. This feature could expose that
problem in more cases, depending on how much normalization the compiler
performs.
- The meaning of `'<…>` is quite subtle. `T: 'a` is already a common source of
confusion for new Rust users; this feature could make that problem worse.
- Replacing the `+ use<…>` feature so soon after introducing it will cause
churn, annoying users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Versus [`unsafe<…>` binders](https://hackmd.io/@compiler-errors/HkXwoBPaR)

To the degree that it enables defining ADTs with lifetimes the compiler can’t
understand (e.g., self-references), the “unsafe binders” proposal serves as an
alternative to this feature. Being more narrowly targeted, it has better
ergonomics for its specific use-case, but does not address all the use-cases of
this RFC.

There is no conflict between the two features, and Rust could potentially adopt
both.

## Versus a different syntax

# Prior art
[prior-art]: #prior-art

[RFC 3617](./3617-precise-capturing.md) is essentially a subset of this feature,
with a slightly different syntax.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- As mentioned earlier in the “drawbacks” section, it’s unclear how much
normalization the compiler can feasibly apply to function pointer types
containing outlived lifetimes.
- “Outlived lifetime” may not be the best/least confusing name for this feature,
for use in documentation. This should be bikeshedded before stabilization.

# Future possibilities
[future-possibilities]: #future-possibilities

None.
