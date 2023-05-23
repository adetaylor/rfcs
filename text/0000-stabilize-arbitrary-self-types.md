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

When declaring a method, users can declare the type of the `self` receiver to be any type `T` where `T: Receiver<Target = Self>`.

The `Receiver` trait is simple and only requires to specify the `Target` type to be resolved to:

```rust
trait Receiver {
    type Target: ?Sized;
}
```

The `Receiver` trait is already implemented for a few types from the standard, i.e.
- smart pointer: `Arc<Self>`, `Box<Self>`, `Pin<Self>` and `Rc<Self>`
- references: `&Self`, `&mut Self` and `Self`
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

Receivers are recursive and therefore allowed to be nested. If type `T` implements `Receiver<Target=U>`, and type `U` implements `Deref<Target=Self>`, `T` is a valid receiver (and so on outward).

For example, this self type is valid:

```rust
impl MyType {
     fn by_box_to_rc(self: Box<Rc<Self>>) { ... }
}
```

## Object safety

TODO:
- DispatchFromDyn
- What is the relation with Lukas' [RFC "Unsize and CoerceUnsize v2"](https://github.com/ferrous-systems/unsize-experiments)?

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

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

TODO. To include:

- object safety
- compatibility concerns / does this break semver?
- does it cause any risk that folks implementing `Deref` for other reasons will also be forced into new behavior? ("scoping")
- method shadowing
- if we later implement `MethodReceiver` to allow 
- all the other concerns discussed in https://github.com/rust-lang/rust/issues/44874#issuecomment-1306142542


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

TODO. To include:

- Arguments for/against doing this as a new `MethodReceiver` trait
- Arguments for/against stabilizing the unstable `Receiver` trait instead of this
- Arguments for/against doing a pointer-based `Deref` or `MethodReceiver` at the same time as this, or whether it truly can be orthogonal (as discussed at https://github.com/rust-lang/rust/issues/44874#issuecomment-1483607125).
- In particular, if we later implement a blanket `MethodReceiver` for `Deref` then we run into compatibility breakges, so we need to figure out if we'd do that

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

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

TODO

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
