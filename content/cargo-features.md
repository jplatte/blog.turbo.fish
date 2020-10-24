+++
title = "Advanced Cargo `[features]` Usage"
date = 2020-10-24
draft = true
+++

Last year, [in my Rust 2020 blog post][rust-2020], I asked for Cargo crate
features to receive some attention. Fortunately, it seems like I was not the
only one and the big feature resolver rewrite that was required to fix a number
of issues has happened. Unfortunately, the new feature resolve is still
Nightly-only, and it still doesn't fix all of my gripes with Cargo `[features]`.
Recently, I discovered some tricks to work around some of these gripes, so I
thought I should share them here!

## Reusing the name of an optional dependency for a feature

<!-- Just rename the dependency in `Cargo.toml`! -->
<!-- Disclaimer: Minimum Supported Cargo Version: X.Y.0 -->

### … without actually renaming it

<!-- Rename it back using `extern crate`! -->

## Activating a dependency if a combination of features is active

<small>
    Or activating a feature if a combination of other features is active
</small>

… is unfortunately not possible in the general case. However, there is a
workaround that works well enough for a subset of use cases: Adding a combined
feature. That is, if you have an `f-foo` feature for the `c-foo` crate and an
`f-bar` feature for the `c-bar` crate, and also want to activate the `c-foo-bar`
crate if both `f-foo` and `f-bar` are active, just require your users to use a
new feature, `f-foo-bar`, which automatically activates `f-foo` and `f-bar`, but
also the `c-foo-bar` dependency.

<!-- doesn't work if f-foo and f-bar activated from different reverse deps -->
<!-- good for mutually exclusive features such as TLS backend & async runtime -->
<!-- requires the same thing in reverse deps if they want to re-export feats -->
<!--
    if f-foo + f-bar *requires* c-foo-bar, can use

    ```
    #[cfg(all(feature = "f-foo", feature = "f-bar", not(feature = "f-foo-bar")))]
    compile_error!("...");
    ```

    to make sure 
-->

## Mutually Exclusive Features

<!-- ## Avoiding `#[cfg]` Hell with Mutually Exclusive Features -->

<!-- … and `compile_error!`s -->
<!--
    may use sub-crate to ensure mutual exclusiveness without spamming unrelated
    compiler errors if violated, or entering `#[cfg]` hell
-->
