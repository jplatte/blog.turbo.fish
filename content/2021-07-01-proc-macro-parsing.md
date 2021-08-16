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

so it's easier to follow along if you're read them already, but should be
understandable on its own if you are already somewhat familiar with writing
proc-macros using `syn` & `quote`.

## Preparation

*syn parsing feature enabled by default, full feature sometimes needed*

## The plan

In the last article, we added a custom attribute for our derive macro that uses
the syntax `#[getter(name = "foo")]`. Now we've decided it should really be
`#[getter(name = foo)]`, *TODO*

*https://caniuse.rs/features/unrestricted_attribute_tokens*

<div class="info">

Note that custom attributes (either registered by derive macros, or as
standalone attribute proc-macros) are not the only use case for parsing custom
syntax. Perhaps more importantly, you can have your own
<span class="abbrev" title="domain-specific languages">DSLs</span> within
function-like macros, *TODO*

*https://yew.rs/concepts/html*

</div>

## TODO

*update the syntax from `#[getter(name = "foo")]` to `#[getter(name = foo)]`*

*deprecation here or in tips-and-tricks?*

*leave keyword idents to tips-and-tricks?*