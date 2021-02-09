+++
title = "Procedural Macros: Walkthrough of a simple derive macro"
template = "page-lumeo.html"
draft = true
+++

This is the second article in my series on procedural macros. If you don't know
what procedural macros are, I highly recommend reading
[the previous article][basics] first.

As the title says, this will be a walkthrough (detailed explanation) of a simple
derive macro: `Getters`. `#[derive(Getters)]` will generate an accessor function
for every named field of its input struct (it will report an error when used on
other kinds of types). So

```rust
#[derive(Getters)]
struct NewsFeed {
    name: String,
    url: String,
    category: Option<String>,
    tags: Vec<String>,
}
```

will generate

```rust
impl NewsFeed {
    pub fn name(&self) -> &str {
        &self.name
    }

    pub fn url(&self) -> &str {
        &self.url
    }

    pub fn category(&self) -> Option<&str> {
        self.category.as_deref()
    }

    pub fn tags(&self) -> &[String] {
        &self.tags
    }
}
```

To make it easy to play around with the code in the next sections, I've created
a git repository that will contain all of the examples from this blog series
eventually. No need to go there now if you plan to continue reading; there will
be a link to the commits for the changes from this article at the end.

If you just want to read code without explanations though, here is the link you
need: <https://github.com/jplatte/proc-macro-blog-examples>

[basics]: /proc-macro-basics/

## Getting started

First off, we need to create the proc-macro crate that is going to contain our
derive macro. We'll name it `getters` and create it using

```sh
cargo init --lib getters
```

Then we add

```toml
[lib]
proc-macro = true
```

to its `Cargo.toml` and add dependencies on `proc-macro2`, `syn` and `quote`.

<small>

You can do this on the command line with [cargo-edit]:
`cargo add proc-macro2 syn quote`

</small>

As mentioned in the first article, derive macros are simply functions with the
`#[proc_macro_derive(Name)]` attribute, so we add

```rust
use proc_macro::TokenStream;

#[proc_macro_derive(Getters)]
pub fn getters(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

to `lib.rs`. This derive macro can now already be used! It just doesn't do
anything yet.

[cargo-edit]: https://crates.io/crates/cargo-edit

## Parsing the input

*declare macro that parses input, outputs empty token stream*

*panic on non-structs*
*note how errors should often have an explicit span, but not here*

*panic on structs w/o named fields*

## Generating some output

*make sure to include `#[automatically_derived]`*

*Touch on hygiene, peer dependencies*

## Working with generic types

Our initial example produces the expected output, now we're done, right?
Unfortunately, unless we just forbid generic structs as inputs, we aren't.
Let's consider what our current logic will generate for:

```rust
#[derive(Getters)]
struct BorrowedNewsFeed<'a> {
    name: &'a str,
    url: &'a str,
    category: Option<&'a str>,
    tags: Vec<&'a str>,
}
```

It will generate `impl BorrowedNewsFeed { ... }`! That won't work, since we're
not allowed to elide the lifetime in impl blocks, and even if we were could, our
generated methods would mention the lifetime `'a` in their return types. Clearly
we need to introduce the type's generic parameters for the generated `impl` too.

You may assume that handling generics correctly will be a lot of work, and it
can certainly be depending on the macro. However, for basic ones like ours we
can just let `syn` do all of the heavy lifting:

```rust
let (impl_generics, ty_generics, where_clause) = st.generics.split_for_impl();
```

## Next up

*come back to panic and note that next post discusses error handling*
