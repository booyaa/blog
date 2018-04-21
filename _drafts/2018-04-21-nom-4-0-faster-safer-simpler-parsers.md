---
layout: post
title: "nom 4.0: faster, safer, simpler parsers"
date: 2018-04-21 12:00:00 +01:00
categories:
- General
tags:
- Rust
---

I'm delighted to announce that [nom](https://github.com/geal/nom), the extremely
fast Rust parser combinators library, has reached major version 4.

**TL;DR: the new nom version is very nice, you can find a summary of what changed
in [the upgrade documentation](https://github.com/Geal/nom/blob/master/doc/upgrading_to_nom_4.md)**
**side note: how fast is nom? it can reach [2GB/s when parsing HTTP requests](https://github.com/Geal/parser_benchmarks/tree/master/http)**

It took nearly 6 months of development and the library went through nearly 5
entire rewrites. Compare that to previous major releases, which took a month at
most to do. But it was worth it! This new release cleans up a lot of old bugs
and unintuitive behaviours, simplifies some common patterns, is faster, uses less
memory, gives better errors, but the way parsers are written stay the same.
It's like an entirely new engine under the same body work!

## Moving from `IResult` to `Result`

This was a long standing request. nom used a three-legged enum as return type for the parsers:

```rust
// example parser signature
fn parser(input: I) -> IResult<I,O> { ... }

pub enum IResult<I,O,E=u32> {
/// remaining input, result value
Done(I,O),
/// indicates the parser encountered an error. E is a custom error type you can redefine
Error(Err),
/// Incomplete contains a Needed, an enum than can represent a known quantity of input data, or unknown
Incomplete(Needed)
}

pub enum Needed {
/// needs more data, but we do not know how much
Unknown,
/// contains the required total data size
Size(usize)
}

// if the "verbose-errors" feature is not active
pub type Err<E=u32> = ErrorKind;

// if the "verbose-errors" feature is active
pub enum Err<P,E=u32> {
/// An error code, represented by an ErrorKind, which can contain a custom error code represented by E
Code(ErrorKind),
/// An error code, and the next error
Node(ErrorKind, Vec<Err<P,E>>),
/// An error code, and the input position
Position(ErrorKind, P),
/// An error code, the input position and the next error
NodePosition(ErrorKind, P, Vec<Err<P,E>>)
}
```

That old `IResult` structure did not transform well to the commonly used `Result`,
people did not want to see the `Incomplete` case (when the parser indicates it does
not have enough data to decide) if they do not need it. And the different error types
depending on the `verbose-errors` feature were confusing and causing errors when nom
appeared multiple times in dependency trees.

So I replaced it with a new, `Result` based design:

<code>
pub type IResult&lt;I, O, E = u32&gt; = Result&lt;(I, O), Err&lt;I, E&gt;&gt;;

pub enum Err&lt;I, E = u32&gt; {
/// There was not enough data
Incomplete(Needed),
/// The parser had an error (recoverable)
Error(Context&lt;I, E&gt;),
/// The parser had an unrecoverable error
Failure(Context&lt;I, E&gt;),
}

pub enum Needed {
/// needs more data, but we do not know how much
Unknown,
/// contains the required additional data size
Size(usize)
}

// if the "verbose-errors" feature is inactive
pub enum Context&lt;I, E = u32&gt; {
Code(I, ErrorKind),
}

// if the "verbose-errors" feature is active
pub enum Context&lt;I, E = u32&gt; {
Code(I, ErrorKind),
List(Vec&lt;(I, ErrorKind)&gt;),
}</code>


Aside from being more compatible with, like, the whole Rust ecosystem, this new design
has lots of interesting points:

- the `Context` enum is now extended by the `verbose-errors` feature, so it is the same type
- errors always store position information
- `Incomplete` has moved to the error case so you can easily ignore it
- `IResult::Done(remaining, value)` has been replaced with `Ok(remaining, value)` so you could easily do `let (remaining, value) = parser(input)?;` like you would do with other `Result` based functions
- the `Err` enum now contains a`Failure` case epresenting an unrecoverable error (combinators like `alt!` will not try another branch)

And we get all of these benefits while keeping the same memory footprint in "simple" errors mode,
and reducing it in "verbose" errors mode:

| size of `IResult` | simple errors | verbose errors |
|---|---|---|
| nom 3 | 40 bytes | 64 bytes |
| nom 4 | 40 bytes | 48 bytes |

And it gets faster! Depending on the format, I have seen improvements between 4% and 40%!

## Dealing with Incomplete usage

nom's parsers are designed to work around streaming issues: if there is not enough data to decide, a
parser will return `Incomplete` instead of returning a partial value that might be false.

As an example, if you want to parse alphabetic characters then digits, when you get the whole input
`abc123;`, the parser will return `abc` for alphabetic characters, and `123` for the digits, and `;`
as remaining input.

But if you get that input in chunks, like `ab` then `c123;`, the alphabetic characters parser will
return `Incomplete`, because it does not know if there will be more matching characters afterwards.
If it returned `ab` directly, the digit parser would fail on the rest of the input, even though the
input had the valid format.

For some users, though, the input will never be partial (everything could be loaded in memory at once),
and the solution in nom 3 and before was to wrap parts of the parsers with the `complete!()` combinator
that transforms `Incomplete` in `Error`.

nom 4 is much stricter about the behaviour with partial data, but provides better tools to deal with it.
Thanks to the new `AtEof` trait for input types, nom now provides the `CompleteByteSlice(&amp;[u8])` and
`CompleteStr(&amp;str)` input types, for which the `at_eof()` method always returns true.
With these types, no need to put a `complete!()` combinator everywhere, you can juste apply those types
like this:

<code>named!(parser, ... );

// becomes

named!(parser, ... );
</code>

<code>named!(parser, ... );

// becomes

named!(parser, ... );
</code>

<code>named!(parser, ... );

// becomes

named!(parser, ... );
</code>

And as an example, for a unit test:

<code>
assert_eq!(parser("abcd123"), Ok(("123", "abcd"));

// becomes

assert_eq!(parser(CompleteStr("abcd123")), Ok((CompleteStr("123"), CompleteStr("abcd")));
</code>

These types allow you to correctly handle cases like text formats for which there might be a last
empty line or not, as seen in [one of the examples](https://github.com/Geal/nom/blob/87d837006467aebcdb0c37621da874a56c8562b5/tests/multiline.rs).

If those types feel a bit long to write everywhere in the parsers, it's possible
to alias them like this:

<code>
type Input = CompleteByteSlice;
pub fn Input(input:&amp;'a[u8]) -&gt; Input {
CompleteByteSlice(input)
}</code>


## Better documentation

Previous documentation was scattered around and hard to navigate, especially
when trying to find the exact combinator that would work perfectly for what we
want.

So the [new README](https://github.com/Geal/nom/blob/master/README.md) is now
more about what can be done instead of an incomplete reference.

The [documentation homepage](https://docs.rs/nom) is an introduction to parser
combinators and nom's design, with some examples to show common combinators
like `do_parse`.

There's a brand new ["choosing a combinator" doc](https://github.com/Geal/nom/blob/master/doc/choosing_a_combinator.md)
to help you find what you need, arranged by categories, with example usage
and expected results.

And there are new examples for a lot of parser and combinators.

## Various fixes

`no_std` usage is now working correctly. For most of nom's combinators, you
will not need anything more than `core`, and a few basic combinators like
`many0` or `separated_list` require `alloc`.

Some parsers sometimes ended up in the middle of UTF-8 characters, now those
streams are handled correctly.

## the future for nom

[nom 4](https://github.com/geal/nom) is a huge release, and the new design
will probably take some time to settle. There's probably a lot of low hanging
fruit on the performance side, and I look forward to my next obsessive
profiling sessions.

Meanwhile, nom will continue to happily munch bytes for you, as one of the
fastest, most reliable parsing libraries available.

In the future, I'm interested in supporting more use cases, like zero
allocation parsers, and integrating more SIMD work, like what was done for
the HTTP parser. My goal is to get nom parser securing data access in more
and more systems, whatever the language or platform.

