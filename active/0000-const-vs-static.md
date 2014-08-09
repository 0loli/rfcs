- Start Date: 2014-08-08
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Divide global declarations into two categories:

- **constants** declare *constant values*. These represent a value,
  not a memory address. This is the most common thing one would reach
  for and would replace `static` as we know it today in almost all
  cases.
- **statics** declare *global variables*. These represent a memory
  address.  They would be rarely used: the primary use cases are
  global locks, global atomic counters, and interfacing with legacy C
  libraries.

# Motivation

We have been wrestling with the best way to represent globals for some
times. There are number of interrelated issues:

- *Significant addresses and inlining:* For optimization purposes, it
  is useful to be able to inline constant values directly into the
  program. It is even more useful if those constant values do not have
  a known address, because that means the compiler is free to replicate
  them as it wishes. Moreover, if a constant is inlined into downstream
  crates, than they must be recompiled whenever that constant changes.
- *Read-only memory:* Whenever possible, we'd like to place large
  constants into read-only memory. But this means that the data must
  be truly immutable, or else a segfault will result.
- *Global atomic counters and the like:* We'd like to make it possible
  for people to create global locks or atomic counters that can be
  used without resorting to unsafe code.
- *Interfacing with C code:* some C libraries require the use of
  global, mutable data. Other times it's just convenient and threading
  is not a concern.
- *Initializer constants:* there must be a way to have initializer
  constants for things like locks and atomic counters, so that people
  can write `static MY_COUNTER: AtomicUint = INIT_ZERO` or some
  such. It should not be possible to modify these initializer
  constants.
  
The current design is that we have only one keyword, `static`, which
declares a global variable. By default, global variables do not have a
significant address and can be inlined into the program. You can make
a global variable have a *significant* address by marking it
`#[inline(never)]`. Furthermore, you can declare a mutable global
using `static mut`: all accesses to `static mut` variables are
considered unsafe. Because we wish to allow `static` values to be
placed in read-only memory, they are forbidden from having a type that
includes interior mutable data (that is, an appearance of `UnsafeCell`
type).

Some concrete problems with this design are:

- There is no way to have a safe global counter or lock. Those must be
  placed in `static mut` variables, which means that access to them is
  illegal. To resolve this, there is an alternative proposal which
  makes access to `static mut` be considered safe if the type of the
  static mut meets the `Sync` trait.
- The signifiance (no pun intended) of the `#[inline(never)]` annotation
  is not intuitive.
- There is no way to have a generic type constant.

Other less practical and more aesthetic concerns are:

- Although `static` and `let` look and feel analogous, the two behave
  quite differently.  Generally speaking, `static` declarations do not
  declare variables but rather values, which can be inlined and which
  do not have a fixed address. You cannot have interior mutability in
  a `static` variable, but you can in a `let`. So that `static`
  variables can appear in patterns, it is illegal to shadow a `static`
  variable -- but `let` variables cannot appear in patterns. Etc.
- There are other constructs in the language, such as nullary enum
  variants and nullary structs, which look like global data but in
  fact act quite differently. They are actual values which do not have
  a address. They are categorized as rvalues and so forth.

# Detailed design

## Constants

Reintroduce a `const` declaration which declares a *constant*:

    const name: type = value;

Constants may be declared in any scope. They cannot be shadowed.
Constants are considered rvalues. Therefore, taking the address of a
constant actually creates a spot on the local stack -- they by
definition have no significant address. Constants are intended to
behave exactly like a nullary enum variant.

### Possible extension: Generic constants

As a possible extension, it is perfectly reasonable for constants to
have generic parameters. For example, the following constant is legal:

    struct WrappedOption<T> { value: Option<T> }
    const NONE<T> = WrappedOption { value: None }
    
Note that this makes no sense for a `static` variable, which represents
a memory location and hence must have a concrete type.
    
### Possible extension: constant functions

It is possible to imagine constant functions as well. This could help
to address the problem of encapsulating initialization. To avoid the
need to specify what kinds of code can execute in a constant function,
we can limit them syntactically to a single constant expression that
can be expanded at compilation time (no recursion).

    struct LockedData<T:Send> { lock: Lock, value: T }
    
    const LOCKED<T:Send>(t: T) -> LockedData<T> {
        LockedData { lock: INIT_LOCK, value: t }
    }

This would allow us to make the `value` field on `UnsafeCell` private,
among other things.

## Static variables

Repurpose the `static` declaration to declare static variables
only. Static variables always have a single address. `static`
variables can optionally be declared as `mut`. The lifetime of a
`static` variable is `'static`. It is not legal to move from a static.
Accesses to a static variable generate actual reads and writes: the
value is not inlined (but see "Unresolved Questions" below).

Non-`mut` statics must have a type that meets the `Sync` bound. All
access to the static is considered safe (that is, reading the variable
and taking its address).  If the type of the static does not contain
an `UnsafeCell` in its interior, the compiler may place it in
read-only memory, but otherwise it must be placed in mutable memory.

`mut` statics may have any type. All access is considered unsafe.
They may not be placed in read-only memory and their values may

# Drawbacks

This RFC introduces two keywords for global data. Global data is kind
of an edge feature so this feels like overkill. (On the other hand,
the only keyword that most Rust programmers should need to know is
`const` -- I imagine `static` variables will be used quite rarely.)

# Alternatives

The other design under consideration is to keep the current split but
make access to `static mut` be considered safe if the type of the
static mut is `Sync`. For the details of this discussion, please see
[RFC 177](https://github.com/rust-lang/rfcs/pull/177).

One serious concern is with regard to timing. Adding more things to
the Rust 1.0 schedule is inadvisable. Therefore, it would be possible
to take a hybrid approach: keep the current `static` rules, or perhaps
the variation where access to `static mut` is safe, for the time
being, and create `const` declarations after Rust 1.0 is released.

# Unresolved questions

- Should the compiler be allowed to inline the values of `static`
  variables which are deeply immutable (and thus force recompilation)?

- Should we permit `static` variables whose type is not `Sync`, but
  simply make access to them unsafe?

- How hard are the envisioned extensions to implement? If easy, they
  would be nice to have. If hard, they can wait.
