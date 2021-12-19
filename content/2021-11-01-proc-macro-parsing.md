+++
title = "Procedural Macros: Parsing custom syntax"
draft = true
+++

This is the fourth article in my series about procedural macros. In this
article, I will explain how you can use `syn` to parse things that are not Rust
code. It will once again extend the `Getters` derive macro example from the
previous two articles

* [Procedural Macros: A simple derive macro](/proc-macro-simple-derive/) and
* [Procedural Macros: Error handling](/proc-macro-error-handling/),

so it's easier to follow along if you're read them already, but it should be
understandable on its own if you are already somewhat familiar with writing
proc-macros using `syn` & `quote`.

## The plan

In the last article, we added a custom attribute for our derive macro that uses
the syntax `#[getter(name = "foo")]`. But what if we decide it should really be
`#[getter(name = foo)]`, because why would you have to quote the name?

<div class="info">

Note that custom attributes (either registered by derive macros, or as
standalone attribute proc-macros) are not the only use case for parsing custom
syntax. Perhaps more importantly, you can have your own
<span class="abbrev" title="domain-specific languages">DSLs</span> within
function-like macros (or attributes). An advanced example of this is [Yew]'s
[`html!` macro](https://yew.rs/concepts/html).

[Yew]: https://yew.rs/

</div>

This won't work with `syn`s `Meta` type because it predates [support for
arbitrary token streams in proc-macro attributes][unrestricted_attribute_tokens]
in the Rust compiler and arbitrary token streams can't really have a
representation that is similarily easy to pattern match on. That is why we need
some parsing code to extract the data we want from the attribute now.

[unrestricted_attribute_tokens]: https://blog.rust-lang.org/2019/04/11/Rust-1.34.0.html#custom-attributes-accept-arbitrary-token-streams

## `syn` parsing basics

In the previous article when the attribute was added, attribute parsing was a
single line of code out of >30 lines of attribute handling code:

```rust
fn get_name_attr(attr: &Attribute) -> syn::Result<Option<Ident>> {
    // Parsing into `syn::Meta`:
    let meta = attr.parse_meta()?;

    // Pattern matching and error handling...
}
```

When parsing into our own type, we can avoid all of the pattern matching and
even get some pretty good error handling "for free", but depending on the syntax
you want to parse, the parsing code can take a little while to get right.

First thing first though: We need a type to parse the `#[getter(name = foo)]`
attribute into. Since the attribute is not mandatory and other arguments to it
might reasonably be added in the future, we will make it a struct with an
*optional* `name` field:

```rust
struct GetterMeta {
    name: Option<Ident>,
}
```

Luckily when it comes to the parsing code, all we have to parse for now is
`identifier = identifier`; this is because `syn` already takes care of parsing
the attribute's name and giving us only the argument list to parse when using
`Attribute::parse_args`. Here's an illustration from `syn`s
[documentation for that method][parse_args_docs]:

```
#[my_attr(value < 5)]
          ^^^^^^^^^ what gets parsed
```

[parse_args_docs]: https://docs.rs/syn/latest/syn/struct.Attribute.html#method.parse_args

There is a trait we have to implement to make `GetterMeta` usable with
`Attribute::parse_args`, and it's called `Parse`:

```rust
use syn::parse::{Parse, ParseStream};

impl Parse for GetterMeta {
    fn parse(input: ParseStream<'_>) -> syn::Result<Self> {
        todo!()
    }
}
```

Now for the actual parsing code... It is so simple you might wonder why you
would ever bother with `syn::Meta` (spoiler: it's not always this simple).

```rust
use syn::{Ident, Token};

fn parse(input: ParseStream<'_>) -> syn::Result<Self> {
    // Parse the argument name
    let arg_name: Ident = input.parse()?;
    if arg_name != "name" {
        // Same error as before when encountering an unsupported attribute
        return Err(syn::Error::new_spanned(
            arg_name,
            "unsupported getter attribute, expected `name`",
        ));
    }

    // Parse (and discard the span of) the `=` token
    let _: Token![=] = input.parse()?;

    // Parse the argument value
    let name = input.parse()?;

    Ok(Self { name })
}
```

Now just update the `get_name_attr` implementation:

```rust
fn get_name_attr(attr: &Attribute) -> syn::Result<Option<Ident>> {
    let meta: GetterMeta = attr.parse_args()?;
    Ok(meta.name)
}
```

… and parsing of `#[getter(name = foo)]` works!

## Parsing lists

*allow users to rename getters while keeping backwards compatibility*

[syn_punctuated]: https://docs.rs/syn/latest/syn/punctuated/struct.Punctuated.html

## Custom keywords

## Appendix: Deprecating syntax

*re-add support for the literal variation*
