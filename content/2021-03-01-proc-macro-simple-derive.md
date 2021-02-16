+++
title = "Procedural Macros: A simple derive macro"
draft = true
+++

This is the second article in my series on procedural macros. If you don't know
what procedural macros are, I highly recommend reading
[the previous article][basics] first.

This will be a walkthrough (detailed explanation) of a simple derive macro:
`Getters`. `#[derive(Getters)]` will generate an accessor function for every
named field of its input struct (it will report an error when used on other
kinds of types). So

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

Most procedural macros start with an invocation of `syn::parse_macro_input!()`.
We will do the same thing here[^1]:

```rust
use syn::{parse_macro_input, DeriveInput};

let input = parse_macro_input!(input as DeriveInput);
```

This will get us an instance of `DeriveInput`, or report an error back to the
compiler if parsing the `TokenStream` as a `DeriveInput` fails[^2].

All further inspection of the input as well as the generation of the output is
usually done in a sub-module rather than in `lib.rs`, and uses the types from
`proc_macro2` instead of `proc_macro`:

```rust
// lib.rs
mod getters;
use getters::expand_getters;

// getters.rs
use proc_macro2::TokenStream;
use syn::DeriveInput;

pub fn expand_getters(input: DeriveInput) -> TokenStream {
    TokenStream::new()
}
```

Since this is a different `TokenStream` type, we need to add an extra conversion
when using `expand_getters` from the procedural macro function. Here is the
entire, updated definition:

```rust
#[proc_macro_derive(Getters)]
pub fn getters(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    expand_getters(input).into()
}
```

Now before we go to generate output, we need a way of accessing the fields of
our input struct, if it is a struct with named fields. Of course `DeriveInput`
can also contain an enum, unit struct or tuple struct. Figuring out what's
inside is a simple manner of `match`ing:

```rust
use syn::{Data, DataStruct, Fields};

let fields = match input.data {
    Data::Struct(DataStruct { fields: Fields::Named(fields), .. }) => fields,
    _ => panic!("this derive macro only works on structs with named fields"),
};
```

Here, we `panic!()` if the input is not what we expect it to be. Very often,
panicking in procedural macros is not a good idea since it will generate a
compiler error pointing at the usage of the derive macro, rather than pointing
at relevant parts of the macro input. However, in this case there isn't really
a more specific error location than the macro itself. If the input is an `enum`,
we could point at the `enum` token itself, but it seems questionable whether
that would be much better.

If you want to learn about the other fields of `DeriveInput` and `DataStruct`,
the other variants of `Data` and `Fields` and such, you can find all that in
[syn's documentation](https://docs.rs/syn/1.0).

## Generating some output

To generate output, we will use the `quote` crates `quote!` macro.

```rust
use quote::quote;
```

It is usually invoked with curly braces. Inside, you can write arbitary Rust
code, which won't be compiled as part of the proc-macro itself, but instead is
returned as a `TokenStream`. That is, we want to replace the
`TokenStream::new()` at the end of `expand_getters`:

```diff
-TokenStream::new()
+quote! {
+    impl NewsFeed {}
+}
```

This will work for our first example, but of course we want to use the name of
whatever struct our derive was used on. This is where some of the extra symbol
soup from the previous article comes in: interpolation. `quote!` can interpolate
the values of local variables into the output. You tell it to do that by
prefixing an identifier with a hash sign (`#`):

```rust
let st_name = input.ident;

quote! {
    // It's a good practice to use this attribute on macro-generated impl blocks.
    #[automatically_derived]
    impl #st_name {}
}
```

*Touch on hygiene, peer dependencies*

## If something goes wrong

*mention cargo-expand*

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

[^1]: Instead of `let input = parse_macro_input!(input as DeriveInput);`, we could also have written `let input: DeriveInput = parse_macro_input!(input);`. This still seems less common, so I went with the former.

[^2]: A derive macro's input failing to parse as `syn::DeriveInput` is highly unlikely, but possible: Either simply because of a bug in syn, or because the version of syn that's being used is too old to be aware of newer syntax features.
