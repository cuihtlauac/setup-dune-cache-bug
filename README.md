# setup-dune Cache Version Conflict Investigation

This repository was created to investigate a potential cache version conflict issue with `ocaml-dune/setup-dune`.

## Background

In the [ocaml.org](https://github.com/ocaml/ocaml.org) CI, we observed errors like:
```
File "...dune-package", line 1, characters 11-15:
1 | (lang dune 3.22)
               ^^^^
Error: Version 3.22 of the dune language is not supported.
Supported versions: ... 3.0 to 3.21
```

## Findings

### Cache Keys Include Dune Version

The `setup-dune` action **does** include the dune version in cache keys:
- Nightly builds use prefix: `dune-dev-Linux-X64-...`
- Version 3.21.0 uses prefix: `dune-3.21.0-Linux-X64-...`

This means caches are isolated between nightly and specific version builds.

### The Actual Bug Scenario

The bug occurs when:
1. **All nightly builds share the `dune-dev-` cache key prefix**
2. If nightly advances from 3.21.x to 3.22.x between CI runs
3. The newer nightly's `_build/` artifacts (with `(lang dune 3.22)`) are cached
4. A subsequent run restores this cache but the nightly binary might behave differently

This is **timing-dependent** and hard to reproduce in a controlled PoC.

### What Happened in ocaml.org

The issue in ocaml.org was slightly different:
1. `build_with_dev_preview.yml` used setup-dune with nightly (3.22+)
2. setup-dune sets `DUNE_CACHE_ROOT` environment variable
3. Our Makefile downloaded a specific dune version (3.21.0)
4. Our dune inherited `DUNE_CACHE_ROOT` and found artifacts built by nightly 3.22

The fix was to use `version: latest` (stable 3.21.0) in setup-dune to match our Makefile's dune version.

## Recommendations for setup-dune

1. **Include actual nightly version in cache key** (not just `dune-dev-`)
   - Currently: `dune-dev-Linux-X64-...`
   - Better: `dune-dev-3.22.0-Linux-X64-...` or include the git revision

2. **Document the cache key format** so users understand isolation behavior

3. **Consider making `version: latest` the default** now that dune pkg is more mature

## Workflows in This Repo

- `step1-nightly.yml`: Builds with nightly dune, shows cache key and package versions
- `step2-older.yml`: Builds with dune 3.21.0, shows cache behavior

These demonstrate that different version specifications result in different cache key prefixes.
