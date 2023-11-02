# blog.turbo.fish

This repository contains the sources for my blog at <https://blog.turbo.fish/>.

This branch contains a work-in-progress port to my own static site generator,
hinoki. It is not yet publically available.

## Build instructions

```sh
hinoki build
# hinoki also has no dev server yet
(pushd public; python -m http.server)
```
