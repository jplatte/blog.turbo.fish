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

*since custom parsing deserves its own article*

*use `.parse_meta()`*

*use Ident::new with correct input span and note how it already provides good error msg for non-ident string*

<div class="info">

*note that we could have used `format_ident!` too*

</div>

*todo!("error case")*

## The error case

And now to what you've all been waiting for: The error case. But we can't fill
in that `todo!()` just yet. First, we have to do some refactorings.

*update expand_getters to return syn::Result*

<div class="info">

*note that syn::Result is a type alias*

</div>

*update proc-macro entry-point*

## And that's it!

That's all there really is when it comes to proc-macro specific error handling
knowledge. If you want to practice your proc-macro skills but haven't come up
with anything to create or contribute to at this point, have a look at David
Tolnay[^1]'s [proc-macro-workshop].

Next time, I will explain how to parse custom syntax. *todo*

[proc-macro-workshop]: https://github.com/dtolnay/proc-macro-workshop#readme

*link to proc-macro-workshop*
