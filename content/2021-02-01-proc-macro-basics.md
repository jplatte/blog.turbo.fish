+++
title = "Procedural Macros: The Basics"
template = "page-lumeo.html"
draft = true
+++

Procedural macros (often shortened to 'proc-macros') are a unique Rust feature
that allow creating custom derives, attributes and function-like macros in
regular Rust code that is run at compile time. And although they are the only
way of writing custom attributes and derives, vastly more powerful and even
easier to read than (large) declarative macros, there is very little learning
material out there for proc-macros.

That's why I'm here, writing my own blog series (as you could probably tell
from the title) on the matter. Currently, I plan to write 5 more posts after
this one, starting with a very basic derive example that should appear within a
weel or two, continuing with a discussion on proc-macro error handling and then
moving to topics only applicable to smaller subsets of proc-macros.

## Anatomy of a procedural macro

Procedural macros are functions with a `#[proc_macro]`,
`#[proc_macro_derive(Name)]` or `#[proc_macro_attribute]` attribute. They have
one (function-like and derive macros) or two (attribute macros)
`proc_macro::TokenStream` arguments and also return a `proc_macro::TokenStream`.

> You can sort of think of procedural macros as functions from an AST to another
> AST.
>
> — [The Rust Reference][ref]

They must be public (`pub fn ...`) and live in a crate of the type `proc-macro`.
When using Cargo (and who doesn't?) this crate type is declared
in the `lib` section of `Cargo.toml`:

```toml
[lib]
proc-macro = true
```

A proc-macro crate can *only* export proc-macros, no regular functions, types,
modules or `macro_rules!` macros. It can of course define other items, but it
cannot export them for use in other crates.

[ref]: https://doc.rust-lang.org/reference/procedural-macros.html

## The crates involved

Every proc-macro crate can access the builtin [`proc_macro` crate][proc_macro].
There is not much to say about it; the only type from it that a typical
proc-macro interacts with is `proc_macro::TokenStream`.

All of the following articles are going to be about proc-macros that parse their
input with [syn] and produce their output with [quote]. These are not mandatory
for proc-macros, but they are used by the vast majority of them.

syn focuses on parsing Rust code into its own [AST], but also has (optional)
support for parsing custom syntax, which can be very useful when parsing
attributes, function-like macro input or custom syntax embedded into Rust syntax
(most commonly when writing attribute macros).

quote has a tiny API surface of which one item sees the vast majority of `use`s:
`quote!`. `quote!` is a function-like macro that turns its input into a
`proc_macro2::TokenStream` while substituting placeholders.  
If you have read or written a `macro_rules!` macro before, I'll already let you
know this: `quote! { #(#key: #value),* }` is the proc-macro equivalent to
`$($key: $value),*)`.
If you haven't, don't mind the symbol soup above: We'll get to that in the next
article.

<small>

You might be wondering about the `2` in `proc_macro2` above. It's not a typo,
syn and quote hardly interact with the types from the `proc_macro` crate
directly, instead using the wrapper crate [proc-macro2]. This crate only exists
because the builtin crate is limited to procedural macro compilation contexts.
For more details, see the linked documentation.

</small>

[proc_macro]: https://doc.rust-lang.org/proc_macro/
[syn]: https://docs.rs/syn/1.0
[quote]: https://docs.rs/quote/1.0
[proc-macro2]: https://docs.rs/proc-macro2/1.0
[AST]: https://en.wikipedia.org/wiki/Abstract_syntax_tree

## Derive macros vs. other procedural macros

*derives add code, other macros rewrite code*

## A note on macro expansion

*all macros, including procedural ones, run before type checking or resolution!*
