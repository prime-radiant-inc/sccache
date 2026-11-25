# Prime Radiant's sccache fork

This is mozilla/sccache with one open upstream pull request applied. We run it
because our agents build Rust in many git worktrees of the same repository at
once, and stock sccache can't share our own workspace crates between those
worktrees. Each worktree recompiled everything it owned from scratch.

Delete this fork as soon as upstream releases the change. Until then, keep the
patch set to exactly that pull request, rebased onto upstream.

## What's in it

The `primeradiant` branch is upstream `main` (0.20.1) plus
**[#3065](https://github.com/mozilla/sccache/pull/3065), per-compile
`SCCACHE_BASEDIRS`.**

`SCCACHE_BASEDIRS` lists directory prefixes that sccache strips from paths
before hashing a compile. That's what lets two checkouts at different paths
share cache entries. Stock sccache reads the variable once, when the server
starts, and applies it to every compile. One server serves every worktree on a
machine, so it can't hold each worktree's own root, and agent worktrees come
and go all day.

With #3065, sccache also reads the variable from each compile's environment.
Each worktree then names its own root through cargo:

```toml
[env]
SCCACHE_BASEDIRS = { value = "", relative = true }
```

Felixoid, who wrote the original basedirs feature, authored #3065. The
maintainer was reviewing it in November 2026, with only style comments.

## What isn't in it, and why

Stripping the root from the cache key isn't enough on its own. rustc writes the
checkout's absolute path into what it compiles, for example a build script's
`OUT_DIR`. So a crate's output differs between worktrees, and every crate that
depends on it misses. The cure is `--remap-path-prefix=<root>=.` on every rustc
call.

Upstream [#3021](https://github.com/mozilla/sccache/pull/3021)
(`SCCACHE_BASEDIRS_REMAP_PATHS`) adds that remap inside sccache, but only for
compiles sccache caches. Proc-macros and binaries aren't cacheable, so sccache
hands them back to rustc unchanged. A proc-macro such as `rustversion` then
still holds the checkout path, and everything using it misses. We measured 505
hits and 15 misses with #3021, against 520 and 0 with the remap on every call.

So the remap lives in chao's rustc wrapper (`scripts/rustc-wrapper.sh`), which
adds it to every rustc call, cacheable or not. This fork stays one pull request.

## Why: the measurements

We benchmarked chao (`cargo build --workspace --all-targets`) on an idle
16-core AWS instance in November 2026. Worktree A built first into an empty
cache. Worktree B, a second checkout of the same commit, built next.

| Worktree B, warm cache | Wall | Rust cache hits / misses |
|---|---|---|
| stock sccache 0.20.0 | 263s | 327 / 193 |
| this fork, basedirs from `[env]` | 224s | 504 / 16 |
| plus #3021's remap | 231s | 505 / 15 |
| plus chao's wrapper remapping every rustc call | 128s | 520 / 0 |

The last row also needs two chao test helpers that stopped compiling
`env!("CARGO_MANIFEST_DIR")` into library code. What's left (about 100s) is
linked test binaries, which sccache never caches.

Chao's `docs/engineering/sccache.md` has the full reasoning.

## Releases

Pushing a tag `v<upstream version>-primeradiant.<N>` runs
`.github/workflows/primeradiant-release.yml`. It builds `sccache` for
`x86_64-unknown-linux-musl` and `aarch64-apple-darwin`, and publishes each as
`sccache-<tag>-<target>.tar.gz` with a `.sha256` beside it. Chao's
`scripts/rustc-wrapper.sh` pins a tag and those SHA-256 sums.

Upstream's own workflows are disabled in this repository. They build some 15
targets on every push.

## Updating

```sh
git fetch upstream
git rebase upstream/main            # or onto a new release tag
# bump version in Cargo.toml and the sccache entry in Cargo.lock:
#   <upstream version>-primeradiant.<N+1>
cargo test --lib
git push --force-with-lease origin primeradiant
git tag v<version> && git push origin v<version>
```

Then update the tag and the SHA-256 sums in chao's `scripts/rustc-wrapper.sh`.

When upstream merges #3065, drop its commits during the rebase. When upstream
releases it, point chao's wrapper back at upstream's release and archive this
repository.
