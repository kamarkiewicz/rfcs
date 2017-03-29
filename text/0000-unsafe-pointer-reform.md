- Feature Name: Unsafe Pointer ~~Reform~~ Methods
- Start Date: 2015-08-01
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)


# Summary
[summary]: #summary

Copy most of the static `ptr::` functions to methods on unsafe pointers themselves.
Also add a few conveniences for `ptr.offset` with unsigned integers.

```rust
// So this:
ptr::read(self.ptr.offset(idx as isize))

// Becomes this:
self.ptr.add(idx).read()
```

More conveniences should probably be added to unsafe pointers, but this proposal is basically the "minimally controversial" conveniences.




# Motivation
[motivation]: #motivation

* `ptr::foo(ptr)` is unnecessarily annoying (and requires imports)
* `ptr.offset(idx as isize)` is unnecessarily annoying

Swift lets you do this:

```swift
let val = ptr.advanced(by: idx).move()
```

And we want to be cool like Swift, right?




# Detailed design
[design]: #detailed-design


## Methodization

Move the following static functions from `ptr` to methods on 
`*const T` and `*mut T`:

```rust
ptr.copy(dst: *mut T, count: usize)
ptr.copy_nonoverlapping(dst: *mut T, count: usize)
ptr.read() -> T
ptr.read_volatile() -> T
ptr.read_unaligned() -> T
```

And these only on `*mut T`:

```rust
ptr.drop_in_place()
ptr.write(val: T)
ptr.write_bytes(val: u8, count: usize)
ptr.write_volatile(val: T)
ptr.write_unaligned()
ptr.replace(val: T) -> T
```

`ptr.swap` has been excluded from this proposal because it's a symmetric operation, and is consequently a bit weird to methodize.

The static functions should remain even though they should be considered unidiomatic because they can be more convenient in cases where you have a safe pointer. Specifically, they act as a coercion site so `ptr::read(&my_val)` works, and is nicer than `(&my_val as *const _).read()`.



## Unsigned Offset

Your rulers have lied to you: llvm's `GetElementPointer [inbounds]` doesn't care about signs or wrapping. It's fine to feed unsigned integers into it, and have them "overflow" as long as the final two's complement result is inbounds.

As such, let's add some conveniences: 

```rust
ptr.add(offset: usize) -> Self
ptr.sub(offset: usize) -> Self
ptr.wrapping_add(offset: usize) -> Self
ptr.wrapping_sub(offset: usize) -> Self
```

I expect `ptr.add` to replace ~95% of all uses of `ptr.offset`.





# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Docs should be updated to use the new methods over the old ones, pretty much
unconditionally. Otherwise I don't think there's anything to do there.

All the docs for these methods can be basically copy-pasted from the existing
functions they're wrapping, with minor tweaks.




# Drawbacks
[drawbacks]: #drawbacks

Bloat's the stdlib and introduces a schism between old and new style. Nothing worth worrying about.





# Alternatives
[alternatives]: #alternatives


## Overload operators for more ergonomic offsets

Rust doesn't support "unsafe operators", and `offset` is an unsafe function because of the semantics of GetElementPointer. Beyond that, `(ptr + idx).read_volatile()` is a bit wonky.



## Make `offset` generic 

You could make `offset` generic so it accepts `usize` and `isize`. However you would still want the `sub` method, and at that point you might as well have `add` for symmetry. Also `add` is shorter which, as we all know, is better.




# Unresolved questions
[unresolved]: #unresolved-questions

Should `ptr::swap` be made into a method? I am personally ambivalent.
