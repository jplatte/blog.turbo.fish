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
}
```

will generate

```rust
impl NewsFeed {
    pub fn name(&self) -> &String {
        &self.name
    }

    pub fn url(&self) -> &String {
        &self.url
    }

    pub fn category(&self) -> &Option<String> {
        &self.category
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
derive macro. We'll name it `derive_getters` and create it using

```sh
cargo init --lib derive_getters
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
    Data::Struct(DataStruct { fields: Fields::Named(fields), .. }) => fields.named,
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

So far so good, but that `impl` block should actually contain some methods.
Remember that we already extracted the named fields of the input into a local
variable named `fields` earlier? This fields has the type
`syn::punctuated::Punctuated<syn::Field, syn::token::Comma>`, more commonly
written `Punctuated<Field, Token![,]>` (the `Token` macro is also part of syn).
Really all we need from that right now is an iterator of the actual `Field`s to
be able to generate some code for each of them, and of course `Punctuated`
implements `IntoIterator` so we can do this:

```rust
let getters = fields.into_iter().map(|f| {
    // Remember: Interpolation only works for variables, not arbitrary expressions.
    // That's why we need to move these fields into local variables first
    // (borrowing would also work though).
    let field_name = f.ident;
    let field_ty = f.ty;

    quote! {
        pub fn #field_name(&self) -> &#field_ty {
            &self.#field_name
        }
    }
});
```

Which is interpolated into the output slightly differently:

```rust
#[automatically_derived]
impl #st_name {
    #(#getters)*
}
```

The reason we have to enclose `#getters` in `#()*` is that it's not a single
thing: It's an iterator. `#()*` tells `quote!` to interpolate the inner tokens
once per item in all interpolated iterators. In this very simple case, we could
also have called `.collect::<TokenStream>()` on the iterator first and
interpolated the result as just `#getters`, but this repetition syntax is a lot
more powerful than that (see [Appendix A]).

[Appendix A]: #a-interpolation-repetition

## If something goes wrong

We now have a working proc-macro! But what if we had made a mistake in our
generated code? Imagine forgetting the `#` in `&self.#field_name` of the
generated getter methods. Then all getter methods would try to access a field
named literally `field_name`, and we would get errors to that. And they'd point
at where our derive macro was invoked:

```
error[E0609]: no field `field_name` on type `&NewsFeed`
 --> getters/tests/news_feed.rs:3:10
  |
3 | #[derive(Getters)]
  |          ^^^^^^^ unknown field
  |
  = note: available fields are: `name`, `url`, `category`
  = note: this error originates in a derive macro (in Nightly builds, run with -Z macro-backtrace for more info)
```

That's… Not very helpful. And while we might be able to figure things out in
this case, imagine your macro code being 500 lines of code, rather than the
current 34. Clearly, we need a way of seeing the generated code. This is where
[cargo-expand] comes in. You invoke it like any cargo command that invoked the
compiler (e.g. `check` or `build`), but additionally can specify a module path
to filter out of the output, similar to the last argument of `cargo test`:

* `cargo expand foo::bar` – show the generated code for the module `foo::bar`
* `cargo expand --test news_feed` – show the generated code for the `news_feed`
  test

In the case of the forgotten `#`, expanding the news feed test case from the
beginning will make the problem very clear (comments are mine, of course):

```rust
// Some things you will always see in the output of `cargo expand`.
// Usually you can just remove these if you want to compile the expanded output
// separately to find an issue in a large body of generated code.
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std;
// What you wrote, with macros expanded:
use getters::Getters;
// Omitted: Definition of NewsFeed
#[automatically_derived]
impl NewsFeed {
    pub fn name(&self) -> &String {
        &self.field_name
    }
    // Omitted: Other getters
}
// Omitted: Generated main function (because this is a test module)
```

[cargo-expand]: https://github.com/dtolnay/cargo-expand#readme

## Working with generic types

Our initial example produces the expected output, now we're done, right?
Unfortunately, unless we just forbid generic structs as inputs, we aren't.
Let's consider what our current logic will generate for:

```rust
#[derive(Getters)]
struct NewsFeedRef<'a> {
    name: &'a str,
    url: &'a str,
    category: Option<&'a str>,
}
```

It will generate `impl NewsFeedRef { ... }`! That won't work, since we're not
allowed to elide the lifetime in impl blocks, and even if we were could, our
generated methods would mention the lifetime `'a` in their return types. Clearly
we need to introduce the type's generic parameters for the generated `impl` too.

You may assume that handling generics correctly will be a lot of work, and it
can certainly be depending on the macro. However, for basic ones like ours we
can just let `syn` do all of the heavy lifting:

```rust
let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

quote! {
    #[automatically_derived]
    impl #impl_generics #st_name #ty_generics #where_clause {
        #(#getters)
    }
}
```

And that's it! Our derive now works on generic types, including ones with type
or const generics and complex where clauses. There's one thing this example
makes even clearer than the previous ones though: Simply prepending a `&` to a
field's type is not a very good solution to finding an appropriate get method
return type. See [Appendix B] for a look at how we could improve the accessors'
return types.

[Appendix B]: #b-improving-the-accessors-return-types

## Conclusion

At this point, you should have an understanding of how to create a simple
procedural macro! You can look at the macro crate I showed in small bits above
in its entirety [here][complete-macro] or review the changes from each section
separately by going through the [corresponding commits][article-commits].

There are still many things that I haven't covered though. The most important
one in my opinion is error handling, specifically having errors reported as
originating from a part of the input; that's what I'll be explaining in the next
article. You will have to wait a little bit longer for that one, since I had a
head start for this one (I had written parts already when publishing the first
article), which is not the case for the next one.

[complete-macro]: https://github.com/jplatte/proc-macro-blog-examples/tree/simple-derive-v1/derive_getters
[article-commits]: https://github.com/jplatte/proc-macro-blog-examples/compare/init...simple-derive-v1

## Appendix

I want to show some things that are not crucial to the article, and if included
above would make the main article a rather long read. They're also too long for
footnotes, so I created this section instead.

### A. Interpolation repetition

As mentioned above, interpolation repetition is a lot more powerful than simply
expanding an iterator. Here is an example:

```rust
quote! {
    struct #name {
        #(pub #fields: #tys),*
    }
}
```

`fields` and `tys` are both iterators or lists here and will be iterated in
lockstep to produce a list of struct fields, separated with commas (that is,
there won't be a comma after the last field generated by the interpolation,
which would be there if the comma was inside the parentheses).

There is also a short section on interpolation
[in `quote!`s documentation][quote-docs] if you want to learn more about
interpolation, for example the supported types.

[quote-docs]: https://docs.rs/quote/1.0/quote/macro.quote.html#interpolation

### B. Improving the accessors' return types

Since what the macro does so far is still very basic, I wanted to show a small
thing that makes it a tiny bit more sophisticated: Customizing the output type
for the generated getters.

The first very simple "specialization" that I touched on with the
`NewsFeedRef` example is references. These can be detected reliably:

```rust
use syn::{Type, TypeReference};

let getters = fields.into_iter().map(|f| {
    let field_name = f.ident;
    let return_ty = match f.ty {
        // shared references can simply by copied
        Type::Reference(r @ TypeReference { mutability: None, .. }) => quote! { #r },
        // fallback to adding a reference
        ty => quote! { &#ty },
    };

    quote! {
        pub fn #field_name(&self) -> #return_ty {
            &self.#field_name
        }
    }
});
```

And due to auto-deref, we don't even need to adjust the function body. This is
more luck than anything though, and we'll need to adjust the function body too
if we want to add more specializations, for example for `Option<_>` and
`String`:

```rust
use syn::{Path, TypePath};

let (return_ty, body) = match f.ty {
    Type::Reference(r @ TypeReference { mutability: None, .. }) => {
        (quote! { #r }, quote! { self.#field_name })
    }
    Type::Path(TypePath { path, .. }) if path.is_ident("String") => {
        (quote! { &::core::primitive::str }, quote! { &self.#field_name })
    }
    Type::Path(ty @ TypePath { .. }) => match option_inner_type(&ty.path) {
        // Option<String> => Option<&str> (.as_deref())
        Some(Type::Path(TypePath { path, .. })) if path.is_ident("String") => (
            quote! { ::std::option::Option<&::core::primitive::str> },
            quote! { self.#field_name.as_deref() },
        ),
        // Option<T> => Option<&T> (.as_ref())
        Some(inner_ty) => (
            quote! { ::std::option::Option<&#inner_ty> },
            quote! { self.#field_name.as_ref() },
        ),
        None => (quote! { &#ty }, quote! { &self.#field_name }),
    },
    ty => (quote! { &#ty }, quote! { &self.#field_name }),
};

quote! {
    pub fn #field_name(&self) -> #return_ty {
        #body
    }
}
```

with the helper function `option_inner_type` being defined as

```rust
use syn::{GenericArgument, PathArguments};

fn option_inner_type(path: &Path) -> Option<&Type> {
    if path.leading_colon.is_some() {
        return None;
    }

    if path.segments.len() != 1 || path.segments[0].ident != "Option" {
        return None;
    }

    let ab = match &path.segments[0].arguments {
        PathArguments::AngleBracketed(ab) => ab,
        _ => return None,
    };

    if ab.args.len() != 1 {
        return None;
    }

    match &ab.args[0] {
        GenericArgument::Type(t) => Some(t),
        _ => None,
    }
}
```

As you can see, things can get more complicated quickly even though we barely
special-cased any types, and maybe you also noticed that we're currently only
applying the special cases to `String`s and `Option`s that are referred to as
exactly that (i.e., qualifying `Option` as `std::option::Option` will bypass the
special case).

<div class="info">

Note that, as mentioned [last time], it is impossible to apply these kinds of
special cases correctly under all circumstances. Users could always shadow
`std`s `Option` or `String` types, or even things like `bool` / `str` / `u8`.

When you want to explicitly use one of the `std` / `core` types, you can do so
with absolute paths, but knowing what type names from the input resolve to is
simple impossible.

</div>

I hope this extended example was a good showcase for what to expect from reading
or writing real-world proc-macro code. In the end it's still all regular Rust
code that inspects a few data structures with the only "magic" being inside
`quote!`.

The full code after these changes can be found [here][simple-derive-ext].

[last time]: ../proc-macro-basics/#a-note-on-macro-expansion
[simple-derive-ext]: https://github.com/jplatte/proc-macro-blog-examples/tree/simple-derive-ext-v1/derive_getters

[^1]: Instead of `let input = parse_macro_input!(input as DeriveInput);`, we could also have written `let input: DeriveInput = parse_macro_input!(input);`. This still seems less common, so I went with the former.

[^2]: A derive macro's input failing to parse as `syn::DeriveInput` is highly unlikely, but possible: Either simply because of a bug in syn, or because the version of syn that's being used is too old to be aware of newer syntax features.
