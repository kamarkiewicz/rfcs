- Feature Name: solidify-default-rust
- Start Date: 2015-06-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Specify `repr(Rust)` properties that are useless to any kind of fuzzer or optimizer to
give programmers more control and understanding.



# Motivation

[RFC #0079](https://github.com/rust-lang/rfcs/blob/master/text/0079-undefined-struct-layout.md)
made the default struct layout completely undefined. Whether or not you agree with
this, it's certainly limiting if you want to use the default repr and know what's
going on.

In particular, there are several situations where there's nothing rust could
plausibly do to the layout that wouldn't be completely arbitrary or flat-out bad,
and as such we might as well formally state that this is in fact the case.

In addition the unspecified layout thing has caused some uncertainty as to
whether things that should trivially be true are.



# Detailed design

For the rest of the document we will refer only to structs, but these properties
also apply to tuples, which will be considered to desugar to a struct in the
following manner:

```rust
type (T1, T2, T3) = struct Tuple3<T1, T2, T3: ?Sized>(T1, T2, T3);
```



## Destructors

While drop flags exist on structs, any type with a destructor has an extra
unspecified positive-sized field. This means that a type may not have as
nice layout properties as expected based on the rules that follow.



## Alignment and Padding

If a structs fields can only be Sized, Rust will use all of the same alignment
and padding rules as `repr(C)`, the only difference between `repr(C)` and
`repr(Rust)` is the *ordering* of fields. Further, zero-sized types cannot
influence the alignment or padding of a struct.

However if a struct contains a `?Sized` field as well as another non-zero-sized
field, its padding astrategy is unspecified. This is
because it may be desirable to right-align all the Sized fields to avoid runtime
computation of the Sized field's alignment.

A `repr(C)` struct cannot have a `?Sized` field. Because this is technically a
breaking change, this will just warn that `repr(C)` was not acknowledged. This of
course implies tuples are not `repr(C)`, and their padding is usually unspecified.



## Zero-Sized Types

A struct that only contains zero-sized types is also zero-sized. It has no alignment
requirements. All zero-sized types can be freely transmuted amongst each-other.
There is an allocated instance of every zero-sized type at address `heap::EMPTY`.



## Unary Types

All structs that have a single positive-sized field of type T (`?Sized` or not)
and no destructor can be safely transmuted between each-other, as well as T
itself. That is, all of the following types have an identical run-time layout:

* `T`
* `NewType(T)`
* `GenericNewType<T: ?Sized>(T)`
* `(T,)`
* `[T; 1]`
* `NewTypeWithZeroSizedJunk((), (), T, ())`



## Pointer Equivalences

All builtin-pointers have the same layout (C's) when they point to a Sized type,
even if they point to a *different* Sized type. For unsized types layout is
unspeicified, but all the pointer types should definitely have the *same*
layout for the the *same* unsized type (casting is a noop).



## Final Fields

If the last field of a struct is `?Sized`, it must always be the last
field in-memory. Such a struct with exactly *two* positive-sized fields
therefore has defined ordering but undefined padding.



# Drawbacks

Nailing ourselves to the C repr in some places may *hypothetically* prevent some
interesting tricks. However this RFC is purposefully very conservative, and as
such the odds of preventing a legitimate optimization is unlikely.

The value of being able to rely on some *basic* layout guarantees is worth more
than being able to do something bizarre to a type with 1 field.



# Alternatives

The space of alternative layout strategies is potentially infinite.



# Unresolved questions



## Box layout

Note that one cannot *in general* guarantee that a Box has the same layout as
a builtin-pointer, as a Box using a local allocator may want to be fat to store
a pointer back to the allocator. The methods and functions in `std::boxed`
must instead be used. However it may be desirable to specify the default
global allocator will cause a Box to be the same layout.

Note that `Rc<T>` and `Arc<T>` are *definitely* not the same a `*mut T`.



## repr(fixed)

`repr(C)` doesn't make much sense for generic, ?Sized, or zero-sized structs, and
non-C-like enums. However it is still desirable in some cases to specify layout.
It may be worth including a `repr(fixed)` to specify this and lint against `repr(C)`
more aggressively in general.

