+++
title = "TBD"
date = 2020-10-29
draft = true
+++

# Q: How do you make a proc-macro find a peer dependency through a rename?

A: [proc-macro-crate](https://crates.io/crates/proc-macro-crate).

## Q: What is a peer dependency?

A: I borrowed this term from NPM. It describes a dependency that has to also be
present in order for the dependency you're actually interested in to work. For
example, you need to have a dependency on `serde` in order to use
`serde_derive`.

# Q: How do you make a derive proc-macro find a peer dependency through a re-export?

A: You don't.

# Q: How do you make an attribute proc-macro find a peer dependency through a re-export?

A: You don't.

# Q: How do you make a function-like proc-macro find a peer dependency through a re-export?

A. You don't. However…

If the peer dependency is actually more of a parent crate, and the only way
users would want to use this macro, …
