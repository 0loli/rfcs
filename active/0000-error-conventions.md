- Start Date: (fill me in with today's date, 2014-09-11)
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# Summary

This is a *conventions* RFC for formalizing the basic conventions around error
handling in Rust libraries.

The high-level overview is:

* For *catastrophic errors*, abort the process or fail the task depending on
  whether any recovery is possible.

* For *contract violations*, fail the task. (Recover from programmer errors at a coarse grain.)

* For *obstructions to the operation*, use `Result` (or, less often,
  `Option`). (Recover from obstructions at a fine grain.)

* Prefer liberal function contracts, especially if reporting errors in input
  values may be useful to a function's caller.

This RFC follows up on [two](https://github.com/rust-lang/rfcs/pull/204)
[earlier](https://github.com/rust-lang/rfcs/pull/220) attempts by giving more
leeway in when to fail the task.

# Motivation

Rust provides two basic strategies for dealing with errors:

* *Task failure*, which unwinds to at least the task boundary, and by default
  propagates to other tasks through poisoned channels and mutexes. Task failure
  works well for coarse-grained error handling.

* *The Result type*, which allows functions to signal error conditions through
  the value that they return. Together with a lint and the `try!` macro,
  `Result` works well for fine-grained error handling.

However, while there have been some general trends in the usage of the two
handling mechanisms, we need to have formal guidelines in order to ensure
consistency as we stabilize library APIs. That is the purpose of this RFC.

For the most part, the RFC proposes guidelines that are already followed today,
but it tries to motivate and clarify them.

# Detailed design

Errors fall into one of three categories:

* Catastrophic errors, e.g. out of memory.
* Contract violations, e.g. wrong input encoding, index out of bounds.
* Obstructions, e.g. file not found, parse error.

The basic principle of the conventions is that:

* Catastrophic errors and programming errors (bugs) can and should only be
recovered at a *coarse grain*, i.e. a task boundary.
* Obstructions preventing an operation should be reported at a maximally *fine
grain* -- to the immediate invoker of the operation.

## Catastrophic errors

An error is _catastrophic_ if there is no meaningful way for the current task to
continue after the error occurs.

Catastrophic errors are _extremely_ rare, especially outside of `libstd`.

**Canonical examples**: out of memory, stack overflow.

### For catastrophic errors, abort the process or fail the task.

For errors like out-of-memory, even correctly unwinding is impossible (since it
may require allocation); the process is aborted.

For errors like stack overflow, Rust currently aborts the process, but could in
principle fail the task, which would allow reporting and recovery from a
supervisory task.

## Contract violations

An API may define a contract that goes beyond the type checking enforced by the
compiler. For example, slices support an indexing operation, with the contract
that the supplied index must be in bounds.

Contracts can be complex. For example, the `RefCell` type requires that
`borrow_mut` not be called until all existing borrows have been relinquished.

### For contract violations, fail the task.

A contract violation is always a bug, and for bugs we follow the Erlang
philosophy of "let it crash": we assume that software *will* have bugs, and we
design coarse-grained task boundaries to report, and perhaps recover, from these
bugs.

### Contract design

One subtle aspect of these guidelines is that the contract for a function is
chosen by an API designer -- and so the designer also determines what counts as
a violation.

This RFC does not attempt to give hard-and-fast rules for designing
contracts. However, here are some rough guidelines:

* Prefer expressing contracts through static types whenever possible.

* It *must* be possible to write code that uses the API without violating the
  contract.

* Contracts are most justified when violations are *inarguably* bugs -- but this
  is surprisingly rare.

* Consider whether the API client could benefit from the contract-checking
  logic.  The checks may be expensive. Or there may be useful programming
  patterns where the client does not want to check inputs before hand, but would
  rather attempt the operation and then find out whether the inputs were invalid.

* When in doubt, use loose contracts and instead return a `Result` or `Option`.

## Obstructions

An operation is *obstructed* if it cannot be completed for some reason, even
though the operation's contract has been satisfied. Obstructed operations must
still leave the relevant data structures in a coherent state.

Obstructions may involve external conditions (e.g., I/O), or they may involve
aspects of the input that are not covered by the contract.

**Canonical examples**: file not found, parse error.

### For obstructions, use `Result` (or `Option`).

The
[`Result<T,E>` type](http://static.rust-lang.org/doc/master/std/result/index.html)
represents either a success (yielding `T`) or failure (yielding `E`). By
returning a `Result`, a function allows its clients to discover and react to
obstructions in a fine-grained way.

If there are multiple ways an operation might be obstructed, or there is useful
information about the obstruction (such as where in the input a parse error
occurred), prefer to use `Result`. For operations with a single obvious
obstruction (like popping from an empty stack), use `Option`.

(Currently, `Option` does not interact well with other error-handling
infrastructure like `try!`, but this will likely be improved in the future.)

## Do not provide both `Result` and `fail!` variants.

An API should not provide both `Result`-producing and `fail`ing versions of an
operation. It should provide just the `Result` version, allowing clients to use
`try!` or `unwrap` instead as needed. This is part of the general pattern of
cutting down on redundant variants by instead using method chaining.

There is one exception to this rule, however. Some APIs are strongly oriented
around failure, in the sense that their functions/methods are explicitly
intended as assertions.  If there is no other way to check in advance for the
validity of invoking an operation `foo`, however, the API may provide a
`foo_catch` variant that returns a `Result`.

The main examples in `libstd` providing both variants are:

* Channels, which are the primary point of failure propagation between tasks. As
  such, calling `recv()` is an _assertion_ that the other end of the channel is
  still alive, which will propagate failures from the other end of the
  channel. On the other hand, since there is no separate way to atomically test
  whether the other end has hung up, channels provide a `recv_opt` variant that
  produces a `Result`.

  > **[FIXME]** The `_opt` suffix needs to be replaced by a `_catch` suffix.

* `RefCell`, which provides a dynamic version of the borrowing rules. Calling
  the `borrow()` method is intended as an assertion that the cell is in a
  borrowable state, and will `fail!` otherwise. On the other hand, there is no
  separate way to check the state of the `RefCell`, so the module provides a
  `try_borrow` variant that produces a `Result`.

  > **[FIXME]** The `try_` prefix needs to be replaced by a `_catch` catch.

# Drawbacks

The main drawbacks of this proposal are:

* Task failure remains somewhat of a landmine: one must be sure to document, and
  be aware of, all relevant function contracts in order to avoid task failure.

* The choice of what to make part of a function's contract remains somewhat
  subjective, so these guidelines cannot be used to decisively resolve
  disagreements about an API's design.

The alternatives mentioned below do not suffer from these problems, but have
drawbacks of their own.

# Alternatives

[Two](https://github.com/rust-lang/rfcs/pull/204)
[alternative](https://github.com/rust-lang/rfcs/pull/220) designs have been
given in earlier RFCs, both of which take a much harder line on using `fail!`
(or, put differently, do not allow most functions to have contracts).

As was
[pointed out by @SiegeLord](https://github.com/rust-lang/rfcs/pull/220#issuecomment-54715268),
however, mixing what might be seen as contract violations with obstructions can
make it much more difficult to write obstruction-robust code; see the linked
comment for more detail.
