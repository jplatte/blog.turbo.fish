+++
title = "Advanced Cargo [features] Usage"
date = 2020-10-24
+++

Last year, [in my Rust 2020 blog post][rust-2020], I asked for Cargo crate
features to receive some attention. Fortunately, it seems like I was not the
only one and the big feature resolver rewrite that was required to fix a number
of issues has happened. Unfortunately, the new feature resolve is still
Nightly-only, and it still doesn't fix all of my gripes with Cargo `[features]`.
Recently, I discovered some tricks to work around some of these gripes, so I
thought I should share them here!

[rust-2020]: https://blog.turbo.fish/rust-2020/

## Reusing the name of an optional dependency for a feature

Sometimes, you have an optional dependency, and would really like to activate
a feature or another dependency automatically if that dependency is activated –
exactly like if it was a crate feature. However, you can't just create a feature
of the same name because that would cause a conflict (dependencies and features
share the same namespace¹). Adding a feature of a different name and requiring
your users to activate that would be a breaking change, for something that
should really be straight-forward. What to do?

To answer that, let me first back up a bit:

Since [Cargo 1.31], you can rename dependencies in `Cargo.toml`. Usually, you
would do this to import multiple versions of the same crate, or, most often, to
just use the crate under a different (shorter) name. However, it also means that
referring to it in feature dependencies…

```toml
[features]
my_feature = ["<here>"]
```

is done using the name you chose, rather than the original name. This means you
can reuse the original crate name as the name of a feature[^1]! However, it also
means that you now have to refer to the dependency using a different name in
your code. Unless…

[Cargo 1.31]: https://blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html#cargo-features

### … without actually renaming it

You just rename it back in your crate root! Before renaming was possible in
`Cargo.toml`, you would do it through `extern crate foo as bar;`. Turns out you
can do this to just rename it dependency back if it's renamed in `Cargo.toml`.
In Rust 2018 crates, this will result in both names referring to the same crate,
but that should not be an issue in practice.

To put all this together into a made-up example, let's say you have a crate
`big_crate` that defines a bunch of types, and has an optional dependency on
`serde` to make them (de)serializable for users who need that:

```toml
[dependencies]
serde = { version = "1.0.117", features = ["derive"], optional = true }
# other dependencies
```

Now your crate is rather large, and some of your users are only interested in
a rather small subset of your crate. You decide that it makes sense to have this
subset in an independent crate `small_crate`, and refactor things accordingly,
re-exporting the new crate's contents such that the API of `big_crate` stays the
same. However, some of the types with optional (de)serialization have moved too!
You need to make sure that they continue to implement `Deserialize` and
`Serialize` if `big_crate`s `serde` dependency is enabled. To do that, first
rename `serde` in `Cargo.toml`:

```toml
[dependencies]
serde_cr = { package = "serde", version = "1.0.117", features = ["derive"], optional = true }
small_crate = "0.1.0" # Added during the refactoring
# other dependencies
```

Then add a `serde` feature:

```toml
[features]
serde = [
    "serde_cr", # Enable the serde dependency...
    "small_crate/serde", # ... and small_crate's serde feature
]
```

and finally, undo the renaming from `Cargo.toml` in `big_crate`s `lib.rs`:

```rust
extern crate serde_cr as serde;
```

That's it. Onto the next trick!

## Activating a dependency if a combination of features is active

<small>
    Or activating a feature if a combination of other features is active
</small>

… is unfortunately not possible in the general case. However, there is a
workaround that works well enough for a subset of use cases: Adding a combined
feature. That is, if you have an `f_foo` feature for the `c_foo` crate and an
`f_bar` feature for the `c_bar` crate, and also need to activate the `c_foo_bar`
crate if both `f_foo` and `f_bar` are active, just require your users to use a
new feature, `f_foo_bar`, which automatically activates `f_foo` and `f_bar`, but
also the `c_foo_bar` dependency.

The reason this doesn't work in the general case is that other library crates
(not just applications) might want to use features your crate provides. If one
`rdep_1` activates `f_foo` and `rdep_2` activates `f_bar`, another crate
depending on both `rdep_1` and `rdep_2` would have to know how these work
internally and depend on your crate explicitly without actually using it, just
to activate `f_foo_bar`.

As a practical example, you can have a look at [my PR adding rustls as an
alternative TLS backend in SQLx][sqlx-pr]. There, I added *six* of these
combined features for every possible combination of runtime (`async-std` /
`tokio` / `actix`) and TLS backend (`native-tls` / `rustls`). Some of the
combined features activate crates that are only relevant for that combination
of runtime and TLS backend, for example `tokio` + `native-tls` ⇒
`tokio-native-tls`.

This trick works very well for runtimes and TLS backends in SQLx because there
is no reason for it to expose them independently. Other crates can either select
both a runtime and TLS backend at the same time, or have their own runtime + TLS
backend features that activate the corresponding ones in SQLx.

[sqlx-pr]: https://github.com/launchbadge/sqlx/pull/735

## Good error messages for mutually exclusive feature misuse

Mutually exclusive features are not natively supported in Cargo. Of course it is
easy to make something not compile with a certain feature set, but more often
than not, you want to provide a useful error message if a conflicting feature
set is requested. That part is relatively easy, it takes two lines of code:

```rust
#[cfg(all(feature = "feat1", feature = "feat2"))]
compile_error!("`feat1` and `feat2` may not be used at the same time");
```

However, if your crate would already not compile with these two features
activated at the same time, for example due to duplicate definitions of the
same item, this new error message is not going to replace those[^2] but rather
just appear alongside them. That's not very user friendly!

To fix this, you can carefully set up your `#[cfg]`s such that all of the other
errors go away for all invalid feature sets a user could provide, for example
by using `#[cfg(all(feature = "feat1", not(feature = "feat2")))]` rather than
just `#[cfg(feature = "feat1")]` on the first definition of an item that has
different definitions for the two features. Of course, this quickly becomes a
burden if there is a non-trivial amount of feature-gated code, so I am here to
offer an alternative: Move that compile error into its own crate[^3].

It might seem weird to have a crate whose sole purpose is to emit a compiler
error if a certain set of its features is activated, but this actually neatly
solves the problem: By depending on a crate that contains the feature checks,
you make sure that they run before any of the code in your own crate is
compiled. Just forward all of your own features to this crate, like this:

```toml
[dependencies]
# implementation detail of your crate – should be versioned in lockstep and
# always be used with an exact version requirement.
my_feature_check = "=0.7.2"

[features]
feat1 = ["my_feature_check/feat1"]
feat2 = ["my_feature_check/feat2"]
```

… and move the `compile_error!` into `my_feature_check`.

[^1]: There is an [unstable feature][ns-features] that changes this.

[^2]: This is only true for some kinds of errors, but enough of them for this to
be a problem more often than not.

[^3]: There are other alternatives that work without a separate crate, but this
is the only one I know of that doesn't degrade readability of the actual crate
or IDE functionalities.

[ns-features]: https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#namespaced-features
