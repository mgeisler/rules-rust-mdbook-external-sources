# rules_rust_mdbook External Repository Path Bug

This repository contains a minimal reproduction of a path resolution bug in
`rules_rust_mdbook`.

## The Problem

When any part of an `mdbook` target (either the `book.toml` or the source files)
comes from an **external repository**, the build fails. This is because the
rule's staging logic and its invocation logic handle repository boundaries
inconsistently.

### Root Cause

1. **Staging:** The `rules_rust` Rust `process_wrapper` stages files into a
   shadow work directory using `file.short_path`. For external repositories,
   this path starts with `../`.
2. **Invocation:** The Starlark rule tells `mdbook` the root path using
   `file.path`. For external repositories, this path starts with `external/`.
3. **Cross-Repo boundaries:** Even if the book configuration is local, external
   sources are staged "outside" the shadow tree root (due to the `../` prefix in
   `short_path`), making them inaccessible to `mdbook`.

## Reproduction

### Case 1: All files in an external repository
```bash
bazel build //:repro-external-sources
```
Fails with:
`Error: Couldn't open SUMMARY.md in ".../repro-external-sources_/external/content+/src"`.
The files are actually staged at `repro-external-sources_/../content+`.

### Case 2: Local book.toml, External sources
```bash
bazel build //:repro-mixed-sources
```
Fails with:
`Error: Couldn't open SUMMARY.md in ".../repro-mixed-sources_/src"`. The
sources are missing from the shadow root because they were staged at
`repro-mixed-sources_/../content+/src`.
