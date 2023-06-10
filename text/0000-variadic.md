- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

One paragraph explanation of the feature.

# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation

```rust
// # Basic variadic functions

/// Takes a variadic set of type arguments,
/// returns those arguments collected into a tuple.
/// Variadic value sets match the `..` pattern, `...` is the prefix splat operator.
//
//      ╭─ Declare variadic generic parameter of zero or more types.
//      │          ╭─ Declare variadic function parameter of zero or more values.
//      │          │                 ╭─ Splat (expand) into tuple (type level).
//      │          │                 │  TODO: must trailing comma be necessary?
//    ╭─┴─────╮  ╭─┴──────────╮    ╭─┴──────────╮
fn id< Ts @ .. >( vs @ .. : Ts ) -> ( ... Ts , ) {
    // `vs` is a variadic binding.
    // The type of `vs` is the variadic typeset `Ts`.
    // Notably, `vs` is *not* a tuple.
    // However, it coerces to one (as long as every type in `Ts` is `Sized`).
    vs
}

#[test]
fn test_id() {
    assert_eq!((), id());
    assert_eq!((3,), id(3));
    assert_eq!((42, "hello world"), id(42, "hello world"));
    assert_eq!((false, false, 0), id(false, false, 0));
    assert_eq!((b"t u r b o", "f i s h"), id::<[u8; 9], &str>(b"t u r b o", "f i s h"));
}

/// Does tha same thing as `id`, but you have to pass in at least one value.
//
//                                                         ╭─ Trailing comma unnecessary
//                                                         │  as preceding comma shows
//                                                         │  this is a list
//                                                       ╭─┴────────╮
fn id_at_least_one<H, Ts @ ..>(head: H, tail @ ..: Ts) -> (H, ...Ts) {
    //      ╭─ Use value-level splat operator to construct a tuple.
    //      │  This prefix operator works on variadic bindings, tuples,
    //      │  and tuple structs whose fields are all visible.
    //    ╭─┴─────╮
    (head, ...tail )
}

#[test]
fn test_id_at_least_one() {
    // assert_eq!((), id_at_least_one()); ERROR expected at least 2 parameters, found 1
    assert_eq!((3,), id_at_least_one(3));
    assert_eq!((42, "hello world"), id_at_least_one(42, "hello world"));
    assert_eq!((false, false, 0), id_at_least_one(false, false, 0));
    assert_eq!((b"t u r b o", "f i s h"), id_at_least_one::<[u8; 9], &str>(b"t u r b o", "f i s h"));
}

/// Returns a tuple wrapping all its arguments in `Some`.
//
//                                             ╭─ Map each `T` to an `Option<T>`.
//                                             │  There is a shorthand/sugar form: `Option<Ts>`.
//                                             │  Why do we even want a long version, you ask?
//                                             │  That will be discussed later.
//                                           ╭─┴────────────────────╮
fn option_wrap<Ts @ ..>(ts @ ..: Ts) -> ( ... for<T in Ts> Option<T> , ) {
    // `static for` loops over variadic bindings
    // are expressions that evaluate to a variadic set of values.
    // In this example, said value is immediately coerced to a tuple.
    //
    // Variadic `for` does not support `break`, but you can `continue` with a value.
    //
    // TODO: this is highly bikesheddable.
    let wrapped: (...Option<Ts>,) = static for t in ts {
        Some(t)
    };

    return wrapped;
}

#[test]
fn test_option_wrap() {
    assert_eq!((), option_wrap());
    assert_eq!((Some(3),), option_wrap(3));
    assert_eq!((Some(42), Some("hello world")), option_wrap(42, "hello world"));
    assert_eq!((Some(false), Some(false), 0), id_at_least_one(false, false, 0));
    assert_eq!((Some(b"t u r b o"), Some("f i s h")), option_wrap::<[u8; 9], &str>(b"t u r b o", "f i s h"));
}

fn swing_around<H, Ts @ ..>(head: H, tail @ ..: Ts) -> (...Ts, H) {
    (...tail, head)
}

#[test]
fn test_swing_around() {
    // assert_eq!((), swing_around()); ERROR expected at least 2 parameters, found 1
    assert_eq!((3,), swing_around(3));
    assert_eq!(("hello world", 42), swing_around(42, "hello world"));
    assert_eq!((0, false, false), swing_around(false, false, 0));
    assert_eq!(("f i s h", b"t u r b o"), swing_around::<[u8; 9], &str>(b"t u r b o", "f i s h"));
}

fn add_one_on<Ts @ ..>(ts @ ..: Ts) -> (...Ts, u32) {
    (...ts, 17)
}

#[test]
fn test_add_one_on() {
    assert_eq!((17,), add_one_on());
    assert_eq!((3, 17), add_one_on(3));
    assert_eq!((42, "hello world", 17), add_one_on(42, "hello world"));
    assert_eq!((false, false, 0, 17), add_one_on(false, false, 0));
    assert_eq!((b"t u r b o", "f i s h", 17), add_one_on::<[u8; 9], &str>(b"t u r b o", "f i s h"));
}

///         ╭─ Every type in the set must meet the trait bound.
///         │  Shorthand for `where for<T in Ts> T: Default`
///       ╭─┴───────────────╮
fn bounds< Ts @ .. : Default >() -> (...Ts,) {
    // Type version of `static for` from earlier.
    // TODO: `static for type`? needs bikeshed.
    static for type T in Ts {
        T::default()
    }
}

#[test]
fn test_bounds() {
    assert_eq!((), bounds());
    assert_eq!((0,), bounds::<u32>());
    assert_eq!((0, false), bounds::<u32, bool>());
}

/// Yes, variadic parameters can come in front of non-variadic ones.
fn assert_last_is_3<Ts @ .., Last: PartialEq<i32>>(tuple: (...Ts, Last)) -> (...Ts,) {
    match tuple {
        (front @ .., back) where back == 3 => front,
        _ => pamic!("last element of tuple is not 3!"),
    }
}

// # Stuff that doesn't work

// ERROR ambiguous variadic generics (where does one set end and the other start.?)
// fn doesnt_work_either<Ts @ .., Us @ ..>() {}

// ERROR variadic type parameters can't be combined with default type parameters
// (though a vaiadic parameter can itself have a default.)
// TODO this limitation could cause problems...
//fn also_forbidden<Ts @ .., Defaulted = i32>() {}

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

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

Why should we *not* do this?

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art

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

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities

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
