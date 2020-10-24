+++
title = "Splitting crates for fun and profit"
date = 2020-06-20
draft = true
+++

*TL;DR: some of the `url` crate's functionality has been split into a separate crate,
`form_urlencoded`. `serde_urlencoded` now depends on this crate instead of `url`, cutting its
dependency graph in half.*
