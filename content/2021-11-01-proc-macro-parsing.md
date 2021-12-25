+++
title = "Procedural Macros: Parsing custom syntax"
draft = true
+++

This is the fourth article in my series about procedural macros. In this
article, I will explain how you can use `syn` to parse things that are not Rust
code. It will once again extend the `Getters` derive macro (which generates get
methods for a struct's fields) from the previous two articles

* [Procedural Macros: A simple derive macro](/proc-macro-simple-derive/) and
* [Procedural Macros: Error handling](/proc-macro-error-handling/),

so it's easier to follow along if you're read them already, but it should be
understandable on its own if you are already somewhat familiar with writing
proc-macros using `syn` & `quote`.

## Why use custom parsing?

In the last article, we added a custom attribute for our derive macro that uses
the syntax `#[getter(name = "foo")]`. But what if we decide it should really be
`#[getter(name = foo)]`, because why would you have to quote the name?

This won't work with `syn`s `Meta` type because it predates [support for
arbitrary token streams in proc-macro attributes][unrestricted_attribute_tokens]
in the Rust compiler and arbitrary token streams can't really have a
representation that is similarily easy to pattern match on. That is why we need
some parsing code to extract the data we want from the attribute now.

[unrestricted_attribute_tokens]: https://blog.rust-lang.org/2019/04/11/Rust-1.34.0.html#custom-attributes-accept-arbitrary-token-streams

<div class="info">

Note that custom attribute arguments are not the only use case for parsing
custom syntax. Perhaps more importantly, you can have your own
<span class="abbrev" title="domain-specific languages">DSLs</span> within
function-like macros (or attributes). An advanced example of this is [Yew]'s
[`html!` macro](https://yew.rs/concepts/html).

[Yew]: https://yew.rs/

</div>

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
optional `name` field:

```rust
struct GetterMeta {
    name: Option<Ident>,
}
```

Luckily when it comes to the parsing code, all we have to parse for now is
`name = <identifier>`; this is because `syn` already takes care of parsing
the attribute's name and giving us only the argument list to parse when using
`Attribute::parse_args`. Here's an illustration from
[the documentation for that method][parse_args_docs]:

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

Now for the actual parsing code… It is so simple you might wonder why you
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

    Ok(Self { name: Some(name) })
}
```

<div class="info">

If you are wondering about the `Token` macro above: It is simply a nice and easy
way to refer to the types in `syn`s [`token` module][mod_token]. See
[its documentation][macro_token] for more information.

[mod_token]: https://docs.rs/syn/1.0/syn/token/index.html
[macro_token]: https://docs.rs/syn/1.0/syn/macro.Token.html

</div>

Now just update the `get_name_attr` implementation:

```rust
fn get_name_attr(attr: &Attribute) -> syn::Result<Option<Ident>> {
    let meta: GetterMeta = attr.parse_args()?;
    Ok(meta.name)
}
```

… and parsing of `#[getter(name = foo)]` works!

<div class="info">

If you are reading this separate from [the previous article][prev_article] and
need a quick reminder on how this function fits in with the rest of the macro
code, you can have a look at the full code including the changes from above
[here][parsing_basics_code].

[parsing_basics_code]: https://github.com/jplatte/proc-macro-blog-examples/tree/8eadec2c3c95f9e2e975aa402653d476ead8c71d/derive_getters

</div>

<div class="info">

If you are interested in a small trick that allows you to change the intended
syntax of a macro like here in a way that only shows a deprecation warning for
uses of the previous style rather than breaking those uses, have a look at
[Appendix A](#a-deprecating-custom-syntax).

</div>

## Branching

Now that the most basic case is covered, how about something a little more
complex? Let's say we want to add support for setting the visibility of a
generated getter function via `#[getter(vis = <visibility>)]`. Of course
specifying both a custom name and visibility at the same time should be
supported too, in whatever order the user prefers. That means error handling
won't be the only kind of branching needed anymore.

```rust
// Previous possibility:
#[getter(name = foo)]
field: Ty,

// New possibilities: (1)
// Override the visibility of a generated getter
#[getter(vis = pub(crate))]
field: Ty,

// New possibilities: (2)
// Override name and visibility in separate attributes
#[getter(name = foo)]
#[getter(vis = pub(crate))]
field: Ty,
#[getter(vis = pub(crate))]
#[getter(name = foo)]
field: Ty,

// New possibilities: (3)
// Override name and visibility in a single attribute
#[getter(name = foo, vis = pub(crate))]
field: Ty,
#[getter(vis = pub(crate), name = foo)]
field: Ty,
```

The first thing we need to do to support any of these is add another field to
the `GetterMeta` struct:

```rust
use syn::Visibility;

struct GetterMeta {
    name: Option<Ident>,
    vis: Option<Visibility>,
}
```

Adding support for case (1) doesn't require much work, the `Parse`
implementation for `GetterMeta` simply needs one more branch:

```rust
impl Parse for GetterMeta {
    fn parse(input: ParseStream<'_>) -> syn::Result<Self> {
        let arg_name: Ident = input.parse()?;
        if arg_name == "name" {
            let _: Token![=] = input.parse()?;
            let name = input.parse()?;

            Ok(Self { name: Some(name), vis: None })
        } else if arg_name == "vis" {
            let _: Token![=] = input.parse()?;
            let vis = input.parse()?;

            Ok(Self { name: None, vis: Some(vis) })
        } else {
            Err(syn::Error::new_spanned(
                arg_name,
                "unsupported getter attribute, expected `name` or `vis`",
            ))
        }
    }
}
```

For case (2), we introduce `GetterMeta::merge` as a way to reduce two
`GetterMeta`s to one, raising an error if one of the arguments is provided
multiple times:

```rust
use quote::ToTokens;

impl GetterMeta {
    fn merge(self, other: GetterMeta) -> syn::Result<Self> {
        fn either<T: ToTokens>(a: Option<T>, b: Option<T>) -> syn::Result<Option<T>> {
            match (a, b) {
                (None, None) => Ok(None),
                (Some(val), None) | (None, Some(val)) => Ok(Some(val)),
                (Some(a), Some(b)) => {
                    let mut error = syn::Error::new_spanned(a, "redundant attribute argument");
                    error.combine(syn::Error::new_spanned(b, "note: first one here"));
                    Err(error)
                }
            }
        }

        Ok(Self { name: either(self.name, other.name)?, vis: either(self.vis, other.vis)? })
    }
}
```

Then comes the adjustment of the actual getter method generation. Here is the
*previous* code for that:

```rust
let attrs: Vec<_> =
    f.attrs.iter().filter(|attr| attr.path.is_ident("getter")).collect();

let name_from_attr = match attrs.len() {
    0 => None,
    1 => get_name_attr(attrs[0])?,
    _ => {
        let mut error =
            syn::Error::new_spanned(attrs[1], "redundant `getter(name)` attribute");
        error.combine(syn::Error::new_spanned(attrs[0], "note: first one here"));
        return Err(error);
    }
};

let method_name =
    name_from_attr.unwrap_or_else(|| f.ident.clone().expect("a named field"));
let field_name = f.ident;
let field_ty = f.ty;

Ok(quote! {
    pub fn #method_name(&self) -> &#field_ty {
        &self.#field_name
    }
})
```

Now that we want to check all of the attributes, we no longer need to first
check how many `getter` attributes there are, we can simply parse all of them
and fold them into one using the new `GetterMeta::merge`:

```rust
let meta: GetterMeta = f
    .attrs
    .iter()
    .filter(|attr| attr.path.is_ident("getter"))
    // First create an initial empty GetterMeta (using a derived Default impl).
    // Then try parsing the attributes one-by-one and merging them into that
    // instance, stopping if there is an errors from either `.parse_args()` or
    // `.merge()` and propagating the first error (if any) out of this code with
    // the second `?`.
    .try_fold(GetterMeta::default(), |meta, attr| meta.merge(attr.parse_args()?))?;

// Extract visibility argument in addition to name (falling back to `pub`)
let visibility = meta.vis.unwrap_or_else(|| parse_quote! { pub });
let method_name =
    meta.name.unwrap_or_else(|| f.ident.clone().expect("a named field"));
let field_name = f.ident;
let field_ty = f.ty;

Ok(quote! {
    // vvvvv Usage of visibility argument
    #visibility fn #method_name(&self) -> &#field_ty {
        &self.#field_name
    }
})
```

There's of course many alternative ways to solve this, but I found `try_fold` to
be the most elegant solution after some experimentation.

Now onto case (3)…

## Lists

One way of dealing with the case of multiple arguments in one attribute would be
to write a parsing loop and call `input.parse()?` for the list delimiters like
for the argument names, `=` tokens and argument values. However, it is often
easier to use syn's [`Punctuated`] type for this, which is similar to a `Vec`
except it has a second generic argument for a delimiter that separates list
elements and can be used for parsing through a few associated functions.

[`Punctuated`]: https://docs.rs/syn/latest/syn/punctuated/struct.Punctuated.html

Depending on the task at hand, we might want to create an enum that represents a
single argument to the derive macro, but in this case we can just reuse the
existing logic and simply parse a `GetterMeta` that always has exactly one field
set to `Some`, then use the `merge` function created above to collect them into
a `GetterMeta` with all the arguments for a given field.

```rust
// Before
.try_fold(GetterMeta::default(), |meta, attr| meta.merge(attr.parse_args()?))?;

/// After
use syn::punctuated::Punctuated;

.try_fold(GetterMeta::default(), |meta, attr| {
    let list: Punctuated<GetterMeta, Token![,]> =
        attr.parse_args_with(Punctuated::parse_terminated)?;

    list.into_iter().try_fold(meta, GetterMeta::merge)
})?;
```

## Lookahead

One more thing I want to showcase before concluding this article is lookahead.
In the `GetterMeta` parsing code above, we started off by parsing an identifier
because regardless of whether we are parsing a name or visibility argument, it
starts with `name` or `vis` which can both be parsed to `Ident`.

However, `syn` also provides ways of trying different parsers on the next token
in the input in the form of [`ParseStream::lookahead1`][lookahead1]. This even
applies to our case since what we really want to do is parse either exactly
`name`, or exactly `vis`; not an arbitrary identifier. We can create custom
keyword types for each using [the `custom_keyword` macro][custom_keyword] and
then try parsing each of them:

```rust
mod kw {
    use syn::custom_keyword;

    custom_keyword!(name);
    custom_keyword!(vis);
}

impl Parse for GetterMeta {
    fn parse(input: ParseStream<'_>) -> syn::Result<Self> {
        let lookahead = input.lookahead1();
        if lookahead.peek(kw::name) {
            let _: Token![=] = input.parse()?;
            let name = input.parse()?;

            Ok(Self { name: Some(name), vis: None })
        } else if lookahead.peek(kw::vis) {
            let _: Token![=] = input.parse()?;
            let vis = input.parse()?;

            Ok(Self { name: None, vis: Some(vis) })
        } else {
            // … and we get an appropriate error message for free!
            // See Appendix B for a way to ensure this with tests.
            Err(lookahead.error())
        }
    }
}
```

[lookahead1]: https://docs.rs/syn/1.0/syn/parse/struct.ParseBuffer.html#method.lookahead1
[custom_keyword]: https://docs.rs/syn/1.0/syn/macro.custom_keyword.html

## Enough for today?

That's all of what I can think of as being important to know about parsing
custom syntax with `syn`. Like in the second article of this series, there's an
appendix here with some tangentially related topics that you might find
interesting (so this article does not end here), but I hope you now have an idea
of how to write your own proc-macro parsing code!

Even though it has taken more than half a year from the last article to this
one, I still plan to add at least an article about
<span class="abbrev" title="abstract syntax tree">AST</span> traversal and
possibly one more after that one. See you next time!

## Appendix

### A. Deprecating custom syntax

*re-add support for the literal variation*

### B. Testing errors / warnings
