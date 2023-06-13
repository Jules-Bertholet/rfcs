# Variadics design sketch

## Use-cases we want to support

- [x] [`iter::zip`](https://doc.rust-lang.org/std/iter/fn.zip.html)
- [x] [`futures::join`](https://docs.rs/futures/latest/futures/future/fn.join.html)
- [x] [`Fn` traits](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) ([GitHub issue](https://github.com/rust-lang/rust/issues/41058))
- [x] Implementing traits for tuples ([stdlib](https://doc.rust-lang.org/std/primitive.tuple.html#trait-implementations), [Sourcegraph](https://sourcegraph.com/search?q=context:global+lang:Rust+impl%5B+%3C%5D+.*+for+%5C%28&patternType=regexp&sm=0&groupBy=repo))
- [x] Implementing traits for function pointers/`dyn Fn` ([stdlib](https://doc.rust-lang.org/std/primitive.fn.html#trait-implementations), [`wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen/blob/f569fddb62cdeb2bb9ba230fe59d3fba143cf92c/src/convert/closures.rs))
- [x] `#![feature(unsized_fn_params)]`
  - But don't penalize `Sized` ergonomics to accomplish it.
- [ ] [`frunk::HList`](https://docs.rs/frunk/latest/frunk/hlist/index.html)
- [ ] [`futures::select`](https://docs.rs/futures/latest/futures/future/fn.select.html)
- [ ] Other [`frunk`](https://docs.rs/frunk/latest/frunk/) use-cases?
- [ ] Compile-time reflection/macro-free `derive`?
- [ ] Suggestions welcome

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
- [Circle](https://github.com/seanbaxter/circle/blob/master/new-circle/README.md#pack-traits)
- [C++](https://en.cppreference.com/w/cpp/language/parameter_pack)
- [D](https://dlang.org/spec/function.html#variadic)
- [Java](https://docs.oracle.com/javase/8/docs/technotes/guides/language/varargs.html)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [PHP](https://www.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)
- [Ruby](https://docs.ruby-lang.org/en/3.2/syntax/calling_methods_rdoc.html#label-Array+to+Arguments+Conversion)
- [Scala](https://docs.scala-lang.org/scala3/reference/changed-features/vararg-splices.html)
- [Swift](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md)
- [Zig](https://ziglang.org/documentation/master/#comptime)

## Other links

- [Analysing variadics, and how to add them to Rust - PoignardAzur](https://poignardazur.github.io/2021/01/30/variadic-generics/)
- [More enum types - Yoshua Wuyts](https://blog.yoshuawuyts.com/more-enum-types/)

## Variadics and values

### `...` operator

`...` ("splat") is a prefix operator that unpacks a parenthesized list of values. It operates on tuples, tuple structs, arrays, and references to them. (For tuple structs, all the fields must be visible at the location where `...` is invoked. Also, if the struct is marked `non_exhaustive`, then splatting only works within the defining crate.)

TODO: should it be called "splat", "spread", or something else?

```rust
let a: (i32, u32) = (3, 4);
let b: (i32, u32, usize) = (...a, 1);
let c: (usize, i32, u32) = (1, ...a);
struct Foo(bool, bool);
let d: (bool, bool) = (...Foo(true, false),); // TODO: should trailing comma be required?
let e: Foo = Foo(...d);
let f: [i32, 3] = [1, ...[2, 3]];
let g: (i32, i32, i32) = (1, ...(2, 3));
let h: Foo: = Foo(...[true, false]);
let i: [i32, 3] = [1, ...(2, 3)];
```

Splatting a reference results in references.

```rust
let j: (i32, &usize, &bool) = (3, ...&(4, false));
let k: (i32, &mut usize, &mut bool) = (3, ...&mut (4, false));
```

### `...` pattern

`...` can also be used in patterns.

#### Arrays

For arrays, `...` patterns serve as an alternative to `ident @ ..`.

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
let &(...head, tail) = t;
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

`static for` loops allow iterating over the elements of a splattable value (tuple, tuple struct, array, references to these). The loop is unrolled at compile-time.

TODO: bikeshed syntax.

```rust
let mut v: Vec<Box<dyn Debug>> = Vec::new();

static for i in (2, "hi", 5.0) {
    v.push(Box::new(i));
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

```rust
static for i in (42, false) {
    if we_are_done() {
        break;
    }

    some_operation(i);
}
```

You can `static for` over multiple splattable values at once, as long as they are the same length.

```rust
let t: ((i8, u8), (i16, u16), (i32, u32)) = static for i, u in (-1, -2, -3), (1, 2, 3) {
    (i, u)
};

assert_eq!(t, ((-1, 1), (-2, 2), (-3, 3)));
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

## Variadics and types



### Generic parameter syntax

Today, generic parameter lists are necessarily flat. At the value level, we can use arrays and tuples, to give structure to the parameters we pass in. But at the type level, this is not possible. More complex uses of variadics may need structured parameter lists, so we suggest syntax for it in the table below. However, the majority of cases will hopefully only use the first row in the table of proposed additions.

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

`static for` can also work with both values and types/lifetimes/consts at the same time:

```rust
let t: ((i32, bool), (u32, usize), (i32, u8)) = static for val, type T in (1, 2, 1), <bool, usize, u8> {
    (val, T::default())
};

assert_eq!(t, ((1, false), (2, 0), (1, 0)));
```

### Type-level `...`

Type-level `...` unpacks a comma-separated list of types, lifetimes, or consts at the type level.

This feature is meant for use with variadic generics, so the concrete examples below will be underwhelming.

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

Once again, for use with variadic generics, so the concrete examples below will be underwhelming.

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

You can put trait bounds on variadic generic parameters.

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

Where clauses are also supported, via for-in.

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

```rust
// ERROR ambiguous variadic generics (where does one set end and the other start?)
// fn doesnt_work<...Ts, ...Us>() {}
```

We now have enough to implement several traits for tuples of any length.

```rust
impl<...Ts: fmt::Debug> fmt::Debug for (...Ts,) {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        let mut list = f.debug_list();

        static for elem in self {
            f.entry(elem);
        }
        
        list.finish()
    }
}
```

```rust
impl<...Ts: Default> Default for (...Ts,) {
    fn default() -> Self {
        static for type T in Ts {
            T::default()
        }
    }
}
```

### `...` patterns in function parameter lists

`...` patterns can also be used in function parameter lists.

As is our tradition, we start with a contrived concrete example before introducing generics.

```rust
//                    ╭─ This binding has tuple type.
//                    │  However, in terms of calling convention,
//                    │  elements are passed individually.
//                    │  TODO: allow `tuple @ ..` as alternative syntax?
//                    │
//                    │           ╭─ No need to splat or parenthesize.
//                    │           │  TODO: bikeshed above statement
//                   ╭┴────────╮ ╭┴─────────╮
fn collect_contrived( ... tuple : <u32, i32> ) -> (u32, i32) {
    tuple
}

// The above and below functions are exactly equivalent, in signature, semantics, and calling convention.

fn collect_contrived(a: u32, b: i32) -> (u32, i32) {
    (a, b)
}
```

When using a `...` binding with unsized argument types via `unsized_fn_params`, there are additional restrictions.

```rust
#![feature(unsized_fn_params)]

fn do_stuff_with_unsized_params(...tuple: <[u32], dyn Debug>) {
    // `tuple` is still a tuple, sort of.
    // But: you aren't allowed to take its address, or pass it to a function by value.
    // The only thing you can do is index into its fields;
    // either directly, via pattern match, or with `static for`.
    // These restrictions won't be lifted even if `([u32], dyn Debug)` gets a defined layout,
    // but they may be with `#![feature(unsized_locals)]`.

    dbg!(&tuple.0[0]);
    dbg!(&tuple.1);
}
```

### Varargs

Combining function parameter `...` and variadic generics gives us varargs.

```rust
/// Takes a variadic set of type arguments,
/// returns those arguments collected into a tuple.
fn collect<...Ts>(...vs: Ts) -> (...Ts,) {
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
fn collect_at_least_one<H, ...Ts>(head: H, ...tail: Ts) -> (H, ...Ts) {
    (head, ...tail)
}

#[test]
fn collect_at_least_one() {
    // assert_eq!(id_at_least_one(), ()); ERROR expected at least 2 parameters, found 1
    assert_eq!(collect_at_least_one(3), (3,));
    assert_eq!(collect_at_least_one(42, "hello world"), (42, "hello world"));
    assert_eq!(collect_at_least_one(false, false, 0), (false, false, 0));
    assert_eq!(collect_at_least_one::<[u8; 9], &str>(b"t u r b o", "f i s h"), (b"t u r b o", "f i s h"));
}
```

### Variadics with ADTs

```rust
struct Tup<...Ts>(i32, ...Ts);

impl<...Ts> Tup<...Ts> {
    fn new(i: i32, ...ts: Ts) {
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

pub struct Zip<...Is>(...Is,)

impl<...Is> Zip<...Is> {
    fn new(...iters: Is) {
        Self(...iters)
    }
}

impl<...Is: Iterator> Iterator for Zip<...Is> {
    type Item = (...for<I in Is> I::Item,);

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

impl<...Futs> Join<...Futs,> {
    fn new(...futures: Futs) {
        let wrapped_futs = static for future in futures {
            maybe_done(future)
        };

        Self(...wrapped_futs)
    }
}

impl<...Futs: Future> Future for Join<...Futs> {
    type Output = (...for<F in Futs > F::Output,);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut all_done = true;

        // TODO: what is the best API for pin projection?

        // Long, annoying type specified for example purposes.
        // In reality, you would infer it with `(..)`.
        let futs: (...for<F in Futs> Pin<&mut F>,) = self.project();

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

```rust
// `FnOnce`

trait FnOnce<...Args> {
    type Output;

    fn call_once(self, ...args: Args) -> Self::Output;
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

### Advanced examples

```rust
// Implement non-symmetric `PartialEq` for tuples

impl<...<Ts: PartialEq<Us>, Us>> PartialEq<(...for<U in Us> U,)> for (...for<T in Ts> T,) {
    fn eq(&self, other: &(...for<U in Us> U,)) {
        static for l, r in self, other {
            if l != r {
                return false;
            }
        }

        true
    }
}
```

```rust
/// Pair of tuples → tuple of pairs
pub fn zip<...<Ts, Us>>(left: (...for<T in Ts> T,), right: (...for<U in Us> U,)) -> (...for<<T, U> in <Ts, Us>> (T, U),) {
    static for l, r in left, right {
        (l, r)
    }
}
```

```rust
/// Tuple of pairs → pair of tuples
pub fn unzip<...<Ts, Us>>(zipped: (...for<<T, U> in <Ts, Us>> (T, U),) -> ((...for<T in Ts> T,), (...for<U in Us> U,)) {
    // This one is a bit mind-bending.
    // TODO: could it be made more intuitive?
    static for ...elems in ...zipped {
        elems
    }
}
```

### TODO: `match` and recursion

`static for` is not the only way to loop over a tuple. You can also use recursion, with `match`.

```rust
/// Debug-print every element of the tuple.
fn recursive_dbg<...Ts: Debug>(ts: (...Ts,)) {
    match ts {
        () => (),
        (head, ...tail) => {
            print!("{head:?}, ");
            recursive_dbg(tail);
        }
    }
}
```

Recursion also allows iterating in a different order.

```rust
/// Debug-print every element of the tuple, in reverse order.
fn recursive_dbg<...Ts: Debug>(ts: (...Ts,)) {
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
fn reverse_tuple<...Ts: Debug>(ts: (...Ts,)) -> _ /* what do we put here? */ {
    match ts {
        () => (),
        (head, ...tail) => {
           (...reverse_tuple(tail), head)
        }
    }
}
```

A more generalized type-level recursion mechanism/`match` would be the most flexible option, but also complex. Alternatively, we could have a set of primitives, like `Rotate<...T>` or `Reverse<...T>`, for type-level operations. Also, what restrictions would need to be imposed to make type inference tractable, and avoid post-monomorphization errors? How would unification work?

It's also not clear how the `match` + recursion approach could be applied to iterating over types/lifetimes/consts at the value level.

#### Type-level `match`

What might type-level `match` look like?

```rust
...type Reverse<...Ts> = match Ts <
    // Restriction: lifetimes can't affect which branch is taken, otherwise you would have unsound specialization
    <> => <>,
    <Head, ...Tail> => <...Reverse<Tail>, Head>,
>;

/// Reverse the order of the elements in the tuple.
fn reverse_tuple<...Ts: Debug>(ts: (...Ts,)) -> (...Reverse<...Ts>) {
    match ts {
        () => (),
        (head, ...tail) => {
           (...reverse_tuple(tail), head)
        }
    }
}
```

### TODO: Pack length constraints

It would be desirable to allow constraining the length of a generic parameter pack. For example, to write a generalized MxN → NxM nested tuple transformation, you need to assert that all the inner tuples are of the same length. Speculative syntax:

```rust
/// Tranform MxN tuple into NxM
//                                   ╭─ Each `Ts` in `Tss` has `N` elements
//                                  ╭┴╮
fn pivot<const N: usize, ...<...Tss; N >>(tuples: (...for<Ts in Tss> (...Ts,),)) {
    static for ...t in ...tuples {
        t
    }
}
```

### TODO: Random access

It would be nice to have a way to express "get the `N`th value of this tuple, if it exists", where `N` is a const generic parameter. One would also need a type-level equivalent for generic parameter packs.

### TODO: Variadic variant lists with enums

With APIs like `futures::select`, you want to pass in N values, and get back a value corresponding to one of the `N` arguments. [Yoshua Wuyt's blog posts](https://blog.yoshuawuyts.com/more-enum-types/) explore this in detail. To support this case, one might want enum types whose variats correspnd to variadic generic parameters. But perhaps the additional complexity is not necessary; for example, in `futures::select` we could instead ask the caller to wrap the result type of their futures in an enm they define.

## TODO

- [ ] Add additional advanced examples
- [ ] Come up with nomenclature ("comma-separated list of things" is not exactly Shakespearean eloquence)
- [ ] Resolve inline bikeshed TODOs
- [ ] Lifetime elision
