# Toolchain floors

A rule file that targets a newer toolchain than a repo pins does not produce a
warning. It produces a **silent CI outage**, and local gates stay green while it
happens.

## The proven case: Bun

`rules/astro-svelte5-islands.md` targets Bun 1.4.x. Estate repos pinned
`bun@1.3.14` in `packageManager`, and CI resolves the runner's Bun from that
field via `bun-version-file: package.json`. A lockfile generated on 1.4.0 is
`lockfileVersion: 2`, which 1.3.14 cannot parse:

```
bun install v1.3.14
2 |   "lockfileVersion": 2,
error: Unknown lockfile version at bun.lock:2:22
error: lockfile had changes, but lockfile is frozen
```

Every workflow failed at its first step, so nothing downstream ran — not lint,
types, tests or build. The same pin governs `deploy.yml`, so merging would have
broken the production deploy quietly: the previous `gh-pages` build stays live.

Two properties make this worse than an ordinary version bump:

1. **It only bites on a *fresh* lockfile.** Bun 1.4 writes v2 for a new lockfile
   but preserves v1 in an existing one. One repo was hit because its lockfile
   was recreated when migrating off npm; the other, whose `bun.lock` predates
   the migration, stayed v1 and stayed green. Any repo that regenerates its
   lockfile inherits the break.
2. **Local verification cannot see it.** The local Bun *is* 1.4.0. CI is the
   only signal.

Resolved in the two affected repos by moving to `bun@1.4.0` with `@types/bun`
alongside.

## Outstanding

| Rule | Targets | Repo pins | Status |
|---|---|---|---|
| `astro-svelte5-islands` | Bun 1.4.x | `bun@1.3.14` in 7 repos | **unreconciled** |
| `swift-6_3-appletv` | tvOS 26 | `project.yml` 18.0 | unverified |
| `rust-1_98-core` | Rust 1.98 | `rust-toolchain.toml` 1.93–1.97 | unverified |
| `go-1_27-core` | Go 1.27 | `go.mod` 1.25.0 | unverified |

The seven remaining Bun repos are one lockfile regeneration away from the break
above.

## The general lesson

Any verification that stops at "it builds here" ships this class of defect. The
break lived in the interaction between a committed pin and a locally-generated
lockfile — invisible to every local gate.
