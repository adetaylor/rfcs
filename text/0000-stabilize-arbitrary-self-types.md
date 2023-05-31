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
/// # SAFETY
/// - the type needs to be ABI-compatible with a fat pointer
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

## Object safety

Receivers are object safe if they implement the (unstable) `core::ops::DispatchFromDyn` trait.

As not all receivers might want to permit object safety or are unable to support it. Therefore object safety should remain being encoded in a different trait than the here proposed `Receiver` trait, likely `DispatchFromDyn`.

Since `DispatchFromDyn` is unstable at the moment, object-safe receivers might be delayed until `DispatchFromDyn` is stabilized. `Receiver` and `DispatchFromDyn` can be stabilized together, but `Receiver` is not blocked in this, since non-object-safe receivers already cover a big chunk of the use-cases.

## Lifetime Elision

TODO

## Diagnostics

TODO ensure we include some analysis of extra diagnostics required. Known gotchas:
- In a trait, using `self: SomeSmartPointerWhichOnlySupportsSizedTypes<Self>` without using `where Self: Sized` on the trait definition results in poor diagnostics.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Implementations of this trait may obscure what method is being called where in similar ways to deref coercions obscuring it.
- The use of this feature together with `Deref` implementations may cause ambiguious situations. Invoking `Ptr(Bar).foo()` will require the use of fully qualified paths (`Bar::foo` vs `Ptr::foo` or `Ptr::<T>::foo`) to disambiguate the call.
    ```rs
    use std::ops::{Deref, Receiver};

    pub struct Ptr<T>(T);

    impl<T> Deref for Ptr<T> {
        type Target = T;

        fn deref(&self) -> &T {
            &self.0
        }
    }

    impl Receiver for Ptr<T> {
        type Target = T;
    }

    impl<T> Ptr<T> {
        pub fn foo(&self) {
            println!("hip")
        }
    }

    pub struct Bar;

    impl Bar {
        fn foo(self: &Ptr<Self>) {
            println!("hop")
        }
    }

    fn main() {
        let a = Ptr(Bar);
        a.foo(); // hip or hop? error[E0034]: multiple applicable items in scope
        Ptr::foo(a); // hip
        Bar::foo(a); // hop
    }
    ```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Deref-based

The current `Deref`-based implementation is not sufficient, because there are types which can not implement `Deref`, but would be good candidates to be used as self types (for example raw pointers). Therefore there is a need for the `Receiver` trait.

In theory we could use both traits, so a type can be used as a receiver if it implements `Deref` or `Receiver`. All the types that can implement `Deref` do so. All the types that cannot implement `Deref` implement `Receiver`. There could be a blanked implementation `impl<T> Receiver for T where T: Deref`. 

The advantage of that would be, that there is a vast amount of types that implement `Deref` today, which then could immediately be used as self types.

But there are some concerns with using both traits.
Firstly, it makes the feature more complicated, because it is not one, but two traits and it might be unclear when to choose which of the two.
Secondly, since so many types already implement `Deref`, adding functionality to it (in this case enabling types as method receivers) bears the risk of breaking someones types. But so far we could not identify any possiblities where this would be the case.

## Generic parameter

Change the trait definition to have a generic parameter instead of an associated type:

```rust
pub trait Receiver<T: ?Sized> {}
```

to allow an impl of the form `impl Receiver<T> for T`. This would enable `Self` to be used as is by the trait impl rule instead of a special case.

## Not do it

As always there is the option to not do this. But this feature already kind of half-exists (I am talking about `Box`, `Pin` etc.) and it makes a lot of sense to also take the last step and therefore enable non-libstd types to be used as self types.

# Prior art
[prior-art]: #prior-art

A previous PR based on the `Deref` alternative has been proposed before https://github.com/rust-lang/rfcs/pull/2362.

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

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC is an example of replacing special casing aka. compiler magic with clear and transparent definitions. We believe this is a good thing and should be done whenever possible.
