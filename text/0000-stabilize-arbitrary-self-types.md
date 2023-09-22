- Feature Name: Arbitrary Self Types 2.0
- Start Date: 2023-05-04
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow types that implement the new `trait Receiver<Target=Self>` to be the receiver of a method.

# Motivation
[motivation]: #motivation

Today, methods can only be received by value, by reference, or by one of a few blessed smart pointer types from `core`, `alloc` and `std` (`Arc<Self>`, `Box<Self>`, `Pin<P>` and `Rc<Self>`).

It's been assumed that this will eventually be generalized to support any smart pointer, such as an `CustomPtr<Self>`. Since late 2017, it has been available on nightly under the `arbitrary_self_types` feature for types that implement `Deref<Target=Self>` and for raw pointers.

One use-case is cross-language interop (JavaScript, Python, C++), where other languages' references can’t guarantee the aliasing and exclusivity semantics required of a Rust reference. For example, the C++ `this` pointer can't be practically or safely represented as a Rust reference because C++ may retain other pointers to the data and it might mutate at any time. Yet, calling C++ methods intrinsically requires a `this` reference.

Another case is when the existence of a reference is, itself, semantically important — for example, reference counting, or if relayout of a UI should occur each time a mutable reference ceases to exist. In these cases it's not OK to allow a regular Rust reference to exist, and yet sometimes we still want to be able to call methods on a reference-like thing.

In theory, users can define their own smart pointers. In practice, they're second-class citizens compared to the smart pointers in Rust's standard library. User-defined smart pointers to `T` can accept method calls only if the receiver (`self`) type is `&T` or `&mut T`, which isn't acceptable if we can't safely create native Rust references to a `T`.

This RFC proposes to loosen this restriction to allow custom smart pointer types to be accepted as a `self` type just like for the standard library types.

See also [this blog post](https://medium.com/@adetaylor/the-case-for-stabilizing-arbitrary-self-types-b07bab22bb45), especially for a list of more specific use-cases.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When declaring a method, users can declare the type of the `self` receiver to be any type `T` where `T: Receiver<Target = Self>` or `Self`.

The `Receiver` trait is simple and only requires to specify the `Target` type:

```rust
trait Receiver {
    type Target: ?Sized;
}
```

The `Receiver` trait is already implemented for a few types from the standard:
- smart pointers in the standard library: `Rc<Self>`, `Arc<Self>`, `Box<Self>`, and `Pin<Ptr<Self>>`
- references: `&Self` and `&mut Self`
- pointers: `*const Self` and `*mut Self`

Shorthand exists for references, so that `self` with no ascription is of type `Self`, `&self` is of type `&Self` and `&mut self` is of type `&mut Self`.

All of the following self types are valid:

```rust
impl Foo {
    fn by_value(self /* self: Self */);
    fn by_ref(&self /* self: &Self */);
    fn by_ref_mut(&mut self /* self: &mut Self */);
    fn by_ptr(*const Self);
    fn by_mut_ptr(*mut Self);
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

## `core` libs changes

The `Receiver` trait is made public (removing its `#[doc(hidden)])` attribute), exposing it under `core::ops`. It gains a `Target` associated type.

This trait marks types that can be used as receivers other than the `Self` type of an impl or trait definition.

```rust
pub trait Receiver {
    type Target: ?Sized;
}
```

A blanket implementation is provided for any type that implements `Deref`:

```rust
impl<P: ?Sized, T: ?Sized> Receiver for P
where
    P: Deref<Target = T>,
{
    type Target = T;
}
```

(See [alternatives](#no-blanket-implementation) for discussion of the tradeoffs here.)

It is also implemented for `&T`, `&mut T`, `*const T` and `*mut T`.

## Compiler changes

The existing Rust [reference section for method calls describes the algorithm for assembling method call candidates](https://doc.rust-lang.org/reference/expressions/method-call-expr.html). This algorithm changes in one simple way: instead of dereferencing types (using the `Deref<Target=T>`) trait, we use the new `Receiver<Target=T>` trait to determine the next step.

Because a blanket implementation is provided for users of the `Deref` trait and for `&T`/`&mut T`, the net behavior is similar. But this provides the opportunity for types which can't implement `Deref` to act as method receivers.

Dereferencing a raw pointer usually needs `unsafe` (for good reason!) but in this case, no actual dereferencing occurs. This is used only to determine a list of method candidates; no memory access is performed and thus no `unsafe` is needed.

## Object safety

Receivers are object safe if they implement the (unstable) `core::ops::DispatchFromDyn` trait.

As not all receivers might want to permit object safety or are unable to support it, object safety should remain being encoded in a different trait than the here proposed `Receiver` trait, likely `DispatchFromDyn`.

This RFC does not propose any changes to `DispatchFromDyn`. Since `DispatchFromDyn` is unstable at the moment, object-safe receivers might be delayed until `DispatchFromDyn` is stabilized. `Receiver` is not blocked on further `DispatchFromDyn` work, since non-object-safe receivers already cover a big chunk of the use-cases.

## Lifetime elision

As discussed in the [motivation](#motivation), this new facility is _most likely_ to be used in cases where a standard reference can't normally be used. But in other cases a smart pointer self type might wrap a standard Rust reference, and thus might be parameterized by a lifetime.

Lifetime elision works in the expected fashion:

```rust
struct SmartPtr<'a, T: ?Sized>(&'a T);

impl<'a, T: ?Sized> Receiver for SmartPtr<'a, T> {
    type Target = T;
}

struct MyType;

impl MyType {
    fn m(self: SmartPtr<Self>) {}
    fn n(self: SmartPtr<'_, Self>) {}
    fn o<'a>(self: SmartPtr<'a, Self>) {}
}
```

## Diagnostics

The existing branches in the compiler for "arbitrary self types" already emit excellent diagnostics. We will largely re-use them, with the following improvements:

- In the case where a self type is invalid because it doesn't implement `Receiver`, the existing excellent error message will be updated.
- An easy mistake is to implement `Receiver` for `P<T>`, forgetting to specify `T: ?Sized`. `P<Self>` then only works as a `self` parameter in traits `where Self: Sized`, an unusual stipulation. It's not obvious that `Sized`ness is the problem here, so we will identify this case specifically and produce an error giving that hint.
- There are certain types which feel like they "should" implement `Receiver` but do not: `Weak` and `NotNull`. If these are encountered as a self type, we should produce a specific diagnostic explaining that they do not implement `Receiver` and suggesting that they could be wrapped in a newtype wrapper if method calls are important. This will require `Weak` and `NonNull` be marked as lang items so that the compiler is aware of the special nature of these types. (The authors of this RFC feel that these extra lang-items _are_ worthwhile to produce these improved diagnostics - if the reader disagrees, please let us know.)
- Under some circumstances, the compiler identifies method candidates but then discovers that the self type doesn't match. This results currently in a simple "mismatched types" error; we can provide a more specific error message here. The only known case is where a method is generic over `Receiver`, and the caller explicitly specifies the wrong type:
    ```rust
    #![feature(receiver_trait)]

    use std::ops::Receiver;

    struct SmartPtr<'a, T: ?Sized>(&'a T);

    impl<'a, T: ?Sized> Receiver for SmartPtr<'a, T> {
        type Target = T;
    }

    struct Foo(u32);
    impl Foo {
        fn a<R: Receiver<Target=Self>>(self: R) { }
    }

    fn main() {
        let foo = Foo(1);
        let smart_ptr = SmartPtr(&foo);
        smart_ptr.a(); // this compiles
        smart_ptr.a::<&Foo>(); // currently results in "mismatched types"; we can probably do better
    }
    ```
- If a method `m` is generic over `R: Receiver<Target=T>` (or, perhaps more commonly, `R: Deref<Target=T>`) and `self: R`, then someone calls it with `object_by_value.m()`, it won't work because Rust doesn't know to use `&object_by_value`, and the message `the trait bound Foo: 'Receiver/Deref' is not satisfied` is generated. While correct, this may be surprising because users expect to be able to use `object_by_value.m2()` where `fn m2(&self)`. The resulting error message already suggests that the user create a reference in order to match the `Receiver` trait, so this may be sufficient already, but we may add an additional note here.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Deref coercions can already be confusing and unexpected. `Deref` becomes more powerful and significant if it allows method calls.
- Custom smart pointers are a niche use case (but they're very important for cross-language interoperability.)

## Method shadowing
[method-shadowing]: #method-shadowing

For a smart pointer `P<T>`, a method call `p.m()` might call a method on the smart pointer type itself (`P::m`), or, if the smart pointer implements `Deref<Target=T>`, it might already call `T::m`. This already gives the possibility that `T::m` would be shadowed by `P::m`.

Current Rust standard library smart pointers are designed with this shadowing behavior in mind:

* `Box`, `Pin`, `Rc` and `Arc` already heavily use associated functions rather than methods
* Where they use methods, it's often with the _intention_ of shadowing a method in the inner type (e.g. `Arc::clone`)

These method shadowing risks are effectively the same for `Deref` and `Receiver`. This RFC does not make things worse (it just adds additional flexibility to the `self` parameter type for `T::m`). However it does mean that the `Receiver` trait cannot be added to smart pointer types which were not designed with these concerns in mind.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As this feature has been cooking since 2017, many alternative implementations have been discussed.

## Deref-based
[deref-based]: #deref-based

Unstable Rust contains an implementation of arbitrary self types based around the `Deref<Target=T>` trait. Naturally, that trait provides a means to create a `&T`.

However, if it's OK to create a reference `&T`, you _probably don't need this feature_. You can probably simply use `&self` as your receiver type.

This feature is fundamentally aimed at smart pointer types `P<T>` where it's not safe to create a reference `&T`. As discussed in the rationale, that's most commonly because of semantic differences to pointers in other languages, but it might be because references have special meaning or behavior in some pure Rust domain. Either way, it's not OK to create a Rust reference `&T` or `&mut T`, yet we may want to allow methods to be called on some reference-like thing.

For this reason, implementing `Deref::deref` is problematic for nearly everyone who wants to use arbitrary self types.

If you're implementing a smart pointer `P<T>` yet you can't allow a reference `&T` to exist, any option for implementing `Deref::deref` has drawbacks:

* Specify `Deref::Target=T` and panic in `Deref::deref`. Not good.
* Specify `Deref::Target=*const T`. This works with the current arbitrary self types feature, but is only possible if your smart pointer type contains a `*const T` which you can reference - this isn't the case for (for instance) weak pointers or types containing `NonNull`.

## No blanket implementation for `Deref`
[no-blanket-implementation]: #no-blanket-implementation

The other major approach previously discussed is to have a `Receiver` trait, as proposed in this RFC, but without a blanket implementation for `T: Deref`. Possible advantages:

* It seems more in keeping with Rust norms where types can be in full control of the traits they implement
* It would allow a type to decide that it wants to be dereferencable, without allowing method calls on the inner type
* It would allow a type to specify a different `Target` for `Deref` vs `Receiver`.

However, this increased flexibility comes at a cost: user confusion. In particular, a different specification of `Target` for these two traits would sometimes lead the compiler to explore radically different chains of types when determining the candidates for dereferencing and method calls.

It's easy to imagine cases where this would be confusing for users. Imagine a type `A` dereferences to `B` yet allows method calls only on `C` (or perhaps more confusingly, allows method calls only on `Pin<&mut B>`).

It's hard to imagine a use-case for this which wouldn't be confusing for users of the type. Unless such a use-case presents itself, the authors of theis RFC believe we should intentionally constrain flexibility in this way in order to provide less surprising behavior for consumers of a type. In other words, implementing `Receiver` for `T: Deref` seems a powerful move to reduce user confusion.

Thanks to the blanket implementation, this RFC is just a small delta from the existing unstable "arbitrary self types" mechanism, which has proven to be useful. Splitting `Receiver` behavior from `Deref` would be a riskier evolution to the Rust language.

(A further suggestion had been to provide separate `Receiver` and `Deref` traits yet have the method resolution logic explore both. This seems to offer no advantages over the blanket implementation, and gives a worst-case O(n*m) number of method candidates).

## Generic parameter

Change the trait definition to have a generic parameter instead of an associated type. There might be permutations here which could allow a single smart pointer type to dispatch method calls to multiple possible receivers - but this would add complexity, no known use case exists, and it might cause worst-case O(n^2) performance on method lookup.

## Do not enable for pointers

It would be possible to respect the `Receiver` trait without allowing dispatch onto raw pointers - they are essentially independent changes to the candidate deduction algorithm.

We don't want to encourage the use of raw pointers, and would prefer rather that raw pointers are wrapped in a custom smart pointer that encodes and documents the invariants. So, there's an argument not to stabilize the raw pointer support.

However, the current unstable `arbitrary_self_types` feature provides support for raw pointer receivers, and with years of experience no major concerns have been spotted. We would prefer not to deviate from the existing proposal more than necessary. Moreover, we are led to believe that raw pointer receivers are quite important for the future of safe Rust, because stacked borrows makes it illegal to materialize references in many positions, and there are a lot of operations (like going from a raw pointer to a raw pointer to a field) where users don't need to or want to do that. We think the utility of including raw pointer receivers outweighs the risks of tempting people to over-use raw pointers.

## Provide compiler support for dereferencing pointers

This RFC proposes to implement `Receiver` for `*mut T` and `*const T` within the library. This is slightly different from the unstable arbitrary self types support, which instead hard-codes pointer support into the candidate deduction algorithm in the compiler.

We prefer the option of specifying behavior in the library using the normal trait, though it's a compatibility break for users of Rust who don't adopt the `core` crate (including compiler tests).

## Implement for `Weak` and `NonNull`

`Weak<T>` and `NonNull<T>` were not supported by the prior unstable arbitrary self types support, but they share the property that it may be desirable to implement method calls to `T` using them as self types. Unfortunately they also share the property that these types have many Rust methods using `self`, `&self` or `&mut self`. If we added to the set of Rust methods in future, we'd [shadow any such method calls](#method-shadowing). We can't implement `Receiver` for these types unless we come up with a policy that all subsequent additions to these types would instead be associated functions. That would make the future APIs for these types a confusing mismash of methods and associated functions, and the extra user complexity doesn't seem merited.

## Not do it

As always there is the option to not do this. But this feature already kind of half-exists (we are talking about `Box`, `Pin` etc.) and it makes a lot of sense to also take the last step and therefore enable non-libstd types to be used as self types.

There is the option of using traits to fill a similar role, e.g.

```rust
trait ForeignLanguageRef {
    type Pointee;
    fn read(&self) -> *const Self::Pointee;
    fn write(&mut self, value: *const Self::Pointee);
}

// --------------------------------------------------------

struct ConcreteForeignLanguageRef<T>(T);

impl<T> ForeignLanguageRef for ConcreteForeignLanguageRef<T> {
    type Pointee = T;

    fn read(&self) -> *const Self::Pointee {
        todo!()
    }

    fn write(&mut self, _value: *const Self::Pointee) {
        todo!()
    }
}

// --------------------------------------------------------

struct SomeForeignLanguageType;

impl ConcreteForeignLanguageRef<SomeForeignLanguageType> {
    fn m(&self) {
        todo!()
    }
}

trait Tr {
    type RustType;

    fn tm(self)
    where
        Self: ForeignLanguageRef<Pointee = Self::RustType>;
}

impl Tr for ConcreteForeignLanguageRef<SomeForeignLanguageType> {
    type RustType = SomeForeignLanguageType;
    fn tm(self) {}
}

fn main() {
    let a = ConcreteForeignLanguageRef(SomeForeignLanguageType);
    a.m();
    a.tm();
}
```

This successfully allows method calls to `m()` and even `tm()` without a reference to a `SomeForeignLanguageType` ever existing. However, due to the orphan rule, this forces every crate to have its own equivalent of `ConcreteForeignLanguageRef`. This workaround has been used by some interop tools, but use across multiple crates requires many generic parameters (`impl ForeignLanguageRef<Pointee=SomeForeignLanguageType>`).

## Always use `unsafe` when interacting with other languages

One main motivation here is cross-language interoperability. As noted in the rationale, C++ references can't be _safely_ represented by Rust references. Many would say that all C++ interop is intrinsically unsafe and that `unsafe` blocks are required. Maybe true: but that just moves the problem - an `unsafe` block requires a human to assert preconditions are met, e.g. that there are no other C++ pointers to the same data. But those preconditions are almost never true, because other languages don't have those rules. This means that a C++ reference can never be a Rust reference, because neither human nor computer can promise things that aren't true.

Only in the very simplest interop scenarios can we claim that a human could audit all the C++ code to eliminate the risk of other pointers existing. In complex projects, that's not possible.

However, a C++ reference _can_ be passed through Rust safely as an opaque token such that method calls can be performed on it. Those method calls actually happen back in the C++ domain where aliasing and concurrent modification are permitted.

For instance,

```rust
struct ForeignLanguageRef<T>;

fn main() {
    let some_foreign_language_reference: ForeignLanguageRef<_> = CallSomeForeignLanguageFunctionToGetAReference();
    // There may be other foreign language references to the referent, with concurrent
    // modification, so some_foreign_language_reference can't be a &T
    // But we still want to be able to do this
    some_foreign_language_reference.SomeForeignLanguageMethod(); // executes in the foreign language. Data is not
        // dereferenced at all in Rust.
}
```

# Prior art
[prior-art]: #prior-art

A previous PR based on the `Deref` alternative has been proposed before https://github.com/rust-lang/rfcs/pull/2362 and was postponed with the expectation that the lang team would [get back to `arbitrary_self_types` eventually](https://github.com/rust-lang/rfcs/pull/2362#issuecomment-527306157).

# Feature gates

This RFC is in an unusual position regarding feature gates. There are two existing gates:

- `arbitrary_self_types` enables, roughly, the _semantics_ we're proposing, albeit [in a different way](#deref-based). It has been used by various projects.
- `receiver_trait` enables the specific trait we propose to use, albeit without the `Target` associated type. It has only been used within the Rust standard library, as far as we know.

Although we presumably have no obligation to maintain compatibility for users of the unstable `arbitrary_self_types` feature, we should consider the least disruptive way to stabilize this feature.

Options are:

* Use the `arbitrary_self_types` feature gate until we stabilize this. Remove the `receiver_trait` feature gate immediately. Later, stabilize this and remove `arbitrary_self_types` too.
* Use the `receiver_trait` feature gate until we stabilize this. Remove the `arbitrary_self_types` feature gate immediately. Later, stabilize this and remove `receiver_trait` too.
* Invent a new feature gate.

This RFC proposes the first course of action, since `arbitrary_self_types` is used externally and we think all currently use-cases should continue to work.

# Summary

This RFC is an example of replacing special casing aka. compiler magic with clear and transparent definitions. We believe this is a good thing and should be done whenever possible.
