+++
title = "Procedural Macros: Tips and Tricks"
draft = true
+++

This is the fifth and last article in my series about procedural macros. I had
planned for two more articles, but decided to merge the ideas for both into one.

If you don't know much about procedural macros yet, this article is probably not
for you. I recommend starting at
[the beginning of the series](/proc-macro-basics/) in that case. This article
just highlights some APIs and patterns that you might find useful when writing
non-trivial proc-macros.

Like the previous three articles, this one is also focused on macros that parse
their input using [`syn`](https://docs.rs/syn/) (though whether / how output is
generated is mostly irrelevant for the below). If you are writing reading this
in a future where `syn` has been supplanted, I'm afraid this article won't do
much good!

## Identifiers

*Ident::parse_any, format_ident!*

## Advanced error handling

*Use fine-grained errors if you want the best user experience*

## How to search (and replace) things

*syn::visit*

## How to handle peer dependencies

*proc_macro_crate?*
