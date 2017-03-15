- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: 2017-03-12
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes several steps forward for `impl Trait`:

- Settling on a particular syntax design, resolving questions around the
  `some`/`any` proposal and others.

- Resolving questions around which type and lifetime parameters are considered
  in scope for an `impl Trait`.

- Adding `impl Trait` to several new locations: argument positions, trait items,
  and trait impls.

The first two proposals, in particular, put us into a position to stabilize the
current version of the feature in the near future.

# Motivation
[motivation]: #motivation

To recap, the current `impl Trait` feature allows functions to write a return
type like `impl Iterator<Item = u64>` or `impl Fn(u64) -> bool`, which says that
the function's return type satisfies the given trait bounds, but nothing more
about it can be assumed. It's useful to impose an abstraction barrier and to
avoid writing down complex (or un-nameable) types. The current feature was
designed very conservatively, and only allows `impl Trait` to be used in
function return position on inherent or free functions.

The core motivation for this RFC is to pave the way toward stabilization of
`impl Trait`; from that perspective, it inherits the motivation of
[the previous RFC](https://github.com/rust-lang/rfcs/pull/1522). Making progress
on this front falls clearly under the rubric of the productivity and
learnability goals for
[the 2017 roadmap](https://github.com/rust-lang/rfcs/pull/1774).

Stabilization is currently blocked on three inter-related questions:

- Will `impl Trait` ever be usable in argument position? With what semantics?

- Will we want to distinguish between `some` and `any`, that is, between
  existential types (where the callee chooses the type) and universal types
  (where the caller chooses)? Or is it enough to deduce the desired meaning from context?

- When you use `impl Trait`, what lifetime and type parameters are in scope for
  the hidden, concrete type that will be returned? Can you customize this set?

This RFC is aimed squarely at resolving these questions. However, by resolving
some of them, it also unlocks the door to an expansion of the feature to new
locations (arguments, traits, trait impls), as we'll see.

# Detailed design
[design]: #detailed-design

## The proposal in a nutshell

- Expand `impl Trait` to allow use in arguments, and in function signatures in
  traits and impls. In argument position, it behaves like an anonymous generic
  parameter. **Each of these expansions will be separately feature-gated.**

- Stick with the `impl Trait` syntax, rather than introducing a `some`/`any`
  distinction.

- Treat all type parameters as in scope for the concrete "witness" type
  underlying a use of `impl Trait`.

- Treat any explicit lifetime bounds (as in `impl Trait + 'a`) as bringing those
  lifetimes into scope, and no other lifetime parameters are explicitly in
  scope. However, type parameters may mention lifetimes which are hence
  *indirectly* in scope.

## Background

Before diving more deeply into the design, let's recap some of the background
that's emerged over time for this corner of the language.

### Universals (`any`) versus existentials (`some`)

There are basically two ways to talk about an "unknown type" in something like a
function signature:

* **Universal quantification**, i.e. "for any type T", i.e. "caller
  chooses". This is how generics work today. When you write `fn foo<T>(t: T)`,
  you're saying that the function will work for *any* choice of `T`, and leaving
  it to your caller to choose the `T`.

* **Existential quantification**, i.e. "for some type T", i.e. "callee
  chooses". This is how `impl Trait` works today (which is in return position
  only). When you write `fn foo() -> impl Iterator`, you're saying that the
  function will produce *some* type `T` that implements `Iterator`, but the
  caller is not allowed to assume anything else about that type.

When it comes to functions, we *usually* want `any T` for arguments, and `some
T` for return values. However, consider the following function:

```rust
fn thin_air<T: Default>() -> T {
    T::default()
}
```

The `thin_air` function says it can produce a value of type `T` for *any* `T`
the caller chooses---so long as `T: Default`. The `collect` function works
similarly. But this pattern is relatively uncommon.

As we'll see later, there are also considerations for *higher-order* functions,
i.e. when you take another function as an argument.

In any case, one longstanding proposal for `impl Trait` is to split it into two
distinct features: `some Trait` and `any Trait`. Then you'd have:

```rust
// These two are equivalent
fn foo<T: MyTrait>(t: T)
fn foo(t: any MyTrait)

// These two are equivalent
fn foo() -> impl Iterator
fn foo() -> some Iterator

// These two are equivalent
fn foo<T: Default>() -> T
fn foo() -> any Default
```

### Scoping for lifetime and type parameters

There's a subtle issue for the semantics of `impl Trait`: what lifetime and type
parameters are considered "in scope" for the underlying concrete type that
implements `Trait`?

#### Type parameters and type equalities

It's easiest to understand this issue through examples where it matters. Suppose
we have the following function:

```rust
fn foo<T>(t: T) -> impl MyTrait { .. }
```

Here we're saying that the function will yield *some* type back, whose identity
we don't know, but which implements `MyTrait`. But, in addition, we have the
type parameter `T`. The question is: can the return type of the function depend
on `T`?

Concretely, we expect at least the following to work:

```rust
vec![
    foo(0u8),
    foo(1u8),
]
```

because we expect both expressions to have the same type, and hence be eligible
to place into a single vector. That's because, although we don't know the
identity of the return type, everything it could depend on is the same in both
cases: `T` is instantiated with `u8`. (Note: there are "generative" variants of
existentials for which this is not the case; see
[Unresolved questions][unresolved]);

But what about the following:

```rust
vec![
    foo(0u8),
    foo(0u16),
]
```

Here, we're making different choices of `T` in the two expressions; can that
affect what return type we get? The `impl Trait` semantics needs to give an
answer to that question.

Clearly there are cases where the return type very much depends on type
parameters, for example the following:

```rust
fn buffer<T: Write>(t: T) -> impl Write {
    BufWriter::new(t)
}
```

But there are also cases where there isn't a dependency, and tracking that
information may be important for type equalities like the vectors above. And
this applies equally to lifetime parameters as well.

#### Lifetime parameters

It's vital to know what lifetime parameters might be used in the concrete type
underlying an `impl Trait`, because that information will affect lifetime
inference.

For concrete types, we're pretty used to thinking about this. Let's take slices:

```rust
impl<T> [T] {
    fn len(&self) -> usize { ... }
    fn first(&self) -> Option<&T> { ... }
}
```

A seasoned Rustacean can read the ownership story directly from these two
signatures. In the case of `len`, the fact that the return type does not involve
any borrowed data means that the borrow of `self` is only used within `len`, and
doesn't need to persist afterwards. For `first`, by contrast, the return value
contains `&T`, which will extend the borrow of `self` for at least as long as
that return value is kept around by the caller.

As a caller, this difference is quite apparent:

```rust
{
    let len = my_slice.len(); // the borrow of `my_slice` lasts only for this line
    *my_slice[0] = 1;         // ... so this mutable borrow is allowed
}

{
    let first = my_slice.first(); // the borrow of `my_slice` lasts for the rest of this scope
    *my_slice[0] = 1;             // ... so this mutable borrow is *NOT* allowed
}
```

Now, the issue is that for `impl Trait`, we're not writing down the concrete
return type, *so it's not obvious what borrows might be active within it*. In
other words, if we write:

```rust
impl<T> [T] {
    fn bork(&self) -> impl SomeTrait { ... }
}
```

it's not clear whether the function is more like `len` or more like `first`.

This is again a question of *what lifetime parameters are in scope* for the
actual return type. It's a question that needs a clear answer (and some
flexibility) for the `impl Trait` design.

## Core assumptions

The design in this RFC is guided by several assumptions which are worth laying
out explicitly.

### Assumption 1: we will eventually have a fully expressive and explicit syntax for existentials

The `impl Trait` syntax can be considered an "implicit" or "sugary" syntax in
that it (1) does not introduce a name for the existential type and (2) does not
allow you to control the scope in which the underlying concrete type is known.

Moreover, some versions of the design (including in this RFC) impose further
limitations on the power of the feature for the same of simplicity.

This is done under the assumption that we will eventually introduce a fully
expressive, explicit syntax for existentials. Such a syntax is sketched in an
appendix to this RFC.

### Assumption 2: treating all *type* variables as in scope for `impl Trait` suffices for the vast majority of cases

The background section discussed scoping issues for `impl Trait`, and the main
implication for *type* parameters (as opposed to lifetimes) is what type
equalities you get for an `impl Trait` return type. We're making two assumptions about that:

- In practice, you usually need to close over most of all of the type parameters.
- In practice, you usually don't care much about type equalities with `impl Trait`.

This latter point means, for example, that it's relatively unusual to do things
like construct the vectors described in the background section.

### Assumption 3: there should be an explicit marker when a lifetime could be embedded in a return type

As mentioned in a
[recent blog post](https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html),
one regret we have around lifetime elision is the fact that it applies when
leaving off a lifetime for a non-`&` type constructor that expects one. For
example, consider:

```rust
impl<T> SomeType<T> {
    fn bork(&self) -> Ref<T> { ... }
}
```

To know whether the borrow of `self` persists in the return value, you have to
know that `Ref` takes a lifetime parameter that's being left out here. This is a
tad too implicit for something as central as ownership.

Now, we also don't want to force you to write an explicit lifetime. We'd instead
prefer a notation that says "there *is* a lifetime here; it's the usual one from
elision". As a purely strawman syntax (an actual RFC on the topic is upcoming),
we might write:

```rust
impl<T> SomeType<T> {
    fn bork(&self) -> Ref<'_, T> { ... }
}
```

In any case, to avoid compounding the mistake around elision, there should be
*some* marker when using `impl Trait` that a lifetime is being captured.

### Assumption 4: existentials are vastly more common in return position, and universals in argument position

As discussed in the background section, it's possible to make sense of `some
Trait` and `any Trait` in arbitrary positions in a function signature. But
experience with the language strongly suggests that `some Trait` semantics is
virtually never wanted in argument position, and `any Trait` semantics is rarely
used in return position.

### Assumption 5: we may be interested in eventually pursuing a bare `fn foo() -> Trait` syntax rather than `fn foo() -> impl Trait`

Today, traits can be used directly as (unsized) types, so that you can write
things like `Box<MyTrait>` to designate a trait object. However, with the advent
of `impl Trait`, there's been a desire to repurpose that syntax, and
[instead write `Box<dyn Trait>`](https://github.com/rust-lang/rfcs/pull/1603) or
some such to designate trait objects.

That would, in particular, allow syntax like the following when taking a closure:

```rust
fn map<U>(self, f: FnOnce(T) -> U) -> Option<U>
```

The pros, cons, and logistics of such a change are out of scope for this
RFC. However, it's taken as an assumption that we want to keep the door open to
such a syntax, and so shouldn't stabilize any variant of `impl Trait` that lacks
a good story for evolving into a bare `Trait` syntax later on.

## Sticking with the `impl Trait` syntax

This RFC proposes to stabilize the `impl Trait` feature with its current syntax,
while also expanding it to encompass argument position. That means, in
particular, *not* introducing an explicit `some`/`any` distinction.

This choice is based partly on the core assumptions:

- Assumption 1, we'll have a fully expressive syntax later.
- Assumption 4, we can use the `some` semantics in return position and `any` in argument position, and almost always be right.
- Assumption 5, we may want bare `Trait` syntax, which would not give "syntactic space" for a `some`/`any` distinction.

One important question is: will people find it easier to understand and use
`impl Trait`, or something like `some Trait` and `any Trait`? Having an explicit
split may make it easier to understand what's going on. But on the other hand,
it's a somewhat complicated distinction to make, and while you usually know
*intuitively* what you want, being forced to spell it out by choosing the
correct choice of `some` or `any` seems like an unnecessary burden, especially
if the choice is almost always dictated by the position.

Pedagogically, if we have an explicit syntax, we retain the option of
explaining what's going on with `impl Trait` by "desugaring" it into that
syntax. From that standpoint, `impl Trait` is meant purely for ergonomics, which
means
[not just what you type, but also what you have to remember](https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html). Having
`impl Trait` "just do the right thing" seems pretty clearly to be the right
choice ergonomically.

## Expansion to arguments

This RFC proposes to allow `impl Trait` in function arguments, in addition to
return position, with the `any Trait` semantics (as per assumption 4). In other
words:

```rust
// These two are equivalent
fn map<U>(self, f: impl FnOnce(T) -> U) -> Option<U>
fn map<U, F>(self, f: F) -> Option<U> where F: FnOnce(T) -> U
```

However, this RFC also proposes to *disallow* use of `impl Trait` within `Fn`
trait sugar or higher-ranked bounds, i.e. to disallow examples like the following:

```rust
fn foo(f: impl Fn(impl SomeTrait) -> impl OtherTrait)
fn bar() -> (impl Fn(impl SomeTrait) -> impl OtherTrait)
```

While we will eventually want to allow such uses, it's likely that we'll want to
introduce nested universal quantifications (i.e., higher-ranked bounds) in at
least some cases; we don't yet have the ability to do so. We can revisit this
question later on, once higher-ranked bounds have gained full expressiveness.

### Explicit instantiation

This RFC does *not* propose any means of explicitly instantiating an `impl
Trait` in argument position. In other words:

```rust
fn foo<T: Trait>(t: T)
fn bar(t: impl Trait)

foo::<u32>(0) // this is allowed
bar::<u32>(0) // this is not
```

Thus, while `impl Trait` in argument position in some sense "desugars" to a
generic parameter, the parameter is treated fully anonymously.

## Expansion to traits

This RFC proposes to allow `impl Trait` in *function signatures* within both
trait definitions and trait impl blocks. It does *not* propose allowing `impl
Trait` to appear in associated type definitions.

Allowing use in traits is motivated by many examples, most prominently:

```rust
trait Iterable {
    type Item;
    fn iter<'a>(&'a self) -> impl Iterator<&'a Item> + 'a
}
```

Here `impl Trait` is used for a type whose identity isn't important, where
introducing an associated type is overkill.

### Functions in trait definitions

Using `impl Trait` in a function within a trait definition is
straightforward. Intuitively, it has the same meaning as with any other
function: `any Trait` in argument position, `some Trait` in return position.

Note, however, that the use in return position is roughly equivalent to
introducing an anonymous associated type that's being returned:

```rust
trait Foo {
    fn bar(&self, f: impl FnOnce());
    fn baz(&self) -> impl Iterator;
}

// is very roughly equivalent to:

trait Foo {
    fn bar<F: FnOnce()>(&self, f: F);

    type __baz: Iterator;
    fn baz(&self) -> __baz;
}
```

However, there are some key differences:

- As mentioned above, `impl Trait` in argument position yields an "anonymous"
  generic parameter; implementations of the trait *must* use `impl Trait` in the
  same position.

- In return position, because the underlying concrete type can close over type
  and lifetime variables in scope, it is in general introducing an
  [associated type constructor](https://github.com/rust-lang/rfcs/pull/1598)
  (ATCs), *not* just an associated type. That's one reason we didn't allow `impl
  Trait` in traits in the first place. But we're now well on our way to
  supporting ATCs in general, so extending the design is not so problematic.

- The associated type (constructor) introduced by `impl Trait` in return
  position is anonymous, and thus not usable within any function other than the
  one that introduced it. In other words, the "scope" of the existential is just
  that function. Larger scopes should be possible with the more expressive and
  explicit syntax sketched in the appendix.

### Functions in trait implementations

When implementing a trait, `impl Trait` can -- and must -- be used *only* when
it appears in the corresponding function signature in the trait's definition:

```rust
trait Foo {
    fn bar(&self, f: impl FnOnce());
    fn baz(&self) -> impl Iterator;
}

impl Foo for MyStruct {
    fn bar(&self, f: impl FnOnce()) { ... }
    fn baz(&self) -> impl Iterator { ... }
}
```

The restrictions are largely there for simplicity; there are a few possible
relaxations that a future RFC could tackle.

### Associated types

While existentials should eventually be usable when defining an associated type,
this RFC proposes to defer that feature to a fully explicit syntax. The appendix
at the end of the RFC sketches what that might look like.

## Scoping for type and lifetime parameters

In argument position, the type fulfilling an `impl Trait` is free to reference
any types or lifetimes whatsoever. So in a signature like:

```rust
fn foo(iter: impl Iterator<Item = u32>);
```

the actual argument type may contain arbitrary lifetimes and mention arbitrary
types. This follows from the desugaring to "anonymous" generic parameters.

For return position, things are more nuanced.

This RFC proposes that *all* type parameters are considered in scope for `impl
Trait` in return position, as per Assumption 2 (which claims that this suffices
for most use-cases) and Assumption 1 (which claims that we'll eventually provide
an explicit syntax with finer-grained control).

The lifetimes in scope include only those mentioned "explicitly" in a bound on
the `impl Trait`. That is:

- For `impl SomeTrait + 'a`, the `'a` is in scope for the concrete witness type.
- For `impl SomeTrait + '_`, the lifetime that elision would imply is in scope
  (this is again using the strawman shorthand syntax for an elided lifetime).

Note, however, that the witness type can freely mention type parameters, which
may themselves involve embedded lifetimes. Consider, for example:

```rust
fn transform(iter: impl Iterator<Item = u32>) -> impl Iterator<Item = u32>
```

Here, if the actual argument type was `SomeIter<'a>`, the return type can
mention `SomeIter<'a>`, and therefore can *indirectly* mention `'a`.

In terms of Assumption 3 -- the constraint that lifetime embedding must be
explicitly marked -- we clearly get that for the explicitly in-scope
variables. For *indirect* mentions of lifetimes, it follows from whatever is
provided for the type parameters, much like the following:

```rust
fn foo<T>(v: Vec<T>) -> vec::IntoIter<T>
```

In this example, the return type can of course reference any lifetimes that `T`
does, but this is apparent from the signature. Likewise with `impl Trait`, where
you should assume that *all* type parameters could appear in the return type.

### Relationship to trait objects

It's worth noting that this treatment of lifetimes is related but not identical
to the way they're handled for trait objects.

In particular, `Box<SomeTrait>` imposes a `'static` requirement on the
underlying object, while `Box<SomeTrait + 'a>` only imposes a `'a`
constraint. The key difference is that, for `impl Trait`, in-scope type
parameters can appear, which indirectly mention additional lifetimes, so `impl
SomeTrait` imposes `'static` only if those type parameters do:

```rust
// In these cases, we know that the concrete return type is 'static
fn foo() -> impl SomeTrait;
fn foo(x: u32) -> impl SomeTrait;
fn foo<T: 'static>(t: T) -> impl SomeTrait;

// In the following case, the concrete return type may embed lifetimes that appear in T:
fn foo<T>(t: T) -> impl SomeTrait;

// ... whereas with Box, the 'static constraint is imposed
fn foo<T>(t: T) -> Box<SomeTrait>;
```

This difference is a natural one when you consider the difference between
generics and trait objects in general -- which is precisely that with generics,
the actual types are *not* erased, and hence auto traits like `Send` work
transparently, as do lifetime constraints.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Generics and traits are a fundamental aspect of Rust, so the pedagogical
approach here is really important. We'll outline the basic contours below, but
in practice it's going to take some trial and error to find the best approach.

One of the hopes for `impl Trait`, as extended by this RFC, is that it *aids*
learnability along several dimensions:

- It makes it possible to meaningfully work with traits without visibly using
  generics, which can provide a gentler learning curve. In particular,
  signatures involving closures are *much* easier to understand. This effect
  would be further heightened if we eventually dropped the need for `impl`, so
  that you could write `fn map<U>(self, f: FnOnce(T) -> U) -> Option<U>`.

- It provides a greater degree of analogy between static and dynamic dispatch
  when working with traits. Introducing trait objects is easier when they can be
  understood as a variant of `impl Trait`, rather than a completely different
  approach. This effect would be further heightened if we moved to `dyn Trait`
  syntax for trait objects.

- It provides a more intuitive way of working with traits and static dispatch in
  an "object" style, smoothing the transition to Rust's take on the topic.

- It provides a more uniform story for static dispatch, allowing it to work in
  both argument and return position.

There are two ways of teaching `impl Trait`:

- Introduce it *prior* to bounded generics, as the first way you learn to
  "consume" traits. That works particularly well with teaching `Iterator` as one
  of the first real traits you see, since `impl Trait` is a strong match for
  working with iterators. As mentioned above, this approach also provides a more
  intuitive stepping stone for those coming from OO-ish languages. Later,
  bounded generics can be introduced as a more powerful, explicit syntax, which
  can also reveal a bit more about the underlying semantic model of `impl
  Trait`.  In this approach, the existential use case doesn't need a great deal
  of ceremony---it just follows naturally from the basic feature.

- Alternatively, introduce it *after* bounded generics, as (1) a sugar for
  generics and (2) a separate mechanism for existentials. This is, of course,
  the way all existing Rust users will come to learn `impl Trait`. And it's
  ultimately important to understand the mechanism in this way. But it's likely
  *not* the ideal way to introduce it at first.

In either case, people should learn `impl Trait` early (since it will appear
often) and in particular prior to learning trait objects. As mentioned above,
trait objects can then be taught using intuitions from `impl Trait`.

There's also some ways in which `impl Trait` can introduce confusion, which
we'll cover in the drawbacks section below.

# Drawbacks
[drawbacks]: #drawbacks

It's widely recognized that we need *some* form of static existentials for
return position, both to be able to return closures (which have un-nameable
types) and to ergonomically return things like iterator chains.

However, there are two broad classes of drawbacks to the approach taken in this RFC.

## Relatively inexpressive sugary syntax

This RFC is built on the idea that we'll eventually have a fully expressive
explicit syntax, and so we should tailor the "sugary" `impl Trait` syntax to
the most common use cases and intuitions.

That means, however, that we give up an opportunity to provide more expressive
but still sugary syntax like `some Trait` and `any Trait`---we certainly don't
want all three.

That syntax is further discussed in Alternatives below.

## Potential for confusion

There are two main avenues for confusion around `impl Trait`:

- Because it's written where a type would normally go, one might expect it to be
  usable *everywhere* a type is accepted (e.g., within `struct` definitions and
  `impl` headers). While it's feasible to allow the feature to be used in more
  locations, the semantics is tricky, and in any case it doesn't behave like a
  normal type, since it's introducing an existential. The approach in this RFC
  is to have a very clear line: `impl Trait` is a notation for function
  signatures only, and there's a separate explicit notation (TBD) that can be
  used to provide more general existentials (which can then be used as if they
  were normal types).

- You can use `impl Trait` in both argument and return position, but the meaning
  is different in the two cases. On the one hand, the meaning is generally the
  intuitive one---it behaves as one would likely expect. But it blurs the line a
  bit between the `some` and `any` meanings, which could lead to people trying
  to use generics for existentials. We may be able to provide some help through
  errors, or eventually provide a syntax like `<out T>` for named existentials.

There's also the fact that `impl Trait` introduces "yet another" way to take a
bounded generic argument (in addition to `<T: Trait>` and `<T> where T:
Trait`). However, these ways of writing a signature are not *semantically*
distinct ways; they're just *stylistically* different. It's feasible that
rustfmt could even make the choice automatically.

# Alternatives
[alternatives]: #alternatives

There's been a *lot* of discussion about the `impl Trait` feature and various
alternatives. Let's look at some of the most prominent of them.

- **Limiting to return position forever**. A particularly conservative approach
  would be to treat `impl Trait` as used purely for existentials and limit its
  use to return position in functions (and perhaps some other places where we
  want to allow for existentials). Limiting the feature in this way, however,
  loses out on some significant ergonomic and pedagogical wins (previously
  discussed in the RFC), and risks confusion around the "special case" treatment
  of return types.

- **Finer grained sugary syntax**. There are a couple options for making the sugary syntax more powerful:

  - `some`/`any` notation, which allows selecting between universals and
    existentials at will. The RFC has already made some argument for why it does
    not seem so important to permit this distinction for `impl Trait`. And doing
    so has some significant downsides: it demands a more sophisticated
    understanding of the underlying type theory, which precludes using `impl
    Trait` as an early teaching tool; it seems easy to get confused and choose
    the wrong variant; and we'd almost certainly need different keywords (that
    don't mirror the existing `Some` and `Any` names), but it's not clear that there are good choices.

  - `impl<...> Trait` syntax, as a way of giving more precise control over which
  type and lifetime parameters are in scope. The idea is that the parameters
  listed in the `<...>` are in scope, and nothing else is. This syntax, however,
  is not forward-compatible with a bare `Trait` syntax. It's also not clear how
  to get the right *defaults* without introducing some inconsistency; if you
  leave off the `<>` altogether, we'd presumably like something like the
  defaults proposed in this RFC (otherwise, the feature would be very
  unergonomic). But that would mean that, when transitioning from no `<>` to
  including a `<>` section, you go from including *all* type parameters to
  including only the listed set, which is a bit counterintuitive.

# Unresolved questions
[unresolved]: #unresolved-questions

**Full evidence for core assumptions**. The assumptions in this RFC are stated
  with anecdotal and intuitive evidence, but the argument would be stronger with
  more empirical evidence. It's not entirely clear how best to gather that,
  though many of the assumptions could be validated by using an unstable version
  of the proposed feature.

**The precedence rules around `impl Trait + 'a` need to be nailed down.**

**The RFC assumes that we only want "applicative" existentials**, which always
resolve to the same type when in-scope parameters are the same:

```rust
fn foo() -> impl SomeTrait { ... }

fn bar() {
    // valid, because we know the underlying return type will be the same in both cases:
    let v = vec![foo(), foo()];
}
```

However, it's also possible to provide "generative" existentials, which give you
a *fresh* type whenever they are unpacked, even when their arguments are the
same---which would rule out the example above. That's a powerful feature,
because it means in effect that you can generate a fresh type *for every dynamic
invocation of a function*, thereby giving you a way to hoist dynamic information
into the type system.

As one example, generative existentials can be used to "bless" integers as being
in bounds for a particular slice, so that bounds checks can be safely
elided. This is currently possible to encode in Rust by using callbacks with
fresh lifetimes (see Section 6.3 of
[@Gankro's thesis](https://github.com/Gankro/thesis/raw/master/thesis.pdf), but
generative existentials would provide a much more natural mechanism.

We may want to consider adding some form of generative existentials in the
future, but would almost certainly want to do so via the fully
expressive/explicit syntax, rather than through `impl Trait`.

# Appendix: a sketch of a fully-explicit syntax

This section contains a **brief sketch** of a fully-explicit syntax for
existentials. It's a strawman proposal based on many previously-discussed ideas,
and should not be bikeshedded as part of this RFC. The goal is just to give a
flavor of how the full system could eventually fit together.

The basic idea is to introduce an `abstype` item for declaring abstract types:

```rust
abstype MyType: SomeTrait;
```

This construct would be usable anywhere items currently are. It would declare an
existential type whose concrete implementation is known **within the item scope
in which it is declared**, and that concrete type would be determined by
inference based on the same scope. Outside of that scope, the type would be
opaque in the same way as `impl Trait`.

So, for example:

```rust
mod example {
    static NEXT_TOKEN: Cell<u64> = Cell::new(0);

    pub abstype Token: Eq;
    pub fn fresh() -> Token {
        let r = NEXT_TOKEN.get();
        NEXT_TOKEN.set(r + 1);
        r
    }
}

fn main() {
    assert!(example::fresh() != example::fresh());

    // fails to compile, because in this scope we don't know that `Token` is `u64`
    let _ = example::fresh() + 1;
}
```

Of course, in this particular example we could just as well have used `fn
fresh() -> impl Eq`, but `abstype` allows us to use the *same* existential type in multiple locations in an API:

```rust
mod example {
    pub abstype Secret: SomeTrait;

    pub fn foo() -> Secret { ... }
    pub fn bar(s: Secret) -> Secret { ... }

    pub struct Baz {
        quux: Secret,
        // ...
    }
}
```

Already `abstype` gives greater expressiveness than `impl Trait` in several
respects:

- It allows existentials to be named, so that the same one can be referred to
multiple times within an API.

- It allows existentials to appear within structs.

- It allows existentials to appear within function arguments.

- It gives tight control over the "scope" of the existential---what portion of
  the code is allowed to know what the concrete witness type for the existential
  is. For `impl Trait`, it's always just a single function.


But we also wanted more control over scoping of type and lifetime
parameters. For this, we can introduce existential *type constructors*:

```rust
abstype MyIter<'a>: Iterator<Item = u32>;

impl SomeType<T> {
    // we know that 'a is in scope for the return type, but *not* `T`
    fn iter<'a>(&'a self, ) -> MyIter<'a> { ... }
}
```

(These type constructors raise various issues around inference, which I believe
are tractable, but are out of scope for this sketch).

It's worth noting that there's some relationship between `abstype` and the
"newtype deriving" concept: from an external perspective, `abstype` introduces a
new type but automatically delegates any of the listed trait bounds to the
underlying witness type.

Finally, a word on syntax:

- Why `abstype Foo: Trait;` rather than `type Foo = impl Trait;`?
  - Two reasons. First, to avoid confusion about `impl Trait` seeming to be like a
  type, when it is actually an existential. Second, for forward compatibility
  with bare `Trait` syntax.

- Why not `type Foo: Trait`?
  - That may be a fine syntax, but for clarity in presenting the idea I
    preferred to introduce a new keyword.

There are many detailed questions that would need to be resolved to fully
specify this more expressive syntax, but the hope here is to show that (1)
there's a plausible direction to take here and (2) give a sense for how `impl
Trait` and a more expressive form could fit together.
