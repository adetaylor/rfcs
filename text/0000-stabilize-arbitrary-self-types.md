- Feature Name: Arbitrary Self Types 2.0
- Start Date: 2023-05-04
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow types that implement the new `trait Receiver<Target=Self>` to be the receiver of a method.

# Motivation
[motivation]: #motivation

Today, methods can only be received by value, by reference, or by one of a few blessed smart pointer types from `core`, `alloc` and `std` (`Arc<Self>`, `Box<Self>`, `Pin<Self>` and `Rc<Self>`).

It's been assumed that this will eventually be generalized to support any smart pointer, such as an `CustomPtr<Self>`. Since late 2017, it has been available on nightly under the `arbitrary_self_types` feature for types that implement `Deref<Target=Self>` and for raw pointers.

One relevant use-case is cross-language interop (JavaScript, Python, C++), where other languages' references can’t guarantee the aliasing and exclusivity semantics required of a Rust reference. For example, the C++ `this` pointer can't be practically or safely represented as a Rust reference because C++ may retain other pointers to the data and it might mutate at any time. Yet, calling C++ methods intrinsically requires a `this` reference.

Another case is when the existence of a reference is, itself, semantically important — for example, reference counting, or if relayout of a UI should occur each time a mutable reference ceases to exist. In these cases it's not OK to allow a regular Rust reference to exist, and yet sometimes we still want to be able to call methods on a reference-like thing.

In theory, users can define their own smart pointers. In practice, they're second-class citizens compared to the smart pointers in Rust's standard library. User-defined smart pointers to `T` can accept method calls only if the receiver (`self`) type is `&T` or `&mut T`, which isn't acceptable when we can't safely create native Rust references to a `T`.

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
- smart pointers: `Rc<Self>`, `Arc<Self>`, `Box<Self>`, and `Pin<Ptr<Self>>`, because these types all implement `Deref` and there's a blanket implementation of `Receiver` for `Deref`.
- references: `&Self` and `&mut Self`

Shorthand exists for references, so that `self` with no ascription is of type `Self`, `&self` is of type `&Self` and `&mut self` is of type `&mut Self`.

All of the following self types are valid:

```rust
impl Foo {
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

## `core` libs changes

The `Receiver` trait is made public (removing its `#[doc(hidden)])` attribute), exposing it under `core::ops`. It adds a `Target` associated type.

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

and for both mutable and immutable references.

The existing Rust [reference section for method calls describes the algorithm assuming that the prior version of `arbitrary_self_types` was stabilized](https://doc.rust-lang.org/reference/expressions/method-call-expr.html), so isn't 100% accurate for the current state of stable Rust.

To summarize the algorithms in all three states:

## Without `arbitrary_self_types` or this new `Receiver` trait

This is the current status in stable Rust.

A possible list of candidate types is created by:

1. Deref the receiver expression's type repeatedly, until we encounter any type that doesn't implement the hidden `Receiver` trait (`Self`, `&Self`, `&mut Self`, `Rc<Self>`, `Arc<Self>`, `Box<Self>`, and `Pin<Ptr<Self>>`)
2. Finally attempt an unsized coercion
3. For each type, consider `T`, `&T` and `&mut T`

## With the previous `arbitrary_self_types`
[previous-self-types]: #previous-self-types

This is the status as described in the existing reference.

A possible list of candidate types is created by:

1. Deref the receiver expression's type repeatedly (allowing dereferencing steps via raw pointers too)
2. Finally attempt an unsized coercion
3. For each type, consider `T`, `&T` and `&mut T`

## With this new `Receiver` trait

A possible list of candidate types is created by:

1. Follow the chain of `Receiver` targets (that is, if `T` implements `Receiver`, add `<T as Receiver>::Target` to the chain)
2. Finally attempt an unsized coercion
3. For each type, consider `T`, `&T` and `&mut T`

Because there is a blanket implementation of `Receiver` for `Deref`, this new algorithm is very similar to the previous `arbitrary_self_types`. The differences are (a) we don't allow dereferencing steps via raw pointers, (b) `Receiver` can be implemented by types that don't implement `Deref`.

## Object safety

Receivers are object safe if they implement the (unstable) `core::ops::DispatchFromDyn` trait.

As not all receivers might want to permit object safety or are unable to support it. Therefore object safety should remain being encoded in a different trait than the here proposed `Receiver` trait, likely `DispatchFromDyn`.

Since `DispatchFromDyn` is unstable at the moment, object-safe receivers might be delayed until `DispatchFromDyn` is stabilized. `Receiver` and `DispatchFromDyn` can be stabilized together, but `Receiver` is not blocked in this, since non-object-safe receivers already cover a big chunk of the use-cases.

## Lifetime Elision

As discussed in the [motivation](#motivation), this new facility is most likely
to be used in cases where a standard reference can't normally be used. But
the self type might wrap a standard Rust reference, and thus might be
parameterized by a lifetime.

This works just fine:

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

The existing branches in the compiler for "arbitrary self types" already emit
excellent diagnostics. We will largely re-use them, with the following improvements:

- In the case where a self type is invalid where it doesn't implement `Receiver`,
  the existing excellent error message will be updated
- An easy mistake is to implement `Receiver` for `P<T>` without specifying that `T: ?Sized`. `P<Self>` then only works as a `self` parameter in traits `where Self: Sized`, an unusual stipulation. It's not obvious that `Sized`ness is the problem here, so we will identify this case specifically and produce an error giving that hint.
- There are certain types which feel like they "should" implement `Receiver` but do not: `*const T`, `*mut T`, `Weak` or `NotNull`. If these are encountered as a self type, we should produce a specific diagnostic explaining that they do not implement `Receiver` and suggesting that they could be wrapped in a newtype wrapper if method calls are important. This will require `Weak` and `NonNull` be marked as lang items so that the compiler is aware of the special nature of these types. (The authors of this RFC feel that these extra lang-items _are_ worthwhile to produce these improved diagnostics - if the reader disagrees, please let us know.)

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Deref coercions can already be confusing and unexpected. `Deref` becomes more powerful and significant if it allows method calls.
- If a smart pointer type `P` implements `Deref<Target=T>`, it may well be used to allow method calls on `T` using `fn m(self: P<T>)`
  and similar. This effectively constrains the subsequent implementation of `P`, because any new methods added to `P` are a
  compatibility break - more details in the Method Shadowing section, below.
- Custom smart pointers are a niche use case (but they're very important for cross-language interoperability.)

## Method shadowing
[method-shadowing]: #method-shadowing

Currently for a smart pointer `P`, a method call `p.m()` can only possibly call
a method on that smart pointer type itself - `P::m`.

With arbitrary self types, and assuming `P: Receiver<Target=T>`, it's possible that
the method call could be `P::m` or `T::m`.

It's assumed that smart pointers can't usually predict the possible types to
which they refer (`T`) and so the creator of `P` cannot know in advance what
`T` methods may exist.

This effectively means that adding extra methods to `P` is a possible
compatibility break, because `P` might shadow methods already in `T`.

Fortunately, the Rust standard library smart pointer types were already designed
with this in mind - `Box`, `Pin`, `Rc` and `Arc` already heavily use associated
functions rather than methods. The same approach should be taken by custom smart
pointers. But this does mean that it's difficult to adopt "arbitrary self types"
for existing smart pointer types unless they were designed with this in mind.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Deref-based

Unstable Rust contains an implementation of arbitrary self types based around the
`Deref` trait.

However, if it's OK to create a reference `&T`, you probably don't need this feature.
You can probably simply use `&self` as your receiver type.

This feature is fundamentally aimed at smart pointer types `P<T>` where it's not safe
to create a reference `&T`. As discussed in the rationale, that's most commonly
because of semantic differences to pointers in other languages, but it might be
because references have special meaning or behavior in some pure Rust domain.
Either way, it's not OK to create a Rust reference `&T` or `&mut T`, yet we
may want to allow methods to be called on some reference-like thing.

For this reason, implementing `Deref::deref` is problematic for nearly everyone
who wants to use arbitrary self types.

If you're implementing a smart pointer `P<T>` yet you can't allow a reference `&T`
to exist, any option for implementing `Deref::deref` has drawbacks:

* Specify `Deref::Target=T` and panic in `Deref::deref`. Not good.
* Specify `Deref::Target=*const T`. This works with the current arbitrary self
  types feature, but that's because the current feature allows intermediate
  steps of raw pointers, and we don't think we can stabilize that due to the
  [method shadowing concerns discussed higher up](#method-shadowing). In any case, this is only
  possible if your smart pointer type contains a `*const T` which you can
  reference - this isn't the case for (for instance) weak pointers or types
  containing `NonNull`.

## No blanket implementation for `Deref`

The other major approach previously discussed is to have a `Receiver` trait,
as proposed in this RFC, but without a blanket implementation for `T: Deref`.
An advantage would be the ability for a type to determine independently whether
it should be dereferenceable and/or a method receiver. But no known use-case
exists, and it would be confusing to have distinct chains for dereferencing
and method calls. Implementing `Receiver` for `T: Deref` seems a powerful move
to reduce user confusion.

(A further suggestion had been to provide separate `Receiver` and `Deref` traits
yet have the method resolution logic explore both. This seems to offer no
advantages over the blanket implementation, and gives a worst-case O(n*m) number
of method candidates).

## Generic parameter

Change the trait definition to have a generic parameter instead of an associated type.
There might be permutations here which could allow a single smart pointer type
to dispatch method calls to multiple possible receivers - but this would add
complexity, no known use case exists, and it might cause worst-case O(n^2)
performance on method lookup.

## Enable for pointers too

The current unstable `arbitrary_self_types` feature also allows dispatch
directly onto raw pointers:

```rust
impl Foo {
    fn method(self: *const Self) {
        // ...
    }
}
```

However, we do not propose to stabilize this. For once because of the concerns
mentioned in the [method Shadowing section](#method-shadowing). Secondly
because we do not want to encourage the use of raw pointers, but rather that
raw pointers are wrapped in a custom smart pointer that encodes and documents
the invariants.

## Enable for pointers with additional diagnostics

Elsewhere in Rust, there are already diagnostics saying
> a method with this name may be added to the standard library in the future
and warning about the consequences.

If we chose to stabilize arbitrary self types for raw pointers too, we could
simply warn that _any_ use of a raw pointer as a self type could be subject
to future shadowing by standard library. But, that would essentially make
this feature perpetually warny for raw pointers, so it seems better to just
remove it.

## Enable for pointers behind an unstable flag

The current unstable `arbitrary_self_types` feature _does_ allow method dispatch
on raw pointers. For anyone relying on that, this is a breakage. We could
add that facility behind an alternative unstable flag.

However, as discussed under [method Shadowing section](#method-shadowing) this
would prevent Rust from ever adding more methods to the raw pointer primitive
type. That doesn't feel like it will be OK, and so there is no known path
to stabilizing raw pointer self types. Therefore, we choose to just remove
this facility.

## Allow implementation of `Receiver` for foreign types

An alternative workaround for the removal of this "raw pointer self type"
facility is to allow explicit implementation of `Receiver` for raw pointers.

```rust
impl Receiver for *const MyType {
    type Target = MyType;
}
```

This currently falls foul of the orphan rule. We could add an exception
just as we have for [traits parameterized by local types](https://blog.rust-lang.org/2020/01/30/Rust-1.41.0.html#relaxed-restrictions-when-implementing-traits)
but this would be complex in itself.

## Implement for `Weak` and `NonNull`

`Weak<T>` and `NonNull<T>` were not supported by the prior unstable arbitrary self tpes
support, but they share the property that it may be desirable to implement
method calls to `T` using them as self types. Unfortunately they also share the property that these types
have many Rust methods using `self`, `&self` or `&mut self`. If we added to the set of Rust methods in future,
we'd [shadow any such method calls](#method-shadowing). We can't implement `Receiver` for these types unless
we come up with a policy that all subsequent additions to these types would
instead be associated functions.

## Not do it

As always there is the option to not do this. But this feature already kind of half-exists (I am talking about `Box`, `Pin` etc.) and it makes a lot of sense to also take the last step and therefore enable non-libstd types to be used as self types.

There is the option of using traits to fill a similar role, e.g.

```rust
trait CppPtr {
    type Pointee;
    fn read(&self) -> *const Self::Pointee;
    fn write(&mut self, value: *const Self::Pointee);
}

// --------------------------------------------------------

struct CppPtrType<T>(T);

impl<T> CppPtr for CppPtrType<T> {
    type Pointee = T;

    fn read(&self) -> *const Self::Pointee {
        todo!()
    }

    fn write(&mut self, _value: *const Self::Pointee) {
        todo!()
    }
}

// --------------------------------------------------------

struct SomeCppType;

impl CppPtrType<SomeCppType> {
    fn m(&self) {
        todo!()
    }
}

trait Tr {
    type RustType;

    fn tm(self)
    where
        Self: CppPtr<Pointee = Self::RustType>;
}

impl Tr for CppPtrType<SomeCppType> {
    type RustType = SomeCppType;
    fn tm(self) {}
}

fn main() {
    let a = CppPtrType(SomeCppType);
    a.m();
    a.tm();
}
```

This successfully allows method calls to `m()` and even `tm()` without a reference to a `SomeCppType` ever existing.
However, due to the orphan rule, this forces `SomeCppType` to be in the same crate as `CppPtrType`. This workaround
has been used by some C++ interop tools, but results in complex function signatures in all downstream code
(`impl CppPtr<Pointee=SomeCppType>` all over the place).

## Always use `unsafe` when interacting with other languages

One main motivation here is cross-language interoperability. As noted in the rationale,
C++ references can't be _safely_ represented by Rust references. Some would say that all C++
interop is intrinsically unsafe and that `unsafe` blocks are required. But that doesn't
solve the problem - an `unsafe` block requires a human to assert preconditions are met,
e.g. that there are no other C++ pointers to the same data. But those preconditions are
almost never true, because other languages don't have those rules. This means that a C++ reference
can never be a Rust reference, because neither human nor computer can promise
things that aren't true.

Only in the very simplest interop scenarios can we claim that a human could
audit all the C++ code to eliminate the risk of other pointers exisitng. In
complex projects, that's not possible.

However, a C++ reference _can_ be passed through Rust safely as an opaque token
such that method calls can be performed on it. Those method calls actually happen
back in the C++ domain where aliasing and concurrent modification are "fine".

For instance,

```rust
struct CppRef<T>;

fn main() {
    let some_cpp_reference: CppRef<_> = CallSomeCppFunctionToGetAReference();
    // There may be other C++ references to the referent, with concurrent
    // modification, so some_cpp_reference can't be a &T
    // But we still want to be able to do this
    some_cpp_reference.SomeCppMethod(); // executes in C++. Data is not
        // dereferenced at all in Rust.
}
```

# Prior art
[prior-art]: #prior-art

A previous PR based on the `Deref` alternative has been proposed before https://github.com/rust-lang/rfcs/pull/2362
and was postponed with the expectation that the lang team would [get back to `arbitrary_self_types` eventually](https://github.com/rust-lang/rfcs/pull/2362#issuecomment-527306157).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

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
- This sinister TODO is present in the code:
                // FIXME(arbitrary_self_types): We probably should limit the
                // situations where this can occur by adding additional restrictions
                // to the feature, like the self type can't reference method substs.

# Feature gates

This RFC is in an unusual position regarding feature gates. There are two
existing gates:

- `arbitrary_self_types` enables, roughly, the _semantics_ we're proposing,
  albeit [in a different way](#with-the-previous-arbitrary_self_types) and with
  support for raw pointers as self types. It has been used by various projects.
- `receiver_trait` enables the specific trait we propose to use, albeit
  without the `Target` associated type. It has only been used within the Rust
  standard library, as far as we know.

Although we presumably have no obligation to maintain compatibility for users
of the unstable `arbitrary_self_types` feature, we should consider the least
disruptive way to stabilize this feature.

Options are:

* Use the `receiver_trait` feature gate until we stabilize this. Remove
  the `arbitrary_self_types` feature gate immediately. Later, stabilize this and remove
  `receiver_trait` too.
* Use the `arbitrary_self_types` feature gate until we stabilize this. Remove
  the `receiver_trait` feature gate immediately. Later, stabilize this and remove
  `arbitrary_self_types` too.
* Invent a new feature gate.
* Immediately stabilize this without any feature gate, and remove both
  `arbitrary_self_types` and `receiver_trait`

It seems potentially confusing to alter the semantics of the already-used
`arbitrary_self_types` feature gate, so this RFC proposes the first course
of action.

We propose that this feature be available behind the `receiver_trait` feature
gate for two releases, prior to being fully stabilized. That should allow
enough time for existing users of `arbitrary_self_types` to adapt and report
any concerns.

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC is an example of replacing special casing aka. compiler magic with clear and transparent definitions. We believe this is a good thing and should be done whenever possible.
