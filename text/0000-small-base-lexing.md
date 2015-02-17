- Feature Name: stable, it restricts the language
- Start Date: 2015-02-17
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Lex binary and octal literals as if they were decimal.

# Motivation

Lexing all digits (even ones not valid in the given base) allows for
improved error messages & future proofing (this is more conservative
than the current approach) and less confusion, with little downside.

Currently, the lexer stop lexing a binary and octal literal (`0b10`
and `0o12345670`) as soon as it sees an invalid digit (2-9 or 8-9
respectively), and start lexing a new token, e.g. `0b0123` is two
tokens, `0b01` and `23`. Writing such a thing in normal code gives a
strange error message:

```rust
<anon>:2:9: 2:11 error: expected one of `.`, `;`, `}`, or an operator, found `23`
<anon>:2     0b0123
                 ^~
```

However, it is valid to write such a thing in a macro (e.g. using the
`tt` non-terminal), and thus lexing the adjacent digits as two tokens
can lead to unexpected behaviour.

```rust
macro_rules! expr { ($e: expr) => { $e } }

macro_rules! add {
    ($($token: tt)*) => {
        0 $(+ expr!($token))*
    }
}
fn main() {
    println!("{}", add!(0b0123));
}
```

prints `24` (`add` expands to `0 + 0b01 + 23`).

It would be much nicer for both cases to print an error like:

```rust
error: found invalid digit `2` in binary literal
0b0123
    ^
```

Code that wants two tokens can opt in to it by `0b01 23`, for
example. This is easy to write, and expresses the intent more clearly
anyway.

# Detailed design

The grammar that the lexer uses becomes

```
(0b[0-9]+ | 0o[0-9]+ | [0-9]+ | 0x[0-9a-fA-F]+) suffix
```

instead of just `[01]` and `[0-7]` for the first two, respectively.

However, it is always an error (in the lexer) to have invalid digits
in a numeric literal beginning with `0b` or `0o`. In particular, even
a macro invocation like

```rust
macro_rules! ignore { ($($_t: tt)*) => { {} } }

ignore!(0b0123)
```

is an error even though it doesn't use the tokens.


# Drawbacks

This adds a slightly peculiar special case, that is somewhat unique to
Rust. On the other hand, most languages do not expose the lexical
grammar so directly, and so have more freedom in this respect. That
is, in many languages it is indistinguishable if `0b1234` is one or
two tokens: it is *always* an error either way.


# Alternatives

Not do it, obviously.

# Unresolved questions

None.
