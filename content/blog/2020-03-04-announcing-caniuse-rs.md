+++
title = "Announcing caniuse.rs"
+++
Hello, dear reader! Today, I am excited to announce

## [caniuse.rs](https://caniuse.rs)

![screenshot of caniuse.rs](/img/caniuse_rs.png)

A website I created, which, in contrast to [turbo.fish](https://turbo.fish/),
has a practical use: It allows you to quickly find out which version of Rust
stabilized a certain feature[^1]. It's like `site:blog.rust-lang.org` in your
search engine, but better!

## What for?

You might never have asked yourself "what version of Rust stabilized X". Then
caniuse.rs is probably not going to be that useful for you. But I have been in
that situation quite a few times, and I'm sure other have been too. To list a
few (fictional) scenarios:

* You just noticed that a crate you are working on only uses `lazy_static!` in
  one place, for a thing that you think should really be a `const` instead.
  You're wondering if the functions used to initialize that value are `const fn`
  by now.
* You see a crate use `Struct { redundant: redundant }` field initialization. It
  has a relatively old MSRV, but is it too old for field init shorthands?
* You remember reading an RFC about exhaustive integer pattern matching and are
  now wondering whether that `_ => unreachable!()` arm can finally be removed
  from your `match`.
* It is the year 2025. The last time you worked with Rust was in 2020. You
  wonder whether the never type (`!`) is stable by now.

## That's it!

I don't think there's a point in writing more about the site here when you can
just go over there and try it out yourself.

[» caniuse.rs «](https://caniuse.rs/)

[^1]: With "feature", I don't mean just `#![feature(…)]` features. I also mean
all the things that continually got added to `std`. Functions that were made
`const`. Macros that used to only be callable with arguments that were changed
to allow argument-less calls or vice versa. There's even been a few
language-level changes that were implemented without receiving a feature gate.
