- Feature Name: `intra_rustdoc_links`
- Start Date: 2017-03-06
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add a notation how to create relative links in documentation comments
(based on Rust item paths)
and extend Rustdoc to automatically turn this into working links.


# Motivation
[motivation]: #motivation

It is good practice in the Rust community to
add documentation to all public items of a crate,
as the API documentation as rendered by Rustdoc is the main documentation of most libaries.
Documentation comments at the module (or crate) level are used to
give an overview of the module (or crate)
and describe how the items of a crate can be used together.
To make navigating the documentation easy,
crate authors make these items link to their individual entries
in the API docs.

Currently, these links are plain Markdown links,
and the URLs are the (relative) paths of the items' pages
in the rendered Rustdoc output.
This is sadly very fragile in several ways:

1. As the same doc comment can be rendered on several Rustdoc pages
  and thus on separate directory levels
  (e.g., the summary page of a module, and a struct's own page),
  it is not possible to confidently use relative paths.
  For example,
  adding a link to `../foo/struct.Bar.html`
  to the first paragraph of the doc comment of the module `lorem`
  will work on the rendered `/lorem/index.html` page,
  but not on the crate's summary page `/index.html`.
2. Using absolute paths in links
  (like `/crate-name/foo/struct.Bar.html`)
  to circumvent the previous issue
  might work for the author's own hosted version,
  but will break when
  looking at the documentation using `cargo doc --open`
  (which uses `file:///` URLs)
  or when using docs.rs.
3. Should Rustdoc's file name scheme ever change
  (it has change before, cf. [Rust issue #35236]),
  all manually created links need to be updated.

[Rust issue #35236]: https://github.com/rust-lang/rust/pull/35236

To solve this dilemma,
we propose extending Rustdoc
to be able to generate relative links that work in all contexts.


# Detailed design
[design]: #detailed-design

[Markdown][md]/[CommonMark] allow writing links in several forms
(the names are from the [CommonMark spec][cm-spec] in version 0.27):

[md]: https://daringfireball.net/projects/markdown/syntax
[CommonMark]: http://commonmark.org
[cm-spec]: http://spec.commonmark.org/0.27/

1. `[link text](URL)`
  ([inline link][il])
2. `[link text][link label]`
  ([reference link][rl],
  link label can also be omitted, cf. [shortcut reference links][srl])
  and somewhere else in the document: `[link label]: URL`
  (this part is called [link reference definition][lrd])
3. `<URL>` which will be turned into the equivalent of `[URL](URL)`
  ([autolink][al], required to start with a schema)

[il]: http://spec.commonmark.org/0.27/#inline-link
[rl]: http://spec.commonmark.org/0.27/#reference-link
[srl]: http://spec.commonmark.org/0.27/#shortcut-reference-link
[al]: http://spec.commonmark.org/0.27/#autolinks
[lrd]: http://spec.commonmark.org/0.27/#link-reference-definitions

We prospose that
in each occurance of `URL`
of inline links and link reference definitions,
it should also be possible to write a Rust path
(as defined [in the reference][ref-paths]).
Additionally, automatic [link reference definitions][lrd] should be generated
to allow easy linking to obvious targets.

[ref-paths]: https://github.com/rust-lang-nursery/reference/blob/2d23ea601f017c106a2303094ee1c57ba856d246/src/paths.md

## Additions to the documentation syntax

Rust paths as URLs in inline and reference links:

1. `[Iterator](std::iter::Iterator)`
2. `[Iterator][iter]`,
  and somewhere else in the document: `[iter]: std::iter::Iterator`
3. `[Iterator]`,
  and somewhere else in the document: `[Iterator]: std::iter::Iterator`

## Implied Shortcut Reference Links
[isrl]: #implied-shortcut-reference-links

The third syntax example above shows a
[shortcut reference link][srl],
which is a reference link
whose link text and link label are the same,
and there exists a link reference definition for that label.
For example: `[HashMap]` will be rendered as a link
given a link reference definition like ```[HashMap]: std::collections::HashMap```.

To make linking to items easier,
we introduce "implied link reference definitions":

1. `[std::iter::Iterator]`,
  without having a link reference definition for `Iterator` anywhere else in the document
2. ```[`std::iter::Iterator`]```,
  without having a link reference definition for `Iterator` anywhere else in the document
  (same as previous style but with back ticks to format link as inline code)

If Rustdoc finds a shortcut reference link

1. without a matching link reference definition
2. whose link label,
  after stripping leading and trailing back ticks,
  is a valid Rust path

it will add a link reference definition
for this link label pointing to the Rust path.

[Collapsed reference links][crf] (`[link label][]`) are handled analogously.

[crf]: http://spec.commonmark.org/0.27/#collapsed-reference-link

(This was one of the first ideas suggested
by [CommonMark forum] members
as well as by [Guillaume Gomez].)

[CommonMark forum]: https://talk.commonmark.org/t/what-should-the-rust-community-do-for-linkage/2141
[Guillaume Gomez]: https://github.com/GuillaumeGomez

## Standard-conform Markdown

These additions are valid Markdown,
as defined by the orginal [Markdown syntax definition][md]
as well as the [CommonMark] project.
Especially, Rust paths are valid CommonMark [link destinations],
even with the suffixes described [below][path-ambiguities].

[link destinations]: http://spec.commonmark.org/0.27/#link-destination

## How links will be rendered

The following:

```rust
The offers several ways to fooify [Bars](bars::Bar).
```

should be rendered as:

```html
The offers several ways to fooify <a href="bars/struct.Bar.html">Bars</a>.
```

when on the crates index page (`index.html`),
and as this
when on the page for the `foos` module (`foos/index.html`):

```html
The offers several ways to fooify <a href="../bars/struct.Bar.html">Bars</a>.
```

## No Autolinks Style

When using the autolink syntax (`<URL>`),
the URL has to be an [absolute URI],
i.e., it has to start with an URI scheme.
Thus, it will not be possible to write `<Foo>`
to link to a Rust item called `Foo`
that is in scope
(this also conflicts with Markdown ability to contain arbitrary HTML elements).
And while `<std::iter::Iterator>` is a valid URI
(treating `std:` as the scheme),
to avoid confusion, the RFC does not propose adding any support for autolinks.

[absolute URI]: http://spec.commonmark.org/0.27/#absolute-uri

This means that this will not render a valid link:

```
Does not work: <bars::Bar> :(
```

We suggest to use [Implied Shortcut Reference Links][isrl] instead:

```
Does work: [`bars::Bar`] :)
```

which will be rendered as

```html
Does work: <a href="../bars/struct.Bar.html"><code>bars::Bar</code></a> :)
```

## Resolving paths

The Rust paths used in links are resolved
relative to the item in whose documentation they appear.

For example:

```rust
/// Container for a [Dolor](ipsum::Dolor).
struct Lorem(ipsum::Dolor);

/// Contains various things, mostly [Dolor](ipsum::Dolor) and a helper function,
/// [sit](ipsum::sit).
mod ipsum {
    struct Dolor;

    /// Takes a [Dolor] and does things.
    fn sit(d: Dolor) {}
}

mod amet {
  //! Helper types, can be used with the [ipsum](super::ipsum) module.
}
```

## Path ambiguities
[path-ambiguities]: #path-ambiguities

Rust allows items to have the same name. A short evaluation revealed that

- unit and tuple struct names
  can conflict with
  function names,
- unit and tuple struct names
  can conflict with
  enum names,
- and regular struct, enum, and trait names
  can conflict with
  each other
  (but not function names).

It appears the ambiguity can resolved
by being able to restrict the path to function names.
We propose that in case of ambiguity,
you can add `()` as a suffix to path
to only search for functions.
(Additionally, to link to macros, you must add `!` to the path.)

This was originally proposed by
[@kennytm](https://github.com/kennytm)
[here](https://github.com/rust-lang/rfcs/pull/1946#issuecomment-284719684),
going into more details:

> `<syntax::ptr::P>` — First search for type-like items. If not found, search for value-like items
>
> `<syntax::ptr::P()>` — Only search for functions.
>
> `<std::stringify!>` — Only search for macros.

## Linking to external crates

Rustdoc is already able to link to external crates,
and renders documentation for all dependencies by default.
Referencing the standard library (or `core`)
should generate links with a well-known base path,
e.g. `https://doc.rust-lang.org/nightly/`.
Referencing other external crates
links to the pages Rustdoc has already rendered (or will render) for them.
Special flags (e.g. `cargo doc --no-deps`) will not change this behavior.

## Errors

Ideally, Rustdoc would be able to recognize Rust path syntax,
and if the path cannot be resolved,
print a warning (or an error).

## Complex example

(Excerpt from Diesel's [`expression`][diesel-expression] module.)

[diesel-expression]: https://github.com/diesel-rs/diesel/blob/1daf2581919d82b80c18f00957e5c3d35375c4c0/diesel/src/expression/mod.rs

```rust
// diesel/src/expression/mod.rs

//! AST types representing various typed SQL expressions. Almost all types
//! implement either [`Expression`] or [`AsExpression`].

/// Represents a typed fragment of SQL. Apps should not need to implement this
/// type directly, but it may be common to use this as type boundaries.
/// Libraries should consider using [`infix_predicate!`] or
/// [`postfix_predicate!`] instead of implementing this directly.
pub trait Expression {
    type SqlType;
}

/// Describes how a type can be represented as an expression for a given type.
/// These types couldn't just implement [`Expression`] directly, as many things
/// can be used as an expression of multiple types. ([`String`] for example, can
/// be used as either [`VarChar`] or [`Text`]).
///
/// [`VarChar`]: diesel::types::VarChar
/// [`Text`]: diesel::types::Text
pub trait AsExpression<T> {
    type Expression: Expression<SqlType=T>;
    fn as_expression(self) -> Self::Expression;
}
```

Please note:

- This uses implied shortcut reference links most often.
  Since the original documentation put the type/trait names in back ticks to render them as code, we preserved this style.
  (We don't propose this as a general convention, though.)
- Even though implied shortcut reference links could be used throughout,
  they are not used for the last two links (to `VarChar` and `Text`),
  which are not in scope and need to be linked to by their absolute Rust path.
  To make reading easier and less noisy, reference links are used to rename the links.
  (An assumption is that most readers will recognize these names and know they are part of `diesel::types`.)


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

- Extend the documentation chapter of the book with a subchapter on How to Link to Items.
- Reference the chapter on the module system, to let reads familiarize themselves with Rust paths.
- Maybe present an example use case of a module whose documentation links to several related items.


# Drawbacks
[drawbacks]: #drawbacks

- Rustdoc gets more complex.
- These links won't work when the doc comments are rendered with a default Markdown renderer.
- The Rust paths might conflict with other valid links,
  though we could not think of any.


# Possible Extensions
[possible-extensions]: #possible-extensions

## Linking to methods

To link to methods, it may be necessary to use fully-qualified paths,
like `<Foo as Bar>::bar`.
We have yet to analyse in which cases this is necessary,
and this syntax is currently not described in [the reference's section on paths][ref-paths].


# Alternatives
[alternatives]: #alternatives

- Prefix Rust paths with a URI scheme, e.g. `rust:`
  (cf. [path ambiguities][path-ambiguities]).
- Prefix Rust paths with a URI scheme for the item type, e.g. `struct:`, `enum:`, `trait:`, or `fn:`.

- [javadoc] and [jsdoc]
  use `{@link java.awt.Panel}`
  or `[link text]{@link namepathOrURL}`

[javadoc]: http://docs.oracle.com/javase/8/docs/technotes/tools/windows/javadoc.html
[jsdoc]: http://usejsdoc.org/tags-inline-link.html

- [@kennytm](https://github.com/kennytm)
  listed other syntax alternatives
  [here](https://github.com/rust-lang/rfcs/pull/1946#issuecomment-284718018).


# Unresolved questions
[unresolved]: #unresolved-questions

- Is it possible for Rustdoc to resolve paths?
  Is it easy to implement this?
- There is talk about switching Rustdoc to a different markdown renderer ([pulldown-cmark]).
  Does it support this?
  Does the current renderer?

[pulldown-cmark]: https://github.com/google/pulldown-cmark/
