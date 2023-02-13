- Feature Name: `array::try_from_iter`
- Start Date: 2023-02-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

This feature aims to introduce a standard way of fallibly creating a `[T; N]` from an
`IntoIterator<Item = T>`. It functions by passing an `IntoIterator<Item = T>` which will result in a
`[T; N]`, if and only if the `IntoIterator` holds `>= N` items. Otherwise, it returns an
`array::IntoIter<T>` with the original `IntoIterator`'s items. The proposed method signature follows:

```rust
pub fn try_from_iter<I, const N: usize>(iter: I) -> Result<[I::Item; N], IntoIter<I::Item, N>>
where
    I: IntoIterator,
```

# Motivation

[motivation]: #motivation

In cases in which we want to construct a fixed-size _array_ from a collection or iterator we currently
do not have a standard and safe way of doing so. While it is possible to do this without additional
allocation, this requires some use of `unsafe` and thus leaves the implementer with more room for error
and potential `UB`.

There has been much work done on fallible `Iterator` methods such as `try_collect` and `try_fold`, and
more work that is being undertaken to stabilize things like `try_process`, these do not strictly apply to
this. The purpose of this API is creating a fixed-size array from an iterator where the failure case does
not come from the `Iterator`'s `Item`s, but solely from the `Iterator` not holding enough `Item`s.

Therefore, it seems like giving _array_ a dedicated associated method that narrows the scope of failure
to the relationship between the size of the `Iterator` and the size of the _array_ would be the pragmatic
thing to do.

Additionally, the relatively recent addition of [`iter:next_chunk`](https://github.com/rust-lang/rust/issues/98326)
means this implements basically for free.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

`array::try_from_iter` provides a safe API for creating fixed-size _arrays_ `[T; N]` from `IntoIterator`s.

For a somewhat contrived example, say we want to create a `[String; 32]` from a `HashSet<String>`.

```rust
let set: HashSet<String> = HashSet::new();

/* add some items to the set... */

let Ok(my_array) = std::array::try_from_iter::<_, 32>(set) else {
    /* handle error... */
};

```

Oftentimes it can be more efficient to deal with fixed-size arrays, as they do away with the inherent
indirection that come with many of the other collection types. This means arrays can be great when
lookup speeds are of high importance. The problem is that Rust does not make it particularly easy to
dynamically allocate fixed-size arrays at runtime without the use of `unsafe` code. This feature allows
developers to turn their collections into fixed-size arrays of specified size in a safe and ergonomic
manner.

Naturally it can be the case that a collection or `IntoIterator` does not hold enough elements to fill the
entire length of the _array_. If that happens `try_from_iter` returns an `array::IntoIter` that that holds
the elements of the original `IntoIterator`.

As this change is additive its adaptation should be quite straight forward. There may of course an
argument to be made for choosing a different name with better discoverability.

The addition of any methods that can replace existing `unsafe` code in crates should be an improvement, at
least as it pertains to safety.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The implementation of this feature should be quite straight forward as it is essentially a wrapper around
`iter::next_chunk`:

```rust
#[inline]
pub fn try_from_iter<I, const N: usize>(iter: I) -> Result<[I::Item; N], IntoIter<I::Item, N>>
where
    I: IntoIterator,
{
    iter.into_iter().next_chunk()
}
```

It turns the passed type/collection into `iter`, an `Iterator<Item = T>` and then calls `next_chunk`.

# Drawbacks

[drawbacks]: #drawbacks

There are actually many good arguments against adopting this. The following is an abbreviated list of
some:

1. **The question of the name.** What should this associated function be called? `try_from_iter` is just
   one of multiple options. Alternatively `try_from_iterator` could also be used. The problem with
   a name like `try_from_iterator` is that it may be associated with `try_from_fn` and `try_collect` which
   base their `try` fallibility on the `Try` yielded from function or `Iterator`. In our case this would
   be an inappropriate association, as `try_from_iter` fails if there are not enough items yielded. Still,
   there may be alternative names that fit better.
2. **Should this really be an associated function?**
3. **Do we need this if we could use `iter::next_chunk`?** Currently we can already achieve the
   functionality we get from `try_from_iter` with `iter::next_chunk` and the new function is really just
   a wrapper around what already exists. Having a dedicated method may improve discoverability, but as it
   stands this is a very strong argument against this.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### Rationale

Having a dedicated associated function on `array` would improve discoverability and give a standard way
of turning any collection implementing `IntoIterator` into an array of fixed size.

### Alternatives

Since there still is not complete consensus on around what should be returned from any `TryFrom<Iterator>`
there are many good alternatives and arguments for different APIs and implementations.

Some of them are listed here:

1. Simply use `iter::next_chunk` as suggested [here](https://github.com/rust-lang/rust/pull/107634#discussion_r1103707872).
2. Using something like an `ArrayVec<T, N>` instead.
3. Implement `TryFrom<IntoIterator<Item = T>>` for `[T; N]`. This is problematic because of
   [Invalid collision with TryFrom implementation? #50133](https://github.com/rust-lang/rust/issues/50133).
4. New `TryFromIterator` trait. Would likely also require `TryIntoIterator`.
5. Naming alternatives:  
   2.1 `try_fill_from`  
   2.2 `try_collect_from` (suggests it should be used with an `Iterator` directly)  
   2.3 `from_iter` (does not suggest fallibility)

Adding anything to the `core` and `std` API has lasting implications, and should this be scrutinized from
all angles.

# Prior art

[prior-art]: #prior-art

There exist already a few open issues on the topic of turning iterators into fixed-sized array.

- [Provide a means of turning iterators into fixed-size arrays #81615](https://github.com/rust-lang/rust/issues/81615)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- Naming?
- Should the associated function be on `array` or `Iterator`?
- Is this necessary?
- Are there more pragmatic/ergonomic ways of doing this?
- Could a dedicated `TryFromIterator` trait solve this better?

# Future possibilities

[future-possibilities]: #future-possibilities

In the future, a dedicated trait may be a better fit. This would require more investigation
and a larger change to `core` and `std` and thus is to be seen.
