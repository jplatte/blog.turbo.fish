+++
title = "Procedural Macros: Error handling"
+++

This is the third article in my series about procedural macros. The examples
here are based on the `Getters` derive macro from [the previous article][prev].

As the title says, this time I'll explain error handling, specifically how to
use `syn::Error` to produce errors that will be shown by the compiler as
originating somewhere in the macro input, rather than pointing at the macro
invocation.

[prev]: /proc-macro-simple-derive/

## A use case

Before we can start adding meaningful spans to parts of the macro input, there
has to be the possibility for errors other than those already caught by the
Rust compiler itself. Luckily, there is a common way in which the input of a
derive macro can be wrong in a way specific to that macro, so I can continue on
with the previous `Getters` example rather than coming up with, and explaining,
a new function-like or attribute proc-macro.

That common possibility for errors is attributes: Many derive macros come with
their own attribute(s), and generally they emit an error when one such attribute
is used incorrectly. For the `Getters` macro there is one obvious (to me)
customization possibility that an attribute would enable: Renaming. As such, we
will add a `getter` field attribute that is used as `#[getter(name = "foo")]`.

## Registering the attribute

The first thing that has to be done before starting to look for attributes in
the `DeriveInput` is registering the attribute. By default if `rustc` encounters
an unknown attribute, [that is an error][error]:

[error]: https://www.reddit.com/r/rustjerk/comments/n7b1gu

```
error: cannot find attribute `getter` in this scope
  --> src/ratchet/keys.rs:15:7
   |
15 |     #[getter(name = "init_vec")]
   |       ^^^^^^
```

Making that error disappear is as simple as updating the `#[proc_macro_derive]`
attribute on our proc-macro entry point:

```rust
#[proc_macro_derive(Getters, attributes(getter))]
//                           ^^^^^^^^^^^^^^^^^^ this is new
pub fn getters(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    expand_getters(input).into()
}
```

## Parsing the attribute

Since custom parsing is complex enough to deserve its own article, I'm going to
use comma-separated `syn::Meta` parsing here, which is sufficient for the syntax
shown above. The parser function will return `syn::Result<T>`, which is simply a
type alias for `Result<T, syn::Error>`.

```rust
// Note: syn::Ident is a re-export of proc_macro2::Ident
use syn::{punctuated::Punctuated, Attribute, Ident, Meta, Token};

fn get_name_attr(attr: &Attribute) -> syn::Result<Option<Ident>> {
    // Error when the attribute is just `#[getter]`, or `#[getter = something]`.
    // We require a parenthesized list, i.e. `#[getter(something)]`.
    let meta_list = attr.meta.require_list()?;

    // Parse the contents of the list as Meta structs, delimited by commas.
    let nested = meta_list.parse_args_with(Punctuated::<Meta, Token![,]>::parse_terminated)?;
}
```

Luckily detecting whether an attribute is possible without calling any of
`Attribute`s `parse_` methods, so we can detect whether the attribute is for us
before executing this fallible operation.

But I'm getting ahead of myself… First, let's add more to our new function.
Here is what the most common way of constructing a `syn::Error` looks like (for
me at least):

```rust
use syn::Meta;

let meta_list = match meta {
    Meta::List(list) => list,
    // *Almost* equivalent (see syn documentation) to:
    // use syn::spanned::Spanned;
    //   return Err(syn::Error::new(meta.span(), "expected a list-style attribute"))
    _ => return Err(syn::Error::new_spanned(meta, "expected a list-style attribute")),
};
```

As you can see, creating a `syn::Error` is nothing special.

The rest of `get_name_attr` works in much the same way:

```rust
use syn::{Lit, NestedMeta};

let nested = match meta_list.nested.len() {
    // `#[getter()]` without any arguments is a no-op
    0 => return Ok(None),
    1 => &meta_list.nested[0],
    _ => {
        return Err(syn::Error::new_spanned(
            meta_list.nested,
            "currently only a single getter attribute is supported",
        ));
    }
};

let name_value = match nested {
    NestedMeta::Meta(Meta::NameValue(nv)) => nv,
    _ => return Err(syn::Error::new_spanned(nested, "expected `name = \"<value>\"`")),
};

if !name_value.path.is_ident("name") {
    // Could also silently ignore the unexpected attribute by returning `Ok(None)`
    return Err(syn::Error::new_spanned(
        &name_value.path,
        "unsupported getter attribute, expected `name`",
    ));
}

match &name_value.lit {
    Lit::Str(s) => {
        // Parse string contents to `Ident`, reporting an error on the string
        // literal's span if parsing fails
        syn::parse_str(&s.value()).map_err(|e| syn::Error::new_spanned(s, e))
    }
    lit => Err(syn::Error::new_spanned(lit, "expected string literal")),
}
```

## Adjusting the existing codegen

Now we have a new method to parse `#[getter]` attributes, but we aren't using it
yet. We need to update the existing code generation logic to take these
attributes into account, and the first step towards that is making the
`expand_getters` function fallible as well.

If it's been some time since you read the last article, here is its signature
again (you can also review the entire definition [here](simple-derive-v1)):

[simple-derive-v1]: https://github.com/jplatte/proc-macro-blog-examples/blob/simple-derive-v1/derive_getters/src/getters.rs

```rust
pub fn expand_getters(input: DeriveInput) -> TokenStream {
```

Which now becomes

```rust
pub fn expand_getters(input: DeriveInput) -> syn::Result<TokenStream> {
```

The new `expand_getters` implementation is a bit longer, but still manageable:

```rust
// Same as before
let fields = match input.data {
    Data::Struct(DataStruct { fields: Fields::Named(fields), .. }) => fields.named,
    _ => panic!("this derive macro only works on structs with named fields"),
};

// All the new logic comes in here
let getters = fields
    .into_iter()
    .map(|f| {
        // Collect getter attributes
        let attrs: Vec<_> =
            // This `.filter` is how we make sure to ignore builtin attributes, or
            // ones meant for consumption by different proc-macros.
            f.attrs.iter().filter(|attr| attr.path.is_ident("getter")).collect();

        let name_from_attr = match attrs.len() {
            0 => None,
            1 => get_name_attr(attrs[0])?,
            // Since `#[getter(name = ...)]` is the only available `getter` attribute,
            // we can just assume any attribute with `path.is_ident("getter")` is a
            // `getter(name)` attribute.
            //
            // Thus, if there is two `getter` attributes, there is a redundancy
            // which we should report as an error.
            //
            // On nightly, you could also choose to report a warning and just use one
            // of the attributes, but emitting a warning from a proc-macro is not
            // stable at the time of writing.
            _ => {
                let mut error = syn::Error::new_spanned(
                    attrs[1],
                    "redundant `getter(name)` attribute",
                );
                // `syn::Error::combine` can be used to create an error that spans
                // multiple independent parts of the macro input.
                error.combine(
                    syn::Error::new_spanned(attrs[0], "note: first one here"),
                );
                return Err(error);
            }
        };

        // If there is no `getter(name)` attribute, use the field name like before
        let method_name =
            name_from_attr.unwrap_or_else(|| f.ident.clone().expect("a named field"));
        let field_name = f.ident;
        let field_ty = f.ty;

        Ok(quote! {
            pub fn #method_name(&self) -> &#field_ty {
                &self.#field_name
            }
        })
    })
    // Since `TokenStream` implements `FromIterator<TokenStream>`, concatenating an
    // iterator of token streams without a separator can be using `.collect()` in
    // addition to `quote! { #(#iter)* }`. Through std's `FromIterator` impl for
    // `Result`, we get short-circuiting on errors on top.
    .collect::<syn::Result<TokenStream>>()?;

// Like before
let st_name = input.ident;
let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

// Resulting TokenStream wrapped in Ok
Ok(quote! {
    #[automatically_derived]
    impl #impl_generics #st_name #ty_generics #where_clause {
        // Previously: #(#getters)*
        //
        // Now we don't need that anymore since we already
        // collected the getters into a TokenStream above
        #getters
    }
})
```

<div class="info">

If this is the first time you have seen `.collect::<Result<_, _>>`, you can find the documentation
for the trait implementation that makes it possible [here].

[here]: https://doc.rust-lang.org/std/result/enum.Result.html#impl-FromIterator%3CResult%3CA%2C%20E%3E%3E-for-Result%3CV%2C%20E%3E

</div>

## Passing a `syn::Error` to the compiler

One final piece of the puzzle is missing: How does `syn::Error` become a
compiler error? We can't update our proc-macro entry point to return
`syn::Result`, that would result in an error because proc-macro entry points are
required to return just a `TokenStream`.

However, the solution is almost as easy and you might already have seen it if
you had a look at `syn::Error`s [documentation][error-docs]:

[error-docs]: https://docs.rs/syn/2.0/syn/parse/struct.Error.html

```rust
// Previously, with expand_getters returning proc_macro2::TokenStream
expand_getters(input).into()
// Now, returning syn::Result<proc_macro2::TokenStream>
expand_getters(input).unwrap_or_else(syn::Error::into_compile_error).into()
```

What this does under the hood is actually kind of weird: It produces a
TokenStream like

```rust
quote! { compile_error!("[user-provided error message]"); }
```

but with the span being the one given when constructing the `syn::Error`. As
weird as it is, that's simply the only way to raise a custom compiler error on
stable (as of the time of writing).

<div class="info">

If you haven't seen `compile_error!` before, it's a [builtin macro][comperr].

[comperr]: https://doc.rust-lang.org/std/macro.compile_error.html

</div>

## And that's it!

That's all there really is when it comes to proc-macro specific error handling
knowledge. Like last time, you can review the changes from this blog post in the
accompanying repo:

* [Complete code](https://github.com/jplatte/proc-macro-blog-examples/tree/error-handling-v1/derive_getters)
* [Individual commits](https://github.com/jplatte/proc-macro-blog-examples/compare/simple-derive-v1...error-handling-v1)

If you want to practice your proc-macro skills but haven't come up with anything
to create or contribute to at this point, I recommend having a look at David
Tolnay's [proc-macro-workshop].

Next time, I will explain how to parse custom syntax, which can be useful for
derive macros when you want to go beyond what `syn::Meta` allows, and is crucial
for many attribute macros as well as the majority of function-like proc-macros.
Stay tuned!

[proc-macro-workshop]: https://github.com/dtolnay/proc-macro-workshop#readme
