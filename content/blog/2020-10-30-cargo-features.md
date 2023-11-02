+++
title = "Advanced Cargo [features] Usage"
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

Sometimes, you have an optional dependency, and would like to activate a feature
or another dependency automatically if that dependency is activated – exactly
like if it was a crate feature. However, you can't just create a feature of the
same name because that would cause a conflict (dependencies and features share
the same namespace[^1]). Adding a feature of a different name and requiring your
users to activate that would be a breaking change, for something that should
really be straight-forward. What to do?

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
can reuse the original crate name as the name of a feature! However, it also
means that you now have to refer to the dependency using a different name in
your code. Unless…

[Cargo 1.31]: https://blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html#cargo-features

### … without actually renaming it

You just rename it back in your crate root! Before renaming was possible in
`Cargo.toml`, you would do it through `extern crate foo as bar;`. Turns out you
can do this to just undo a renaming from `Cargo.toml`. In Rust 2018 crates, this
will result in both names referring to the same crate, but that should not be an
issue in practice.

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
    or activating a feature if a combination of other features is active
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
set is requested. This is usually solved through a conditional `compile_error!`:

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
offer an alternative: Move the feature set check into a [build script][]:

```rust
use std::{env, process};

fn main() {
    let feat1_active = env::var_os("CARGO_FEATURE_FEAT1").is_some();
    let feat2_active = env::var_os("CARGO_FEATURE_FEAT2").is_some();

    if feat1_active && feat2_active {
        eprintln!("error: The f1 and f2 features can't be activated at the same time.");
        process::exit(1);
    }
}
```

*Note: An earlier version of this post advocated using `compile_error!`, but in
a separate crate that the main crate forwards all its features to. That also
works, but is more work and as far as I know has no advantages over the build
script approach that I came up with shortly after.*

[build script]: https://doc.rust-lang.org/cargo/reference/build-scripts.html

[^1]: There is an [unstable feature][ns-features] that changes this.

[^2]: This is only true for some kinds of errors, but enough of them for this to
be a problem more often than not.

[ns-features]: https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#namespaced-features
