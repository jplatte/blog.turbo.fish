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

*extend derive example with support for enums with struct variants*

*then with support for `#[getter(name = "foo")]` using `.parse_meta()`*

*link to proc-macro-workshop*

*appendix and/or repo commit for tuple structs / enum tuple variant support?*
