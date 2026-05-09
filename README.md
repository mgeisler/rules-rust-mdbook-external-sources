# rules_rust_mdbook External Repository Path Bug

This repository contains a minimal reproduction of a path resolution bug in
`rules_rust_mdbook`.

## The Problem

When any part of an `mdbook` target (either the `book.toml` or the source files)
comes from an **external repository**, the build fails with a "No such file or
directory" error. This is because the rule's staging logic and its invocation
logic handle repository boundaries inconsistently.

### Root Cause Analysis

The failure stems from a mismatch between the Starlark rule definition and the
Rust `process_wrapper` regarding how external repository paths are resolved.

1. **Staging (`process_wrapper.rs`):** The `MdBookBuild` action uses a Rust
   wrapper to stage files into a shadow work directory (`<target>_`). This
   wrapper receives an input manifest where destinations are derived from
   `file.short_path`. In Bazel, the `short_path` for files in an external
   repository starts with `../` (e.g., `../content+/src/SUMMARY.md`). This
   causes the files to be staged **outside** the shadow root (as siblings to
   it).

2. **Invocation (`mdbook.bzl`):** The `_mdbook_impl` function in `mdbook.bzl`
   tells `mdbook` where the book's root is using `${pwd}/{}` formatted with
   `ctx.file.book.dirname`. For external repositories, `dirname` (based on
   `file.path`) starts with `external/` (e.g., `external/content+/`).

3. **The Mismatch:** `mdbook` is invoked with a path like
   `.../repro-external-sources_/external/content+/`, but the files were actually
   staged by the wrapper at `.../repro-external-sources_/../content+/`.

## Reproduction

### Case 1: All files in an external repository

```bash
bazel build //:repro-external-sources
```

Failure:

```
[ERROR] (mdbook::utils): Error: Couldn't open SUMMARY.md in ".../repro-external-sources_/external/content+/src"
```

The rule expects a nested `external/` directory that doesn't exist in the shadow
tree.

### Case 2: Local book.toml, External sources

```bash
bazel build //:repro-mixed-sources
```

Failure:

```
[ERROR] (mdbook::utils): Error: Couldn't open SUMMARY.md in ".../repro-mixed-sources_/src"
```

Because `book.toml` is local, `mdbook` starts in the shadow root. It expects a
`src/` directory there, but the sources were staged as siblings to the root at
`../content+/src/`.

## Code References

- **Rule Implementation:** `extensions/mdbook/private/mdbook.bzl` (uses
  `ctx.file.book.dirname`)
- **Wrapper Staging:** `extensions/mdbook/private/process_wrapper.rs` (uses
  `short_path` via the input manifest)
