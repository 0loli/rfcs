- Feature Name: `drop_types_in_const`
- Start Date: 2016-01-01
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow types with destructors to be used in `const`/`static` items, as long as the destructor is never run during `const` evaluation.

# Motivation
[motivation]: #motivation

Most collection types do not allocate any memory when constructed empty. With the change to make leaking safe, the restriction on `static` items with destructors
is no longer trequired to be a hard error.

Allowing types with destructors to be directly used in `const` functions and stored in `static`s will remove the need to have
runtime-initialisation for global variables.

# Detailed design
[design]: #detailed-design

- Remove the check for `Drop` types in constant expressions.
- Add an error lint ensuring that `Drop` types are not dropped in a constant expression
 - This includes when another field is moved out of a struct/tuple, and unused arguments in constant functions.

# Drawbacks
[drawbacks]: #drawbacks

Destructors do not run on `static` items (by design), so this can lead to unexpected behavior when a side-effecting type is stored in a `static` (e.g. a RAII temporary folder handle). However, this can already happen using the `lazy_static` crate, or with `Option<DropType>` (which bypasses the existing checks).

# Alternatives
[alternatives]: #alternatives

Existing workarounds are based on storing `Option<T>`, and initialising it to `Some` upon first access. These solutions work, but require runtime intialisation and incur a checking overhead on subsequent accesses.

# Unresolved questions
[unresolved]: #unresolved-questions

- TBD
