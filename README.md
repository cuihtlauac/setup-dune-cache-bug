# setup-dune Cache Version Conflict PoC

This repository demonstrates a potential issue with `ocaml-dune/setup-dune` where cached packages from one dune version can cause failures when a different dune version is used.

## The Problem

The `setup-dune` action caches built packages to speed up subsequent CI runs. However, the cache key doesn't include the dune version. When the cache fallback mechanism finds packages from a different dune version, builds can fail.

Example error:
```
File "...dune-package", line 1, characters 11-15:
1 | (lang dune 3.22)
               ^^^^
Error: Version 3.22 of the dune language is not supported.
Supported versions: ... 3.0 to 3.21
```

## How to Reproduce

1. **Run Step 1 workflow** ("Step 1: Populate Cache with Nightly")
   - Uses `setup-dune@v1` with default (nightly) version
   - Builds the project and caches artifacts
   - Nightly is currently 3.22+

2. **Run Step 2 workflow** ("Step 2: Trigger Bug with Older Version")
   - Uses `setup-dune@v1` with `version: "3.21.0"`
   - Attempts to build, but may restore cache from Step 1
   - Expected: fails because 3.21.0 can't read `(lang dune 3.22)` packages

## Root Cause

The cache key in `setup-dune` includes:
- Cache prefix
- OS/architecture
- Hash of `dune-project`
- Commit SHA

But it does **not** include the dune version. The fallback mechanism can find caches from incompatible dune versions.

## Suggested Fix

Include the dune version in the cache key to prevent cross-version cache pollution.
