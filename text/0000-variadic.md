# Variadics design sketch

## Use-cases we want to support

- [x] [`iter::zip`](https://doc.rust-lang.org/std/iter/fn.zip.html)
- [x] [`futures::join`](https://docs.rs/futures/latest/futures/future/fn.join.html)
- [x] [`Fn` traits](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) ([GitHub issue](https://github.com/rust-lang/rust/issues/41058))
- [x] Implementing traits for tuples ([stdlib](https://doc.rust-lang.org/std/primitive.tuple.html#trait-implementations), [Sourcegraph](https://sourcegraph.com/search?q=context:global+lang:Rust+impl%5B+%3C%5D+.*+for+%5C%28&patternType=regexp&sm=0&groupBy=repo))
- [x] Implementing traits for function pointers/`dyn Fn` ([stdlib](https://doc.rust-lang.org/std/primitive.fn.html#trait-implementations), [`wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen/blob/f569fddb62cdeb2bb9ba230fe59d3fba143cf92c/src/convert/closures.rs))
- [x] `#![feature(unsized_fn_params)]`
  - But don't penalize `Sized` ergonomics to accomplish it.
- [x] [`cmp::max`](https://doc.rust-lang.org/std/cmp/fn.max.html)
- [ ] [`frunk::HList`](https://docs.rs/frunk/latest/frunk/hlist/index.html)
- [ ] [`futures::select`](https://docs.rs/futures/latest/futures/future/fn.select.html)
- [ ] Other [`frunk`](https://docs.rs/frunk/latest/frunk/) use-cases?
- [ ] Compile-time reflection/macro-free `derive`?
- [ ] Suggestions welcome

## Design principles

- Re-use existing language infrastructure and concepts whenever possible (tuples, arrays, pattern matching).
  - "Perfection is achieved, not when there is nothing left to add, but when there is nothing left to take away."
  - But also, support library types!
- Don't support only pre-determined special cases, support all the cases.
  - Provide composable building blocks.
  - Try to be flat when possible, but allow nesting when needed.
  - Lifetime and const generics should be first-class citizens.
  - "Complex is better than complicated."
- Modular - try to have MVP-able subset, with confidence that it won't conflict with a more complete feature.
- Try to keep simple cases simple.
- "Explicit is better than implicit."
  - No implicit iteration! Ugly loops are preferable to beautiful bugs.
  - It should be clear whether an identifier refers to one thing or N things, wherever it is used.
- "There should be one-- and preferably only one --obvious way to do it."

## Prior art

- [C](https://en.cppreference.com/w/c/variadic)
- [Circle](https://github.com/seanbaxter/circle/blob/master/new-circle/README.md#pack-traits)
- [Clojure](https://clojure.org/guides/learn/functions#_multi_arity_functions)
- [Common Lisp](http://clhs.lisp.se/Body/03_dac.htm)
- [Crystal](https://crystal-lang.org/reference/1.8/syntax_and_semantics/default_values_named_arguments_splats_tuples_and_overloading.html#components-of-a-method-definition)
- [C++](https://en.cppreference.com/w/cpp/language/parameter_pack)
  - [Reversing a tuple](https://stackoverflow.com/questions/17178075/how-do-i-reverse-the-order-of-element-types-in-a-tuple-type)
- [C#](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/params)
- [D](https://dlang.org/spec/function.html#variadic)
- [F#](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/parameters-and-arguments#parameter-arrays)
- [Go](https://go.dev/ref/spec#Passing_arguments_to_..._parameters)
- [Haskell](https://wiki.haskell.org/Varargs)
- [Java](https://docs.oracle.com/javase/8/docs/technotes/guides/language/varargs.html)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [Julia](https://docs.julialang.org/en/v1/manual/functions/#Varargs-Functions)
- [Kotlin](https://kotlinlang.org/docs/functions.html#variable-number-of-arguments-varargs)
- [Lua](https://www.lua.org/pil/5.2.html)
- [ML](http://mlton.org/Fold)
- [Nim](https://nim-lang.org/docs/manual.html#types-varargs)
- [PHP](https://www.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)
- Python ([varargs](https://docs.python.org/3/reference/compound_stmts.html#function-definitions), [variadic generics](https://peps.python.org/pep-0646/))
- [Racket](https://docs.racket-lang.org/guide/lambda.html#%28part._rest-args%29)
  - [Typed Racket](https://docs.racket-lang.org/ts-guide/types.html#%28part._varargs%29)
- [R](https://cran.r-project.org/doc/manuals/r-release/R-intro.html#The-three-dots-argument)
- [Raku](https://docs.raku.org/language/signatures#Slurpy_parameters)
- [Ruby](https://docs.ruby-lang.org/en/3.2/syntax/calling_methods_rdoc.html#label-Array+to+Arguments+Conversion)
- [Scala](https://docs.scala-lang.org/scala3/reference/changed-features/vararg-splices.html)
- [Scheme](https://standards.scheme.org/corrected-r7rs/r7rs-Z-H-6.html#TAG:__tex2page_sec_4.1.4)
- [Swift](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md)
- [TCL](https://www.tcl.tk/man/tcl/TclCmd/proc.html)
- [TypeScript](https://www.typescriptlang.org/docs/handbook/2/objects.html#tuple-types)
- [VB.NET](https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/language-features/procedures/parameter-arrays)
- [Zig](https://ziglang.org/documentation/master/#comptime)

## Other links

- [Lang team design notes](https://github.com/rust-lang/lang-team/blob/master/src/design_notes/variadic_generics_design.md)
- [Analysing variadics, and how to add them to Rust - Olivier Faure](https://poignardazur.github.io/2021/01/30/variadic-generics/)
- [rust-variadics-background - Alice Cecile](https://github.com/alice-i-cecile/rust-variadics-background/)
- [[Brainstorming] Use cases for variadic generics - /r/Rust](https://www.reddit.com/r/rust/comments/l8vaa6/brainstorming_use_cases_for_variadic_generics/)
- [More enum types - Yoshua Wuyts](https://blog.yoshuawuyts.com/more-enum-types/)

## Variadic values

### `...` operator

`...` ("unpack") is an eager, prefix, unary operator that unpacks a parenthesized list of values. It operates on tuples, tuple structs, arrays, and references to them. (For tuple structs, all the fields must be visible at the location where `...` is invoked. Also, if the struct is marked `non_exhaustive`, then unpacking only works within the defining crate.)

TODO: should it be called "unpack", "splat", "spread", or something else?

```rust
let a: (i32, u32) = (3, 4);
let b: (i32, u32, usize) = (...a, 1);
let c: (usize, i32, u32) = (1, ...a);
struct Foo(bool, bool);
let d: (bool, bool) = (...Foo(true, false));
let e: Foo = Foo(...d);
let f: [i32, 3] = [1, ...[2, 3]];
let g: (i32, i32, i32) = (1, ...(2, 3));
let h: Foo: = Foo(...[true, false]);
let i: [i32, 3] = [1, ...(2, 3)];
```

Unpacking a reference results in references.

```rust
let j: (i32, &usize, &bool) = (3, ...&(4, false));
let k: (i32, &mut usize, &mut bool) = (3, ...&mut (4, false));
```

### `...` pattern

`...` can also be used in patterns.

#### Arrays

For arrays, `...` patterns serve as an alternative to `ident @ ..`.
TODO: Bikeshed placement of `ref` and `mut` keywords.

```rust
let a: [i32, 3] = [1, 2, 3];
let [...head, tail] = a;
assert_eq!(head, [1, 2]);
let [ref ...head, tail] = a;
let _: &[i32; 2] = head;
let [ref mut ...head, tail] = a;
let _: &mut [i32; 2] = head;
```

#### Tuples

`...` patterns also work with tuples.

```rust
let t: (i32, f64, &str) = (1, 2.0, "hi");
let (...head, tail) = t;
assert_eq!(head, (1, 2.0));
```

By-reference binding modes are a little special, owing to how tuples are stored in memory.

```rust
let t: (i32, f64, &str) = (1, 2.0, "hi");

let (ref ...head, tail) = t;
// Tuple of references, not reference to tuple
let _: (&_, &_) = head;

// Match ergonomics
let (...head, tail) = &t;
// Tuple of references, not reference to tuple
let _: (&_, &_) = head;
```

TODO: extend `ident @ ..` patterns to support this?
TODO: allow using `...` patterns with slices as well?

#### Tuple structs

Same as tuples.

```rust
struct Foo(i32, f64, &str);

let t: Foo = Foo(1, 2.0, "hi");
let Foo(...head, tail) = t;
assert_eq!(head, (1, 2.0));

let Foo(ref ...head, tail) = t;
let _: (&_, &_) = head;

let &Foo(...head, tail) = t;
let _: (&_, &_) = head;
```

### `static for` loop

`static for` loops allow iterating over the elements of am unpackable value (tuple, tuple struct, array, references to these). The loop is unrolled at compile-time.

TODO: bikeshed syntax.

```rust
let mut v: Vec<Box<dyn Debug>> = Vec::new();

static for i in (2, "hi", 5.0) {
    v.push(Box::new(i));
} // TODO: figure out how to make this work with semicolon elision rules

dbg!(&v); // [2, "hi", 5.0]
```

```rust
let mut v: Vec<i32> = Vec::new();
let mut a = [2, 3, 5];

static for i in a {
    v.push(i);
}

dbg!(&v); // [2, 3, 5]
```

```rust
let mut v: Vec<i32> = Vec::new();
let mut a = [2, 3, 5];

static for i in (...a, 4) {
    v.push(i);
}

assert_eq!(v, [2, 3, 5, 4])
```

`static for` loops evaluate to a tuple.

```rust
let _: (Option<u32>, Option<bool>) = static for i in (42, false) {
    Some(i)
};
```

You can `continue` with a value.

```rust
let _: (Option<u32>, Option<bool>) = static for i in (42, false) {
    if !pre_check() {
        continue None;
    }

    Some(expensive_computation(i))
};
```

`continue;` with no value specified is equivalent to `continue ();`.

```rust
static for i in (42, false) {
    if !pre_check() {
        continue;
    }

    expensive_operation(i);
}
```

`static for` can also contain a `break` statement; this changes the return type of the loop to `()`. Such a loop only supports bare `continue;`, with no value provided.
TODO: have separate keywords for these?

```rust
static for i in (42, false) {
    if we_are_done() {
        break;
    }

    some_operation(i);
}
```

You can `static for` over multiple unpackable values at once, as long as they are the same length.

```rust
let t: ((i8, u8), (i16, u16), (i32, u32)) = static for i, u in (-1, -2, -3), (1, 2, 3) {
    (i, u)
};

assert_eq!(t, ((-1, 1), (-2, 2), (-3, 3)));
```

```rust
// ERROR incompatible lengths
//static for i, u in (-1, -2, -3), (1, 3) {};
```

`...` patterns can also be used for working with multiple values at once. When used after `for`, the resulting binding has tuple type.

```rust
let t: ((i8, u8), (i16, u16), (i32, u32)) = static for ...tup in (-1, -2, -3), (1, 2, 3) {
    tup
};

assert_eq!(t, ((-1, 1), (-2, 2), (-3, 3)));
```

```rust
let t: ((i8, u8), (i16, u16), (i32, u32)) = static for ...tup in ...((-1, -2, -3), (1, 2, 3)) {
    tup
};

assert_eq!(t, ((-1, 1), (-2, 2), (-3, 3)));
```

## Variadic types

### `Tuple` trait

The trait `core::marker::Tuple` (TODO: or `core::primitive::Tuple`?) is implemented by the compiler for all tuples. No other impls are allowed. It has an associated constant, `const LENGTH: usize`, equal to the tuple's arity. (Tuples cannot have arity greater than `usize::MAX`.)

```rust
use core::marker::Tuple;

/// Takes a tuple and returns it unmodified. 
fn id<Ts: Tuple>(vs: Ts) -> Ts {
    vs
}

#[test]
fn test_id() {
    assert_eq!(id(()), ());
    assert_eq!(id((3,)), (3,));
    assert_eq!(id((42, "hello world")), (42, "hello world"),);
    assert_eq!(id((false, false, 0)), (false, false, 0));
    assert_eq!(id::<(&[u8; 9], &str)>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h"));
}
```

### Type-level `...`

Type-level `...` unpacks a tuple at the type level.

```rust
type Tup = (u32, bool);
struct Foo<T, U>(T, U);

//     ╭─ Equivalent to `Foo<u32, bool>`
//    ╭┴──────────╮
let _: Foo<...Tup> = Foo(1, false);
```

```rust
/// Does the same thing as `id`, but the tuple must have arity at least 1.
fn id_at_least_one<H, Ts: Tuple>(tuple: (H, ...Ts)) -> (H, ...Ts) {
    tuple
}

#[test]
fn test_id_at_least_one() {
    // assert_eq!(id_at_least_one(()), ()); ERROR expected at tuple of length at least one
    assert_eq!(id_at_least_one((3,)), (3,));
    assert_eq!(id_at_least_one((42, "hello world")), (42, "hello world"));
    assert_eq!(id_at_least_one((false, false, 0)), (false, false, 0));
    assert_eq!(id_at_least_one::<&[u8; 9], (&str,)>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h"));
}
```

```rust
/// Move the first element of the tuple to the end.
fn swing_around<H, Ts: Tuple>((head, ...tail): (H, ...Ts)) -> (...Ts, H) {
  //  ╭─ Use value-level unpack operator to construct the tuple.
  // ╭┴──────╮
    ( ...tail , head)
}

#[test]
fn test_swing_around() {
    //swing_around(()); ERROR expected tuple of arity >= 1, found 0
    assert_eq!(swing_around((3,)), (3,));
    assert_eq!(swing_around((42, "hello world")), ("hello world", 42));
    assert_eq!(swing_around((false, false, 0)), (0, false, false));
    assert_eq!(swing_around::<&[u8; 9], (&str,)>((b"t u r b o", "f i s h")), ("f i s h", b"t u r b o"));
}
```

```rust
/// Push `17` onto the end of the tuple.
fn add_one_on<Ts: Tuple>(vs: Ts) -> (...Ts, u32) {
    (...vs, 17)
}

#[test]
fn test_add_one_on() {
    assert_eq!(add_one_on(()), (17,));
    assert_eq!(add_one_on((3,)), (3, 17));
    assert_eq!(add_one_on((42, "hello world")), (42, "hello world", 17));
    assert_eq!(add_one_on((false, false, 0)), (false, false, 0, 17));
    assert_eq!(add_one_on::<(&[u8; 9], &str,)>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h", 17));
}
```

It also works with arrays:

```rust
//     ╭─ Equivalent to `(i32, i32, i32)`
//    ╭┴────────────╮
let a: (...[i32; 3]) = (2, -5, 7);
```

However, unlike value-level `...`, it does not work with tuple structs.

### `for`-`in` trait bounds

The `for`-`in` syntax for `where` bounds allows bounding the individual elements of a tuple.

```rust
use core::{fmt, marker::Tuple};

impl<Ts: Tuple> fmt::Debug for Ts
where
    // Every element of the tuple `Ts` must implement `Debug`
    for<T in Ts> T: fmt::Debug,
{
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let mut list = f.debug_list();

        static for elem in self {
            f.entry(elem);
        }
        
        list.finish()
    }
}
```

`for`-`in` can also be used to relate multiple types:

```rust
use core::marker::Tuple;

// Implement heterogeneous `PartialEq` for tuples

impl<Ts: Tuple, Us: Tuple> PartialEq<Us> for Ts
where
    // Implicitly requires `Ts` and `Us` to have same length
    for<T, U in Ts, Us> T: PartialEq<U>,
{
    fn eq(&self, other: &Us) {
        static for l, r in self, other {
            if l != r {
                return false;
            }
        }

        true
    }
}
```

It also works with const-generic splattable values:

```rust
fn foo()
where
    // `[u32; 1]`, `[usize; 2]`,  and `[i32; 17]` must all implement `Debug`.
    for<T, const N in (u32, usize, i32), [1, 2, 17]> [T; N]: Debug,
{}
```

And with other splattable types:

```rust
fn foo()
where
    // `[u32; 1]`, `[u32; 2]`, and `[u32; 17]` must all implement `Debug`.
    // (Yes, this is a contrived example)
    for<T, const N in [u32; 3], [1, 2, 17]> [T; N]: Debug,
{}
```

### `for`-`in` type mapping

`for`-`in` syntax can also be used to map existing tuple types to a new type.

```rust
//     ╭─ `(Option<i32>, Option<u32>)`, but written in an overly verbose fashion.
//    ╭┴─────────────────────────────╮
let t: for<T in (i32, u32)> Option<T> == (Some(2), Some(3));

```

```rust
//     ╭─ `([bool; 3], [bool; 12])`, but written in an overly verbose fashion.
//    ╭┴─────────────────────────────────────────╮
let t: (for<const N: usize in (8, 12)> [bool; N]) = ([false; 3], [false; 0]);
```

```rust
//     ╭─ `([bool; 3], [char; 1])`, but written in an overly verbose fashion.
//    ╭┴────────────────────────────────────────────────────╮
let t: for<T, const N: usize in (bool, char), (3, 1)> [T; N] = ([false; 3], ['c']);
```

```rust
//     ╭─ `(usize, (u32, i32, usize), (bool, char))`, but written in an overly verbose fashion.
//    ╭┴──────────────────────────────────────────────────────────╮
let t: (usize, ...for<Ts in ((u32, i32, usize), (bool, char))> Ts) = (4, (1, 7, 4), (true, 'c'));
```

```rust
/// Wraps all the elements of the input tuple in `Some`.
//
//                                   ╭─ Map each type `T` in the tuple `Ts`
//                                   │  to a corresponding `Option<T>`.
//                                  ╭┴─────────────────────╮
fn option_wrap<Ts: Tuple>(vs: Ts) -> for<T in Ts> Option<T> {
    let wrapped: for<T in Ts> Option<T> = static for v in vs {
        Some(v)
    };

    wrapped
}

#[test]
fn test_option_wrap() {
    assert_eq!(option_wrap(()), ());
    assert_eq!(option_wrap((3,)), (Some(3),));
    assert_eq!(option_wrap((42, "hello world")), (Some(42), Some("hello world")));
    assert_eq!(option_wrap((false, false, 0)), (Some(false), Some(false), Some(0)));
    assert_eq!(option_wrap::<(&[u8; 9], &str)>((b"t u r b o", "f i s h")), (Some(b"t u r b o"), Some("f i s h")));
}
```

In where clauses, you can use parentheses to disambiguate.

```rust
use std::marker::Tuple;

fn foo<Ts: Tuple>()
where
    // Every member of `Ts` implements `Debug`
    for<T in Ts> T: Debug,
    // `Ts` itself implements `Debug`
    (for<T in Ts> T): Debug,
{}
```

### `static for` over types and consts

`static for` can also be used to iterate over the members of a splattable type or const.
TODO: bikeshed, figure out interaction with `...ident`.

```rust
let t: (usize, i32) = static for type T in (usize, i32) {
    T::default();
};

assert_eq!(t, (0, 0))
```

```rust
impl<Ts: Tuple> Default for Ts
where
    for<T in Ts> T: Default,
{
    static for type T in Ts {
        T::default()
    }
}
```

```rust
// TODO: should type annotation on `const N` be required?
let t: (usize, usize, usize) = static for const N: usize in (3, 17, 21) {
    N + 1
};

assert_eq!(t, (4, 18, 22));
```

```rust
let t: (usize, i32, bool) = static for type T, const N: T in (usize, i32, bool), (3, -10, true) {
    N + T::default()
};

assert_eq!(t, (3, -10, true));
```

```rust
let t: ((i32, bool), (u32, usize), (i32, u8)) = static for val, type T in (1, 2, 1), (bool, usize, u8) {
    (val, T::default())
};

assert_eq!(t, ((1, false), (2, 0), (1, 0)));
```

### Lifetime packs

Variadics also supports lifetime, via a new concept of "lifetime pack". A lifetime pack is a list of lifetimes, enclosed in angle brackets (parentheses would be ambiquous w.r.t `()` and lifetime elision). They can also be bound to identifiers; these identifiers look like normal lifetimes, but with an extra prefixed `'` per level of nesting. Lifetime packs can be used with `for`-`in`.

```rust
use std::marker::Tuple;

//      ╭─ Accepts an angle-bracketed list of lifetimes
//     ╭┴───╮
fn foo< ''as >()
where
    // `''as` is required to have length 3
    for<'a, T in ''as, (u32, usize, i32)> &'a T: Debug,
{}
```

They also work with `static for`.

```rust
static for 'x in <'static, 'a> {
    // Not much interesting you can do with only a lifetime...
    // But these will get more useful in later examples.
    let _: &'x i32 = &42;
};
```

Lifetime parameters cannot affect monomorphization, so lifetime packs used with `static for` must have known/constrained lengths once all type and const generics are known.
TODO: bikeshed these rules.

```rust
fn foo_broken<''as>() {
    // ERROR: length of `''as` is not constrained
    for 'a is ''as {
        println!("hello world");
    }
}
```

```rust
fn foo_fixed<''as, const N: usize>()
where
    // TODO: this is ugly
    for <'_, _ in ''as, [(); N]>:,
{
    for 'a is ''as {
        println!("hello world");
    }
}
```

You can use a lifetime pack as the union of the contained lifetimes by omitting the extra preceding `'`s.

```rust
fn id<''as, Ts: Tuple>(tup: for<'a, T in ''as, Ts> &'a T) -> impl Sized + 'as
where
    for <'_, _ in ''as, Ts>:,
{
   tup
}
```

## Variadic parameters

### `...` patterns in function parameter lists

`...` patterns can also be used in function parameter lists.

```rust
//                    ╭─ This binding has tuple type.
//                    │  However, in terms of calling convention,
//                    │  elements are passed individually.
//                    │  TODO: allow `tuple @ ..` as alternative syntax?
//                    │
//                   ╭┴────────╮
fn collect_contrived( ... tuple : ... (u32, i32) ) -> (u32, i32) {
    tuple
}

// The above and below functions are exactly equivalent, in signature, semantics, and calling convention.

fn collect_contrived(a: u32, b: i32) -> (u32, i32) {
    (a, b)
}
```

When using a `...` binding with unsized argument types via `unsized_fn_params`, there are additional rules to keep in mind. Specifically: a tuple type is valid whether or not any of its members are `Sized`. But, in addition to the restrictions that all unsized types have, it's not allowed to take the address of multiply-unsized tuples, nor is it allowed to pass them as function arguments, even with `#![feature(unsized_fn_params)]` (for now, until a layout for them is RFCed). They can be thought of as not implementing an unnameable`?Sized`-style trait.

Without `#![feature(unsized_fn_params)]`, you can't produce a value of a mutiply-unsized tuple at all. With the feature gate, you can do so with tuple literal syntax and `...`-in-fn-parameter-lists. You can also index into their fields, destructure them with a pattern match, unpack them with `...`, or iterate over them with `static for`.

```rust
#![feature(unsized_fn_params)]

fn do_stuff_with_unsized_params(...tuple: ...([u32], dyn Debug)) {
    // `tuple` is still a tuple.
    // But: you aren't allowed to take its address[^1], or pass it to a function by value.
    // The only thing you can do is index into its fields;
    // either directly, via pattern match, or with `static for`.
    // These restrictions won't be lifted even if `([u32], dyn Debug)` gets a defined layout,
    // and in fact even a tuple like `(u8, str)` has them,
    // but they may be lifted with `#![feature(unsized_locals)]`.
    //
    // [^1]: You can take a reference if you immiediately unpack it (with `...` or `static for`).

    dbg!(&tuple.0[0]);
    dbg!(&tuple.1);
}
```

```rust
/// Accepts a variadic list of arguments,
/// returns those arguments collected into a tuple.
fn collect<Ts: Tuple>(...vs: ...Ts) -> Ts {
    vs
}

#[test]
fn test_collect() {
    assert_eq!(collect(), ());
    assert_eq!(collect(3), (3,));
    assert_eq!(collect(42, "hello world"), (42, "hello world"));
    assert_eq!(collect(false, false, 0), (false, false, 0));
    assert_eq!(collect::<(&[u8; 9], &str,)>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}
```

```rust
/// Does the same thing as `id`, but you have to pass in at least one value.
fn collect_at_least_one<H, Ts: Tuple>(head: H, ...tail: ...Ts) -> (H, ...Ts) {
    (head, ...tail)
}

#[test]
fn test_collect_at_least_one() {
    // assert_eq!(collect_at_least_one(), ()); ERROR expected at least 2 parameters, found 1
    assert_eq!(collect_at_least_one(3), (3,));
    assert_eq!(collect_at_least_one(42, "hello world"), (42, "hello world"));
    assert_eq!(collect_at_least_one(false, false, 0), (false, false, 0));
    assert_eq!(collect_at_least_one::<&[u8; 9], (&str,)>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}
```

Multiple variadic arguments can follow one another, but turbofish is then required for disambiguation.
TODO: bikeshed above statement.

```rust
fn double_vararg<Ts: Tuple, Us: Tuple>(...fst: ...Ts, ...lst: ...Us) -> (Ts, Us) {
    (fst, lst)
}

#[test]
fn test_double_vararg() {
    assert_eq!(double_vararg::<(i32, u32), (usize,)>(1, 2, 3), ((1, 2), (3,)));
    assert_eq!(double_vararg::<(i32,), (u32, usize)>(1, 2, 3), ((1,), (2, 3)));
}
```

`...` patterns in parameter lists also work with arrays.

```rust
/// Variadic version of `std::cmp::max`
pub fn max<T: Ord, const N: usize>(fst, ...rest: [T; N]) -> T {
    let mut max = fst;
    static for elem in rest {
        if elem > max {
            max = elem;
        }
    }
    
    max
}
```

### Vararg generic parameters

Generic parameters also have a vararg form.

```rust
```rust
/// Accepts a variadic list of arguments,
/// returns those arguments collected into a tuple.
/// Does the same thing as `collect` from earlier.
//               ╭─ `Ts` has tuple type.
//              ╭┴─╮
fn collect2< ... Ts >(...vs: ...Ts) -> Ts {
    vs
}

#[test]
fn test_collect2() {
    assert_eq!(collect(), ());
    assert_eq!(collect(3), (3,));
    assert_eq!(collect(42, "hello world"), (42, "hello world"));
    assert_eq!(collect(false, false, 0), (false, false, 0));
    //                    ╭─ No tuple, generic parameters passed separately.
    //                   ╭┴─────────────╮
    assert_eq!(collect::< &[u8; 9], &str >(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}
```

You can put trait bounds on variadic generic parameters.

```rust
/// Return a tuple with the specified element types, generating each element value by calling `Default::default()`.
//          ╭─ Every type in `Ts` must meet the `: Default` bound.
//         ╭┴─────────────╮
fn default< ...Ts: Default >() -> Ts {
    // `static for` with variadic generic type.
    static for type T in Ts {
        T::default()
    }
}

#[test]
fn test_default() {
    assert_eq!(default::<>(), ());
    assert_eq!(default::<u32>(), (0,));
    assert_eq!(default::<u32, bool>(), (0, false));
}
```

```rust
// `FnOnce` with `unsized_fn_params` support
#![feature(unsized_fn_params)]

//            ╭─ Any of the types in `Args` can be unsized.
//           ╭┴──────────────╮
trait FnOnce< ...Args: ?Sized > {
    type Output;

    fn call_once(self, ...args: ...Args) -> Self::Output;
}

struct Foo;

// No variadic syntax here!
impl FnOnce<u32, i32, i32> for Foo {
    type Output = u32;

    fn call_once(self, a: u32, b: i32, c: i32) -> u32 {
        a + ((b + c) as u32)
    }
}
```

Variadic generic parameters can come in front of non-variadic ones.

```rust
fn is_last_3<...Ts, Last: PartialEq<i32>>((...front, last): (...Ts, Last)) -> (...Ts, bool) {
    (...front, last == 3)
}

#[test]
fn test_is_last_3() {
    assert_eq!(is_last_3(("yeah", "yeah", 3)), ("yeah", "yeah", true));
}
```

(TODO: can variadic paramerers be defaulted themselves?)

Variadic lifetime and const parameters are also allowed.

```rust
fn foo<...''as, ...Ts>(..._refs: for<'a, T in ''as, Ts> &'a T)
where
    for<'a, T in ''as, Ts> T: 'a,
{}
```

```rust
/// Collect the provided const generic params into a `for` loop.
// TODO: needs bikeshed. What if each `const` has a different type? Would that even make sense?
//                 ╭─ Takes some number of cont generic `i32` parameters,
//                 │  collects them into an array.
//                ╭┴────────────────╮
fn collect_consts< ... const Us: i32 >() -> [i32; { Us.len() }] {
    Us
}

#[test]
fn test_collect_consts() {
    assert_eq!(collect_consts::<>(), []);
    assert_eq!(collect_consts::<3, 8, 7>(), [3, 8, 7]);
}
```

You can't have multiple variadic generic parameters of the same kind (type, lifetime, or const) in the same parameter list.

```rust
// ERROR
fn err<...Ts, ...Us>() {}
```

Variadic parameters also can't precede defaulted parameters of the same kind.

```rust
// ERROR
fn err<...Ts, U = i32>() {}
```

Variadic generic paramerers add no new post-monomprphization errors, all possible instantiations must be valid. (There is one exception to this, detailed in an earlier section: tuple and tuple struct arities can't overflow `usize::MAX`.)

```rust
fn foo<...Ts>(tup: Ts) {
    // drop(...tup); // ERROR: `drop` is a function of arity 1, but `tup` could have any arity
}
```

### `..` inferred type parameter lists

In today's Rust, you can mark type parameters as inferred with `_`.

```rust
let a: Option<_> = Some(3_u32);
```

With this proposal, you can use `..` to infer a list of type parameters.

```rust
// `..` is equivalent to `_, _` below
let tup: (usize, ..) = (3, false, "hello");
```

### Variadics with ADTs

```rust
struct Tup<...Ts>(i32, ...Ts);

impl<...Ts> Tup<...Ts> {
    fn new(i: i32, ...ts: ...Ts) {
        Self(i, ...ts)
    }

    fn foo(&self) {
        dbg!(self.0);
        // dbg!(self.1); ERROR: that field might not exist
    }
}
```

```rust
// `iter::Zip`

pub struct Zip<...Is>(...Is)

impl<...Is> Zip<...Is> {
    fn new(...iters: ...Is) {
        Self(...iters)
    }
}

impl<...Is: Iterator> Iterator for Zip<...Is> {
    type Item = for<I in Is> I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        let next: Self::Item = static for i in self {
            i.next()?
        };

        Some(next)
    }
}
```

```rust
// `futures::join`

use  futures_util::future::{MaybeDone, maybe_done};

#[pin_project::project]
pub struct Join<...Futs: Future>(#[pin] ...for<F in Futs> MaybeDone<F>);

impl<...Futs> Join<...Futs> {
    fn new(...futures: ...Futs) {
        let wrapped_futs = static for future in futures {
            maybe_done(future)
        };

        Self(...wrapped_futs)
    }
}

impl<...Futs: Future> Future for Join<...Futs> {
    type Output = for<F in Futs> F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut all_done = true;

        // TODO: what is the best API for pin projection?

        // Long, annoying type specified for example purposes.
        // In reality, you would infer it with `(..)`.
        let futs: for<F in Futs> Pin<&mut F> = self.project();

        static for fut in futs {
            all_done &= fut.poll(cx).is_ready();
        }

        if all_done {
            let ready = static for fut in futs {
                fut.take_output().unwrap()
            };

            Poll::Ready(ready)
        } else {
            Poll::Pending
        }
    }
}
```

### Advanced examples

```rust
/// Pair of tuples → tuple of pairs
pub fn zip<Ts: Tuple, Us: Tuple>(pair: (Ts, Us)) -> for<T, U in Ts, Us> (T, U)
where
    Ts::LENGTH == Us::LENGTH,
{
    static for l, r in pair.0, pair.1 {
        (l, r)
    }
}
```

```rust
/// Tuple of pairs → pair of tuples
pub fn unzip<Ts: Tuple, Us: Tuple>(zipped: for<T, U in Ts, Us> (T, U)) -> (Ts, Us) 
where
    Ts::ARITY == Us::ARITY,
{
    // This one is a bit mind-bending.
    // TODO: could it be made more intuitive?
    static for ...elems in ...zipped {
        elems
    }
}
```

```rust
/// MxN → NxM transformation, generalized version of `zip` and `unzip`
pub fn transpose<...Tss: Tuple>(tup: Tss) -> for<...Ts in ...Tss> Ts
where
    for<Ts in Tss> Ts: Tuple,
    // TODO: a bit mind-bending
    for<...T in ...Ts> ():,
{
    static for ...elems in ...tup {
        elems
    }
}
```

```rust
/// Higher-ranked lifetime binders

trait Foo;

impl<...Ts> Foo for for<...''as where for<'_, _ in ''as, Ts>: (),> fn(...for<'a, T in 'as, Ts> &'a T) {}
```

### TODO: `match` and recursion

We could potetially allow recursion to loop over the elements of a tuple.

```rust
/// Debug-print every element of the tuple.
fn recursive_dbg<...Ts: Debug>(ts: Ts) {
    match ts {
        () => (),
        (head, ...tail) => {
            print!("{head:?}, ");
            recursive_dbg(tail);
        }
    }
}
```

Recursion would allow iterating in a different order.

```rust
/// Debug-print every element of the tuple, in reverse order.
fn recursive_dbg<...Ts: Debug>(ts: Ts) {
    match ts {
        () => (),
        (...head, tail) => {
            print!("{tail:?}, ");
            recursive_dbg(head);
        }
    }
}
```

However, how do we express this recursion at the type level?

```rust
/// Reverse the order of the elements in the tuple.
fn reverse_tuple<...Ts: Debug>(ts: Ts) -> _ /* what do we put here? */ {
    match ts {
        () => (),
        (head, ...tail) => {
           (...reverse_tuple(tail), head)
        }
    }
}
```

A more generalized type-level recursion mechanism/`match` would be the most flexible option, but also complex. Alternatively, we could have a set of primitives, like `Rotate<..>` or `Reverse<..>`, for type-level operations. Also, what restrictions would need to be imposed to make type inference tractable, and avoid post-monomorphization errors? How would unification work?

It's also not clear how the `match` + recursion approach could be applied to iterating over types/lifetimes/consts at the value level.

One final challenge is whether this approach will lead to stack usage problems.

#### Type-level `match`

What might type-level `match` look like?

```rust
type Reverse<Ts> = match Ts {
    // Restriction: lifetimes can't affect which branch is taken, otherwise you would have unsound specialization
    () => (),
    (Head, ...Tail) => (...Reverse<Tail>, Head),
};

/// Reverse the order of the elements in the tuple.
fn reverse_tuple<...Ts: Debug>(ts: Ts) -> Reverse<Ts> {
    match ts {
        () => (),
        (head, ...tail) => {
           (...reverse_tuple(tail), head)
        }
    }
}
```

### TODO: Random access

It would be nice to have a way to express "get the `N`th value of this tuple, if it exists", where `N` is a const generic parameter. There would need to be both value-level and type-level versions of this.

Pattern types might help here, to constrain the index to a valid range.

(People have expressed this desire in feedback to the author, but I would like to see concrete use-cases.)

### TODO: Variadic variant lists with enums

With APIs like `futures::select`, you want to pass in N values, and get back a value corresponding to one of the `N` arguments. [Yoshua Wuyts's blog posts](https://blog.yoshuawuyts.com/more-enum-types/) explore this in detail. To support this case, one might want enum types whose variants correspond to variadic generic parameters. But perhaps the additional complexity is not necessary; for example, in `futures::select` we could instead ask the caller to wrap the result type of their futures in an enum they define.

## TODO

- [ ] Add additional advanced examples
  - [ ] Elided lifetimes
  - [ ] GATs
- [ ] Resolve inline bikeshed TODOs
