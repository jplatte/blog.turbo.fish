+++
title = "Procedural Macros: Parsing custom syntax"
draft = true
+++

This is the fourth article in my series about procedural macros. The examples
here are based on the ones from [the previous article][prev].

In this article, I will explain how you can use `syn` to parse things that are
not Rust code. This is required for a variety of use cases for proc-macros:

* Attributes, either standalone or as part of a derive macro: `syn` only allows
  a limited syntax for attributes if you don't use custom parsing.
* Function-like macros: ________

[prev]: /proc-macro-error-handling/

## ___First heading___

*syn parsing feature enabled by default, full feature sometimes needed*

*update example from last article to use custom parsing for attributes*

*update the syntax from `#[getter(name = "foo")]` to `#[getter(name = foo)]`*

*deprecation here or in tips-and-tricks?*

*leave keyword idents to tips-and-tricks?*
