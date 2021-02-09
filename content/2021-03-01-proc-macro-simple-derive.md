+++
title = "Procedural Macros: Walkthrough of a simple derive macro"
template = "page-lumeo.html"
draft = true
+++

This is the second article in my series on procedural macros. If you don't know
what procedural macros are, I highly recommend reading
[the previous article][basics] first.

As the title says, this will be a walkthrough (detailed explanation) of a simple
derive macro: `Getters`. `#[derive(Getters)]` will generate an accessor function
for every named field of its input struct (it will report an error when used on
other kinds of types). So

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

[basics]: /proc-macro-basics/

## Parsing the input

*declare macro that parses input, outputs empty token stream*

*panic on non-structs*
*note how errors should often have an explicit span, but not here*

*panic on structs w/o named fields*

## Generating some output

*make sure to include `#[automatically_derived]`*

*Touch on hygiene, peer dependencies*

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
