- Feature Name: Arbitray Self Types 2.0
- Start Date: 2023-05-04
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow types that implement the new `trait Receiver<Target=Self>` to be the receiver of a method.

# Motivation
[motivation]: #motivation

Today, methods can only be received by value, by reference, by mutable reference, or by one of a few blessed smart pointer types from `libstd` (`Arc<Self>`, `Box<Self>`, `Pin<Self>` and `Rc<Self>`).
This has always intended to be generalized to support any kind of pointer, such as an `CustomPtr<Self>`. Since late 2017, it has been available on nightly under the `arbitrary_self_types` feature for types that implement `Deref<Target=Self>`.

Because different kinds of "smart pointers" can constrain the semantics in non trivial ways, traits can rely on certain assumptions about the receiver of their method. Just implementing the trait *for* a smart pointer doesn't allow that kind of reliance.

One relevant use-case is cross-language interop (JavaScript, Python, C++), where other languages' equivalents can’t guarantee the aliasing semantics required of a Rust reference. Another case is when the existence of a reference to a thing is, itself, meaningful — for example, reference counting, or if relayout of a UI should occur each time a mutable reference ceases to exist.

In theory, users can define their own smart pointers. In practice, they're second-class citizens compared to the smart pointers in Rust's standard library. User-defined smart pointers to `T` can accept method calls only if the receiver (`self`) type is `&T` or `&mut T`, which causes us to run right into the "reference semantics are not right" problem that we were trying to avoid.

This RFC proposes to loosen this restriction to allow custom smart pointer types to be accepted as a `self` type just like for the standard library types.

See also [this blog post](https://medium.com/@adetaylor/the-case-for-stabilizing-arbitrary-self-types-b07bab22bb45), especially for a list of more specific use-cases.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When declaring a method, users can declare the type of the `self` receiver to be any type `T` where `T: Receiver<Target = Self>` or `Self`.

The `Receiver` trait is simple and only requires to specify the `Target` type to be resolved to:

```rust
trait Receiver {
    type Target: ?Sized;
}
```

The `Receiver` trait is already implemented for a few types from the standard, i.e.
- smart pointer: `Arc<Self>`, `Box<Self>`, `Pin<Self>` and `Rc<Self>`
- references: `&Self` and `&mut Self`
- raw pointer: `*const Self` and `*mut Self`

Shorthand exists for references, so that `self` with no ascription is of type `Self`, `&self` is of type `&Self` and `&mut self` is of type `&mut Self`.

All of the following self types are valid:

```rust
trait Foo {
    fn by_value(self /* self: Self */);
    fn by_ref(&self /* self: &Self */);
    fn by_ref_mut(&mut self /* self: &mut Self */);
    fn by_box(self: Box<Self>);
    fn by_rc(self: Rc<Self>);
    fn by_custom_ptr(self: CustomPtr<Self>);
}

struct CustomPtr<T>(*const T);

impl<T> Receiver for CustomPtr<T> {
    type Target = T;
}
```

## Recursive arbitrary receivers

Receivers are recursive and therefore allowed to be nested. If type `T` implements `Receiver<Target=U>`, and type `U` implements `Receiver<Target=Self>`, `T` is a valid receiver (and so on outward).

For example, this self type is valid:

```rust
impl MyType {
     fn by_box_to_rc(self: Box<Rc<Self>>) { ... }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `Receiver` trait is made public (removing its `#[doc(hidden)])` attribute), exposing it under `core::ops`.
This trait marks types that can be used as receivers other than the `Self` type of an impl or trait definition.

```rust
pub trait Receiver {
    type Target: ?Sized;
}
```

For a type to be a valid receiver for a given impl, the `Self` type of the impl or trait has to appear in its receiver chain.

The receiver chain is constructed as follows:
1. Add the receiver type `T` to the chain.
2. If `T` implements `Receiver`, add `<T as Receiver>::Target` to the chain.
3. Repeat step 2 with the `<T as Receiver>::Target` as `T`.

## Method Resolution

Method resolution will be adjusted to take into account a receiver's receiver chain by taking looking through all impls for all types appearing in the receiver chain.

## core lib changes

The implementations of the hidden `Receiver` trait will be adjusted by removing the `#[doc(hidden)]` attribute and having their corresponding `Target` associated type added.

## Lifetime Elision

TODO

## Diagnostics

TODO ensure we include some analysis of extra diagnostics required. Known gotchas:
- In a trait, using `self: SomeSmartPointerWhichOnlySupportsSizedTypes<Self>` without using `where Self: Sized` on the trait definition results in poor diagnostics.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Implementations of this trait may obscure what method is being called where in similar ways to deref coercions obscuring it.
- The use of this feature together with `Deref` implementations may cause ambiguious situations,
    ```rs

    pub struct Ptr<T>(T);

    impl<T> Deref<T> {
        type Target = T;

        fn deref(&self) -> &T {
            &self.0
        }
    }

    impl<T> Ptr<T> {
        pub fn foo(&self) {}
    }

    pub struct Bar;

    impl Bar {
        fn foo(self: &Ptr<Self>) {}
    }
    ```
    invoking `Ptr(Bar).foo()` here will require the use of fully qualified paths (`Bar::foo` vs `Ptr::foo` or `Ptr::<T>::foo`) to disambiguate the call.

## Object safety

Receivers remain object safe as before, if they implement the `core::ops::DispatchFromDyn` trait they are object safe.
As not all receivers might want to permit object safety (or are even unable to support it), object safety should remain being encoded in a different trait than the here proposed `Receiver` trait.


TODO. To include:

- compatibility concerns / does this break semver?
- method shadowing
- all the other concerns discussed in https://github.com/rust-lang/rust/issues/44874#issuecomment-1306142542


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- An alternative proposal is to use `Deref` and it's associated `Target` type instead. This comes with the problem that not all types that wish to be receivers are able to implement `Deref`, a simple example being raw pointers.
- Change the trait definition to
    ```rust
    pub trait Receiver<T: ?Sized> {}
    ```
    to allow an impl of the form `impl Receiver<T> for T` which would enable `Self` to be used as is by the trait impl rule instead of a special case.


TODO:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

To include:

- Arguments for/against doing this as a new `MethodReceiver` trait
- Arguments for/against stabilizing the unstable `Receiver` trait instead of this
- Arguments for/against doing a pointer-based `Deref` or `MethodReceiver` at the same time as this, or whether it truly can be orthogonal (as discussed at https://github.com/rust-lang/rust/issues/44874#issuecomment-1483607125).
- In particular, if we later implement a blanket `MethodReceiver` for `Deref` then we run into compatibility breakges, so we need to figure out if we'd do that

# Prior art
[prior-art]: #prior-art

A previous PR based on the `Deref` alternative has been proposed before https://github.com/rust-lang/rfcs/pull/2362.

TODO:

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

- There is the option of doing a blanket impl of the `Receiver` trait based on `Deref` impls delegating the `Target` types.

- With the proposed design, it is not possible to be generic over the receiver while permitting the plain `Self` to be slotted in:
    ```rs
    use std::ops::Receiver;

    struct Foo(u32);
    impl Foo {
        fn get<R: Receiver<Target=Self>>(self: R) -> u32 {
            self.0
        }
    }

    fn main() {
        let mut foo = Foo(1);
        foo.get::<&Foo>(); // Error
    }
    ```
    This fails, because `T: Receiver<Target=T>` generally does not hold.
    An alternative would be to lift the associated type into a generic type parameter of the `Receiver` trait, that would allow adding a blanket `impl Receiver<T> for T` without overlap.

TODO:

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

TODO
