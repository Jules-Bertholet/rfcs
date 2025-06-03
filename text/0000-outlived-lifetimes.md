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

In current Rust, it’s difficult to properly specify generic self-referential
structs, even with the help of `unsafe`. Using references imposes unneccessary
restrictions:

```rust
struct DataAndView<T> {
    x: Box<[T]>,
    element: &'static T,
           // ^^^^^^^ This forces us to add a `T: 'static` bound
           //         But what could we write instead?
}
```

To avoid these issues, it’s necessary to fall back to raw pointers:

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

Outlived lifetimes are covariant with respect to their arguments.

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

(This restriction exists to leave room for potentially adding additional,
non-lifetime-related dimensions of variance to types in a future version of
Rust.)

## As a bound in RPIT

When used as a bound in RPIT, outlived lifetimes capture all type parameters
used inside them. In other words, they have the same semantics as `use<…>`, and
therefore entirely replace that feature. Having become obsolete, `use<…>` is
removed from the next edition.

# Drawbacks
[drawbacks]: #drawbacks

- Adds complexity to the language and borrow checker.
- There are some cases in Rust today where Rust considers a pair of types to be
distinct, despite being mutual subtypes. For example, `for<'a, 'b> fn(&'a (),
&'b ())` is a different type from `for<'a> fn(&'a (), &'a ())`. This quirk can
lead to some very confusing borrow-check errors. This feature could expose that
problem in more cases, depending on how much normalization the implementation
performs.
- The meaning of `'<…>` is quite subtle. `T: 'a` is already a common source of
confusion for new Rust users; this feature could make that problem worse.
- Replacing the `+ use<…>` feature so soon after introducing it will cause
churn, annoying users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Versus [`unsafe<…>` binders](https://hackmd.io/@compiler-errors/HkXwoBPaR)

To the degree that it enables writing self-referential data structures, 

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
