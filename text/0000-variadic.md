# Variadics design sketch

## Use-cases we want to support

- `iter::zip`
- `futures::join`
- `Fn` traits
- Implementing traits for tuples
- `frunk`
- `unsized_fn_params`
  - But don't penalize `Sized` ergonomics to accomplish it.
- `futures::select`?
- Gateway drug to compile-time reflection/macro-free `derive`?

## Design principles

- Re-use existing language infrastructure and concepts when possible (tuples, pattern matching).
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

## Prior art

- [C](https://en.cppreference.com/w/c/variadic)
- [C++](https://en.cppreference.com/w/cpp/language/parameter_pack)
- [D](https://dlang.org/spec/function.html#variadic)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [PHP](https://www.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)
- [Ruby](https://docs.ruby-lang.org/en/3.2/syntax/calling_methods_rdoc.html#label-Array+to+Arguments+Conversion)
- [Scala](https://docs.scala-lang.org/scala3/reference/changed-features/vararg-splices.html)
- [Swift](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md)
- [Zig](https://ziglang.org/documentation/master/#comptime)

## Variadics and values

### `...` operator

`...` ("splat") is a prefix operator that unpacks a parenthesized list of values. It operates on tuples, tuple structs, and arrays. (For tuple structs, all the fields must be visible at the location where `...` is invoked. Also, if the struct is marked `non_exhaustive`, then splatting only works within the defining crate.)

TODO: should it be called "splat", "spread", or something else?

```rust
let a: (i32, u32) = (3, 4);
let b: (i32, u32, usize) = (...a, b);
let c: (usize, i32, u32) = (b, ...a);
struct Foo(bool, bool);
let d: (bool, bool) = (...Foo(true, false),); // TODO: should trailing comma be required?
let e: Foo = Foo(...d);
let f: [i32, 3] = [1, ...[2, 3]];
let g: (i32, i32, i32) = (1, ...(2, 3));
let h: Foo: = Foo(...[true, false]);
let i: [i32, 3] = [1, ...(2, 3)];
```

### `...` pattern

`...` can also be used in patterns. There, it serves as an alternative to `ident @ ..`.

#### Arrays

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

`...ident` patterns also work with tuples.

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
let &(...head, tail) = t;
// Tuple of references, not reference to tuple
let _: (&_, &_) = head;
```

TODO: extend `ident @ ..` patterns to support this?

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

`static for` loops allow iterating over the elements of a splattable value (tuple, tuple struct, array).
The loop is unrolled at compile-time.

TODO: bikeshed.

```rust
let mut v: Vec<Box<dyn Debug>> = Vec::new();

static for i in (2, "hi", 5.0) {
    v.push(i);
}

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

TODO: should `static for` over an array evaluate to an array instead?

`static for` does not support `break`, but you can `continue` with a value.

```rust
let _: (Option<u32>, Option<bool>) = static for i in (42, false) {
    if !pre_check() {
        continue None;
    }

    Some(expensive_computation(i))
};
```

`continue` with no value specified is equivalent to `continue ()`.

```rust
static for i in (42, false) {
    if !pre_check() {
        continue;
    }

    expensive_operation(i);
}
```

## Variadics and types

### Generic parameter syntax

Today, generic parameter lists are necessarily flat. At the value level, we can use arrays, tuples, and slices to give structure to the parameters we pass in. But at the type level, this is not possible. More complex uses of variadics may need structured parameter lists, so we suggest syntax for it in the table below. However, the vast majority of cases will hopefully only use the first row in the table of proposed additions.

| **Today's Rust**        | Declare                           | Fill            |
|-------------------------|-----------------------------------|-----------------|
| One type parameter      | `<T>`                             | `<i32>`         |
| Two type parameters     | `<T, U>`                          | `<i32, usize>`  |
| Two lifetime parameters | `<'a, 'b>`                        | `<'a, 'static>` |
| Two const parameters    | `<const A: i32, const B: usize>`  | `<3, 5>`        |

| **Proposed additions**                                                                       | Declare                                                         | Fill                                                                                             |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| One list of zero or more type parameters (MVP needs only this)                               | `<...Ts>`                                                       | `<i32, u32, usize>`                                                                              |
| Two lists of zero or more type parameters                                                    | `<<...Ts>, <...Us>>`                                            | `<<i32, u32, &str>, <u32, usize>>`                                                               |
| Two lists of zero or more lifetime parameters                                                | `<<...'as>, <...'bs>>`                                          | `<<'static, '_>, <'a, 'b>>`                                                                      |
| Two lists of zero or more const parameters                                                   | `<<...const As: i32>, <const Bs: u32>>`                         | `<<-1, 0, 1>, <42>>`                                                                             |
| One list of groupings of one lifetime parameter, one type parameter, and one const parameter | `<...<'as, Ts: 'as, const Ns: Ts>>`                             | `<<'static, i32, -4>, <'_, bool, false>>`                                                        |
| Putting it all together                                                                      | `<'a, ...'bs, ...Ts, ...<'a, T>, <...<Ts, Us>>, <...<Ts, Us>>>` | `<'static, '_, 'b, u32, i64, <'static, u32>, <'static, i32>>, <<u32, usize>, <i32, isize>>, <>>` |

### `static for` over types, lifetimes, and consts

`static for` can also be used to iterate over a block with a list of types, lifetimes, or consts.

The examples in this section show how to use this feature with concrete types, but it is really meant for use with variadic generic types. Examples of that will come later.

TODO: bikeshed.

```rust
let t: (usize, i32) = static for type T in <usize, i32> {
    T::default();
};

assert_eq!(t, (0, 0))
```

```rust
static for 'x in <'static, 'a> {
    // Not much interesting you can do with only a lifetime...
    // But these will get more useful in later examples.
    let _: &'x i32 = &42;
};
```

```rust
let t: (usize, usize, usize) = static for const N: usize in <3, 17, 21> {
    // Like with lifetimes, this only gets useful once you use it with generics.
    N + 1
};

assert_eq!(t, (4, 18, 22));
```

```rust
let t: (usize, i32, bool) = static for <type T, const N: T> in <<usize, 3>, <i32, -10>, <bool, true>> {
    N
};

assert_eq!(t, (3, -10, true));
```

### Type-level `...`

Type-level `...` unpacks a comma-separated list of types, lifetimes, or consts at the type level.

This feature is meant for use with variadic generics, so the concrete examples will be underwheling.

```rust
//     ╭─ `(i32, u32)`, but written in an overly verbose fashion.
//     │  TODO: should trailing comma be required?
//    ╭┴───────────────╮
let t: (...<i32, u32>,) = (2, 3);
```

```rust
//     ╭─ `(i32, u32, usize)`, but written in an overly verbose fashion.
//    ╭┴─────────────────────╮
let t: (...<i32, u32>, isize) = (2, 3, -4);
```

### Type-level for-in

Type-level for-in maps a comma-separated list of types, lifetimes, or consts to a new list of the same length, at the type level.

Once again, for use with variadic generics, so the concrete examples will be underwheling.

```rust
//     ╭─ `(Option<i32>, Option<u32>)`, but written in an overly verbose fashion.
//    ╭┴───────────────────────────────────╮
let t: (...for<T in <i32, u32>> Option<T>,) == (Some(2), Some(3));
```

```rust
//     ╭─ `([bool; 3], [bool; 12])`, but written in an overly verbose fashion.
//    ╭┴─────────────────────────────────────────────╮
let t: (...for<const N: usize in <8, 12>> [bool; N],) = ([false; 3], [false; 0]);
```

```rust
//     ╭─ `([bool; 3], [char; 1])`, but written in an overly verbose fashion.
//    ╭┴──────────────────────────────────────────────────────────────╮
let t: (...for<<T, const N: usize> in <<bool, 3>, <char, 1>>> [T; N],) = ([false; 3], ['c']);
```

```rust
//     ╭─ `(usize, (u32, i32, usize), (bool, char))`, but written in an overly verbose fashion.
//    ╭┴───────────────────────────────────────────────────────────────────╮
let t: (usize, ...for<...Ts in <<u32, i32, usize>, <bool, char>>> (...Ts,)) = (4, (1, 7, 4), (true, 'c'));
```

### Functions with variadic generic parameters

Finally, the good stuff.

This section will just be examples, showing how the concepts we've introduced so far fit together.

```rust
/// Takes a tuple and returns it unmodified.
//
//     ╭─ Declare variadic generic parameter of zero or more types.
//     │
//     │             ╭─ A tuple of all the types in `Ts`.
//     │             │
//     │             ├─────────────╮
//    ╭┴────╮       ╭┴───────╮    ╭┴───────╮  
fn id< ...Ts >( vs : (...Ts,) ) -> (...Ts,) {
    vs
}

#[test]
fn test_id() {
    assert_eq!(id(()), ());
    assert_eq!(id((3,)), (3,));
    assert_eq!(id((42, "hello world")), (42, "hello world"),);
    assert_eq!(id((false, false, 0)), (false, false, 0));
    assert_eq!(id::<[u8; 9], &str>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h"));
}
```

```rust
/// Does the same thing as `id`, but the tuple must have arity at least 1.
fn id_at_least_one<H, ...Ts>(tuple: (H, ...Ts)) -> (H, ...Ts) {
    tuple
}

#[test]
fn test_id_at_least_one() {
    // assert_eq!(id_at_least_one((),), ()); ERROR expected at tuple of length at least one
    assert_eq!(id_at_least_one(3), (3,));
    assert_eq!(id_at_least_one((42, "hello world")), (42, "hello world"));
    assert_eq!(id_at_least_one((false, false, 0)), (false, false, 0));
    assert_eq!(id_at_least_one::<[u8; 9], &str>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h"));
}
```

```rust
/// Wraps all the elements of the input tuple in `Some`.
//
//                                           ╭─ Map each type `T` in the tuple to a corresponding `Option<T>`.
//                                           │  Type-level for-in with generics.
//                                          ╭┴─────────────────────╮
fn option_wrap<...Ts>(vs : (...Ts,)) -> (... for<T in Ts> Option<T> ,) {
    let wrapped: (...for<T in Ts> Option<T>,) = static for v in vs {
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
    assert_eq!(option_wrap::<[u8; 9], &str>((b"t u r b o", "f i s h")), (Some(b"t u r b o"), Some("f i s h")));
}
```

```rust
/// Move the first element of the tuple to the end.
fn swing_around<H, ...Ts>((head, ...tail): (H, ...Ts)) -> (...Ts, H) {
  //  ╭─ Use value-level splat operator to construct the tuple.
  // ╭┴──────╮
    ( ...tail , head)
}

#[test]
fn test_swing_around() {
    //swing_around(()); ERROR expected tuple of arity >= 1, found 0
    assert_eq!(swing_around((3,)), (3,));
    assert_eq!(swing_around((42, "hello world")), ("hello world", 42));
    assert_eq!(swing_around((false, false, 0)), (0, false, false));
    assert_eq!(swing_around::<[u8; 9], &str>((b"t u r b o", "f i s h")), ("f i s h", b"t u r b o"));
}
```

```rust
/// Push `17` onto the end of the tuple.
fn add_one_on<...Ts>(vs : (...Ts,)) -> (...Ts, u32) {
    (...vs, 17)
}

#[test]
fn test_add_one_on() {
    assert_eq!(add_one_on(()), (17,),);
    assert_eq!(add_one_on((3,)), (3, 17));
    assert_eq!(add_one_on((42, "hello world")), (42, "hello world", 17));
    assert_eq!(add_one_on((false, false, 0)), (false, false, 0, 17));
    assert_eq!(add_one_on::<[u8; 9], &str>((b"t u r b o", "f i s h")), (b"t u r b o", "f i s h", 17));
}
```

```rust
/// Return a tuple with the specified element types, generating each element value by calling `Default::default()`.
//          ╭─ Every type in `Ts` must meet the `: Default` bound.
//         ╭┴─────────────╮
fn default< ...Ts: Default >() -> (...Ts,) {
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

```rust
/// Returns `true` iff 3 is equal to every element of the given tuple.
fn all_3<...Ts>(vs : (...Ts,)) -> bool
where
    /// Every type `T` in `Ts` meets the specified bound.
    for<T in Ts> i32: PartialEq<T>,
{
    static for v in vs {
        if 3_i32 != v {
            return false;
        }
    }

    true
}

#[test]
fn test_all_3() {
    assert_eq!(all_3(()), true);
    assert_eq!(all_3((3, 3, 3)), true);
    assert_eq!(all_3((3, 17, 3)), false);
}
```

```rust
/// Collect the provided const generic params into a `for` loop.
fn collect_consts<...const Ns: usize>() -> (...for<const N: usize in Ns> usize,) {
    static for const N: usize in Ns {
        N
    }
}

#[test]
fn test_collect_consts() {
    assert_eq!(collect_consts::<>(), ());
    assert_eq!(collect_consts::<3, 8, 7>(), (3, 8, 7));
}
```

### `...` patterns in function parameter lists

Funv

```rust

/// Takes a variadic set of type arguments,
/// returns those arguments collected into a tuple.
//
//                 ╭─ Declare variadic function parameter of zero or more values.
//                 │  `vs` has tuple type. However, terms of calling convention,
//                 │  the tuple is passed "destructured", as if each argument was separate.
//                 │  If the callee is to use the tuple as a tuple, the callee
//                 │  must reconstruct the tuple on its stack first.
//                 │  (If the callee just needs to access individual fields, this reconstruction
//                 │  is not performed, ensuring that simple variadics are a zero-cost abstraction.)
//                 │
//                 │
//                 │
//                 │  TODO: allow `vs @ ..` as alternative?
//                 │
//                ╭┴─────────╮ 
fn collect<...Ts>( ...vs : Ts ) -> (...Ts,) {
    // Return the tuple `vs`.
    vs
}

#[test]
fn test_collect() {
    assert_eq!(collect(), ());
    assert_eq!(collect(3), (3,));
    assert_eq!(collect(42, "hello world"), (42, "hello world"),);
    assert_eq!(collect(false, false, 0), (false, false, 0));
    assert_eq!(collect::<[u8; 9], &str>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}
```

```rust

/// Does the same thing as `id`, but you have to pass in at least one value.
//
//                                                            ╭─ Type-level tuple
//                                                            │  splat/flatten operator.
//                                                           ╭┴──╮
fn id_at_least_one<H, Ts @ ..>(head: H, tail @ ..: Ts) -> (H, ... Ts) {
    //      ╭─ Use value-level splat operator to construct a tuple.
    //      │  This prefix operator works on tuples,
    //      │  and tuple structs whose fields are all visible.
    //    ╭─┴─────╮
    (head, ...tail )
}

#[test]
fn test_id_at_least_one() {
    // assert_eq!(id_at_least_one(), ()); ERROR expected at least 2 parameters, found 1
    assert_eq!(id_at_least_one(3), (3,));
    assert_eq!(id_at_least_one(42, "hello world"), (42, "hello world"));
    assert_eq!(id_at_least_one(false, false, 0), (false, false, 0));
    assert_eq!(id_at_least_one::<[u8; 9], &str>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}

/// Returns a tuple wrapping all its arguments in `Some`.
//
//                                      ╭─ Map each type `T` in the tuple to a corresponding `Option<T>`.
//                                      │  This type-level for-in operator evaluates to a tuple type.
//                                      │  TODO: bikeshed syntax.
//                                     ╭┴─────────────────────╮
fn option_wrap<Ts @ ..>(ts @ ..: Ts) -> for<T in Ts> Option<T> {
    // `static for` loops over tuples and tuple structs
    // are expressions that evaluate to a tuple.
    //
    // Variadic `for` does not support `break`, but you can `continue` with a value.
    //
    // TODO: bikeshed syntax.
    let wrapped: for<T in Ts> Option<T> = static for t in ts {
        Some(t)
    };

    return wrapped;
}

#[test]
fn test_option_wrap() {
    assert_eq!(option_wrap(), ());
    assert_eq!(option_wrap(3), (Some(3),));
    assert_eq!(option_wrap(42, "hello world"), (Some(42), Some("hello world")));
    assert_eq!(id_at_least_one(false, false, 0), (Some(false), Some(false), Some(0)));
    assert_eq!(option_wrap::<[u8; 9], &str>(b"t u r b o", "f i s h"), (Some(b"t u r b o"), Some("f i s h")));
}

fn swing_around<H, Ts @ ..>(head: H, tail @ ..: Ts) -> (...Ts, H) {
    (...tail, head)
}

#[test]
fn test_swing_around() {
    // assert_eq!((), swing_around()); ERROR expected at least 2 parameters, found 1
    assert_eq!(swing_around(3), (3,));
    assert_eq!(swing_around(42, "hello world"), ("hello world", 42));
    assert_eq!(swing_around(false, false, 0), (0, false, false));
    assert_eq!(swing_around::<[u8; 9], &str>(b"t u r b o", "f i s h"), ("f i s h", b"t u r b o"));
}

fn add_one_on<Ts @ ..>(ts @ ..: Ts) -> (...Ts, u32) {
    (...ts, 17)
}

#[test]
fn test_add_one_on() {
    assert_eq!(add_one_on(), (17,),);
    assert_eq!(add_one_on(3), (3, 17));
    assert_eq!(add_one_on(42, "hello world"), (42, "hello world", 17));
    assert_eq!(add_one_on(false, false, 0), (false, false, 0, 17));
    assert_eq!(add_one_on::<[u8; 9], &str>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h", 17));
}


fn bounds<Ts @ ..>() -> Ts
where
    // TODO: bikeshed syntax.
    for<T in Ts> T: Default
{
    // Like `static for` from earlier, but over a set of types instead of values.
    // TODO: bikeshed syntax.
    static for type T in Ts {
        T::default()
    }
}

#[test]
fn test_bounds() {
    assert_eq!(bounds::<>(), ());
    assert_eq!(bounds::<u32>(), (0,));
    assert_eq!(bounds::<u32, bool>(), (0, false));
}

/// Yes, variadic parameters can come in front of non-variadic ones.
fn is_last_3<Ts @ .., Last: PartialEq<i32>>(tuple @ ..: Ts, last: Last) -> (...Ts, bool) {
    (...tuple, last == 3)
}

#[test]
fn test_is_last_3() {
    assert_eq!(is_last_3("yeah", "yeah", 3), ("yeah", "yeah", true));
}

// ERROR ambiguous variadic generics (where does one set end and the other start.?)
// fn doesnt_work<Ts @ .., Us @ ..>() {}

// # Destructuring tuples

fn add_one_on_2<H, Ts @ ..>(head: H, tail: (...Ts,)) -> (...Ts, H) {
    // `@` binding pattern with `..` on a tuple produces a variadic binding,
    // that can be used like a variadic function parameter.
    let (tail @ ..) = tail;

    (...tail, head)
}

fn add_one_on_3<H, Ts @ ..>(head: H, tail: (...Ts,)) -> (...Ts, H) {
    // can also splat tuples and tuple structs directly
    (...tail, head)
}

// # ADTs and traits

// ## Tuple structs
// a lot of this syntax applies to plain old tuples too

struct Tup<Ts @ ..>(i32, ...Ts);

impl<Ts @ ..> Tup<..Ts> {
    fn new(i: i32, ts @ ..: Ts) {
        Self(i, ...ts)
    }

    fn destructure(&self) {
        let Tup(ref head, ref tail @ ..) = self;
        assert!(*head > 3);
        for t in tail {
            println!("blah");
        }
    }
}

impl<Ts @ ..> Tup<i32, ...Ts> {
    fn debug(&self) {
        dbg!(self.0);
        // dbg!(self.1); ERROR: that field might not exist
    }
}

// `iter::Zip`

pub struct Zip<Is @ ..> {
    is: (...Is,),
}

impl<Is @ ..> Zip<...Is,> {
    fn new(iters @ ..: Is) {
        Self {
            is: (...iters,)
        }
    }
}

impl<Is @ ..> Iterator for Zip<...Is,>
where
    Is: Iterator,
{
    // Short form of `(...for<I in Is> I::Item,)`
    type Item = (...Is::Item,);

    #[inline]
    fn next(&mut self) -> Option<Self::Item> {
        let next: Self::Item = for i in ...self.is { i.next()? };
        Some(next)
    }
}

// `futures::join`

use  futures_util::future::{MaybeDone, maybe_done};

#[pin_project::project]
pub struct Join<Futs @ ..: Future> {
    #[pin]
    futs: (...MaybeDone<Futs>,),
}

impl<Futs @ ..> Join<...Futs,> {
    fn new(futures @ ..: Futs) {
        Self {
            futs: for future in future {
                maybe_done(future)
            }
        }
    }
}

impl<Futs @ ..: Future> Future for Join<...Futs,> {
    type Output = (...Futs::Output,);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut all_done = true;

        // TODO: the ergonomics around pin projection need work.

        let futs = self.futures.project();

        //                   ╭─ "inferred variadic params," variadic version of `_`.
        //                   │  TODO: should trailing comma be required?
        //                  ╭┴─╮
        let futs_mut: &mut ( .. ,) = unsafe { self.futures.get_unchecked_mut() };

        for fut in futs_mut {
            let fut = unsafe { Pin::new_unchecked(&mut fut) };
            all_done &= fut.poll(cx).is_ready();
        }

        if all_done {
            let ready = for fut in futs_mut {
                let fut = unsafe { Pin::new_unchecked(&mut fut) };
                fut.take_output().unwrap()
            };

            Poll::Ready(ready)
        } else {
            Poll::Pending
        }
    }
}

// `FnOnce`

trait FnOnce<Args @ ..> {
    type Output;
    fn call_once(self, args @ ..: Args) -> Self::Output;;
}

struct Foo;

// No variadic syntax here!
impl FnOnce<u32, i32, i32> for Foo {
    type Output = u32;

    fn call_once(self, a: u32, b: i32, c: i32) -> u32 {
        a + ((b + c) as u32)
    }
}

// # Multiple sets of variadic parameters
//
// This is where things start getting tricky.
// TODO: do we need all of this?
// Could we decide not to support these cases—
// or will that lead to API composability issues down the road?

/// Takes a variadic set of mutable references to tuples,
/// returns those arguments collected into a tuple.
///
///                 ╭─ means "I accept some number of lifetimes,
///                 │  then the same number of `T`s, then the same number of `U`s"
///                 │  TODO: needs bikeshed.
///               ╭─┴───────────╮
fn unzip_mut_refs< ('as, Ts, Us) @ ..>(
         // very ugly, but there is a shorthand: `&'as mut (Ts, Us)`
    a @ ..: for<'a, T, U in 'as, Ts, Us> &'a mut (T, U)
  // Shorthand here is `((...&'as mut Ts,), (...&'as mut Us,))`
) -> ((...for<'a, T in 'as, Ts> &'a mut T,),  (...for<'a, U in 'as, Us> &'a mut U,)) {
    let left = for &mut (t, _) in a {
        &mut t
    };

    let right = for &mut (_, u) in a {
        &mut u
    };

    (left, right)
}

#[test]
fn test_unzip_mut_refs() {
    let mut a = (3_u32, false);
    let mut b = (2.0_f64, "hi");
    let mut c = ('c', -4_i32);

    // Note the order in which the generics are specified.
    // However, if we were turbofishing lifetime parameters too,
    // those would all be at the front.
    let (left, right): ((&mut u32, &mut f64, &mut char), (&mut bool, &mut &'static str, &mut i32)) =
        unzip_mut_refs::<u32, bool, f64, &'static str, char, i32>(&mut a, &mut b, &mut c);
}

trait Foo<T @ ..> {
    fn join(&self, args @ ..: T);
}

impl<'f, F: Debug, ('as, Ts: Debug, Us: Debug) @ ..> Foo<
    &'f F,
    ...for<'a, T in 'as, Ts> &'a T
> for Tup<...Us,> {
    ///                        ╭─ new syntax: < > brackets enclose a list
    ///                        │  of types, turns it into a variadic type set
    ///                        │  TODO: ugly
    ///                      ╭─┴─────────────────────────────────────╮
    fn join(&self, args @ ..: <&'f F, ...for<'a, T in 'as, Ts> &'a T> ) {
        let Tup(ref us @ ..) = self;
        for left, right in args, us {
            dbg!(left);
            dbg!(right);
        }
    }
}

fn nested
```
