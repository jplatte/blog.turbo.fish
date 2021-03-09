+++
title = "Procedural Macros: Error handling"
draft = true
+++

This is the third article in my series about procedural macros. The examples
here are based on the `Getters` derive macro from [the previous article][prev].

As the title says, this time I'll explain error handling, specifically how to
use `syn::Error` / `syn::Result` to produce errors that the compiler will emit
as if they originate somewhere in the macro input, rather than pointing at the
macro invocation.

[prev]: /proc-macro-simple-derive/

## A use case

Before we can start adding meaningful spans to parts of the macro input, there
has to be the possibility for errors other than those already caught by the
Rust compiler itself. Luckily, there is a common way in which the input of a
derive macro can be wrong in a way specific to that macro, so I can continue on
with the previous `Getters` example rather than coming up with, and explaining,
a new function-like or attribute proc-macro.

That common possibility for errors is attributes: Many derive macros come with
their own attribute(s) that they then parse, and generally they emit an error
when one such attribute is used incorrectly. For the `Getters` macro there is
one obvious (to me) customization possibility that an attribute would enable:
Renaming. As such, we will add a `getter` field attribute that is used as
`#[getter(name = "foo")]`.

## Registering the attribute

The first thing that has to be done before starting to look for attributes in
the `DeriveInput` is registering the attribute. By default if `rustc` encounters
an unknown attribute, that is an error:

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
//                         ^^^^^^^^^^^^^^^^^^^^ this is new
pub fn getters(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    expand_getters(input).into()
}
```

## Parsing the attribute

Since custom parsing is complex enough to deserve its own article, I'm going to
use `syn::Attribute::parse_meta` here, which is sufficient for the syntax shown
above.

```rust
// note: syn::Ident is a re-export from
use syn::{Attribute, Ident, Meta, NestedMeta};

fn parse_name(attr: Attribute) -> syn::Result<Option<String>> {
    // TODO
}
```

<div class="info">

*note that syn::Result is a type alias*

</div>

## Adjusting the generated code

```rust
let getters = fields
    .into_iter()
    .map(|f| {
        let attrs: Vec<_> =
            f.attrs.iter().filter(|attr| attr.path.is_ident("getter")).collect();

        let name_from_attr = match attrs.len() {
            0 => None,
            1 => get_name_attr(&attrs[0])?,
            _ => {
                let mut error =
                    syn::Error::new_spanned(&attrs[1], "redundant `getter(name)` attribute");
                error.combine(syn::Error::new_spanned(&attrs[0], "note: first one here"));
                return Err(error);
            }
        };

        // if there is no `getter(name)` attribute use the field name like before
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
    .collect::<syn::Result<TokenStream>>()?;
```

For this to work, we also need to wrap the return type of `expand_getters` in
`syn::Result` and update the final expression in the function body:

```rust
pub fn expand_getters(input: DeriveInput) -> syn::Result<TokenStream> {
    // ...

    Ok(quote! {
        // ...
    })
}
```

*update proc-macro entry-point*

## And that's it!

That's all there really is when it comes to proc-macro specific error handling
knowledge. If you want to practice your proc-macro skills but haven't come up
with anything to create or contribute to at this point, have a look at David
Tolnay[^1]'s [proc-macro-workshop].

Next time, I will explain how to parse custom syntax. *todo*

[proc-macro-workshop]: https://github.com/dtolnay/proc-macro-workshop#readme

*link to proc-macro-workshop*
