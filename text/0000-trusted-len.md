- Feature Name: Trusted Len
- Start Date: 2015-05-26
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add

```rust
/// If the response to this function is `Some(n)`, the iterator is claiming
/// that it is *Undefined Behaviour* for it to yield anything other than
/// `n` more elements. This enables unsafe code to trust the len when
/// allocating a buffer to copy elements into. This function is actually 100%
/// safe to call and rely on, but entirely unsafe to *implement*.
///
/// After this iterator yields `None`, it still can't be expected
/// to always yield `None` if `next` is called again. However the `trusted_len`
/// must reflect this by being `None`.
unsafe fn trusted_len(&self) -> Option<usize> {
    None
}
```

to the `Iterator` trait.

# Motivation

Iterators are *the* API for generically manipulating collections of
data.

How do I move data from one collection to another?

```rust
another.extend(one); // Calls ont.into_iter() internally
// or
another.extend(one.iter().cloned()); // preserves `one` by cloning
```

How do I make a collection from some data?

```rust
data.into_iter().collect();
```

In order to be a Serious Systems Programming Language, these operations need
to be optimized to a memcopy in many cases. However the leap from iterator to
memcopy is quite large! rustc (really, LLVM), must rip through a stateful
branchy abstraction to determine that two collections are *really* contiguous
buffers of memory and that we're just unconditionally copying a range from one
into another.

This is partly enabled through the `size_hint` API that iterators expose, which
specifies how many elements the iterator yields. Receivers can then `reserve`
space for exactly the number of elements, better enabling memcopy from kicking
in. However, *they can't then trust the iterator to not overrun the reservation*.

size_hint is a safe function, and unsafe code can't trust safe code to not have
bugs. This means the e.g. Vec::extend must count the elements as they're copied
over and constantly branch on an overrun. It also needs this count to actually
set its new len, even if the iterator "only" underruns.

Whether LLVM successfully identifies that the hint *can* be trusted and that all
this work *can* be reduced to a memcopy has been historically *very* delicate.
Performance of collections code should be more reliable than this.

# Detailed design

Therefore this RFC proposes an `unsafe trusted_len` method be added to the
Iterator trait, on-which iterator consuming code can branch for a fast or slow
path:

```rust
/// If the response to this function is `Some(n)`, the iterator is claiming
/// that it is *Undefined Behaviour* for it to yield anything other than
/// `n` more elements. This enables unsafe code to trust the len when
/// allocating a buffer to copy elements into. This function is actually 100%
/// safe to call and rely on, but entirely unsafe to *implement*.
///
/// After this iterator yields `None`, it still can't be expected
/// to always yield `None` if `next` is called again. However the `trusted_len`
/// must reflect this by being `None`.
unsafe fn trusted_len(&self) -> Option<usize> {
    None
}
```


For the *vast majority* of iterator consumers, whether this response is
`Some` or `None` will be statically determined. In this case trivial
constant folding should be able to eliminate the branch altogether. In the
worst-case this introduces a branch in an already-expensive method which should
be easy to predict should it be in any kind of hot loop.

Using trusted_len is also *entirely optional* for any consumer, so if the
branch is deemed to be useless, it can simply be omitted. Same for
even using size_hint today. Similarly implementors who don't think it would
be useful can simply ignore the functionality and let the default None handle
it.

Note that the function is marked as `unsafe` simply so *implementation* requires
writing the word `unsafe`. Calling this function and relying on its result is
in fact completely safe. In the authors opinion having to write `unsafe` to use
it is minimally onerous, as it is specifically intended to be used in `unsafe`
contexts.

# Drawbacks

* More complexity when implementing iterators (more things to forward and think about).
* Introduces more `unsafe` code into our public APIs.

# Alternatives

The primary counter-candidate for this design is a new `unsafe trait TrustedIterator`
which provides `fn trusted_len() -> usize`. It has the nice benefit of the `unsafe`
being in the right place; it is unsafe to implement but not to use. It is also
more-definitely going to be statically dispatched on. However there are several
downsides with such a design:

* Today's Rust completely lacks specialization, making it impossible for code to
*actually* take advantage of TrustedIterator.
* Some types can genuinely only provide TrustedIterator in a runtime-conditional
manner. For instance `Chain` has a trusted_len iff its two components do *and
their combined sizes don't overflow usize*. This is why Chain can't ever implement
ExactSizeIterator.
* TrustedLen is lost when working with Trait Objects, while trusted_len is
on *all* Iterators.
* Similarly TrustedLen would have to be "passed through" any generic functions
(depending on how specialization is ever implemented).

# Unresolved questions

None yet.
