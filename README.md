# agent-rules

Canonical `.agents/rules/*.md` for my projects. One file per stack,
hand-edited, no build step.

Two tiers, by account:

| `publish` | Rules | How |
|---|---|---|
| `true` | committed | `.github/workflows/sync.yml` fans out on every push to `rules/` |
| `false` | not committed | hydrated locally by `bin/sync` |

Publishing gets you rules that travel to CI, cloud agents and fresh clones. Not
publishing leaves nothing in the repo at all — see [Why the split](#why-the-split).

## Layout

```
rules/           12 canonical files, flat — source is the artifact
manifest.toml    origin slug -> rules, keyed on the remote, never on path
bin/sync         hydrate, de-publish, prune, verify
docs/            toolchain floors and other cross-repo facts
```

## Usage

```sh
bin/sync                      # dry run over every repo (default)
bin/sync --apply              # act
bin/sync engels74/wings-vpn   # limit to one slug
```

`bin/sync` never commits and never pushes. It stages removals and leaves them
for review; `includeIf.gitdir` picks the right identity when you push.

Requires bash >= 4 (macOS ships 3.2 — `brew install bash`).

## Why the split

Not every repo should carry these files. A long, distinctive rule file
committed in two places is trivially correlated — the 16 exact dependency pins
alone survive any paraphrasing — so some repos are hydrated locally instead and
commit nothing.

Targets live in two places: `manifest.toml` for published repos, and an
optional `manifest.local.toml` for repos that should never appear in a public
list. `bin/sync` reads both.

The local manifest is resolved outside this worktree, first match wins:

1. `$AGENT_RULES_LOCAL_MANIFEST`
2. `$XDG_CONFIG_HOME/agent-rules/manifest.local.toml` (default `~/.config`)
3. `./manifest.local.toml` — legacy fallback, gitignored

Prefer 2. In-tree it was unpushable but still deletable by `git clean -xdf`,
missing from a fresh clone, and one `git add -f` from being staged.

For `publish = false` repos, exclusion is written to `.git/info/exclude` rather
than `.gitignore`, because `.gitignore` is itself committed — a shared ignore
line would be a (weak) fingerprint of its own. `bin/sync` de-publishes tracked
files *before* excluding them (git ignores exclude rules for already-tracked
paths) and then asserts `git check-ignore` passes, so a stray `git add -A`
cannot leak a file.

**Tradeoff:** cloud agents on non-publishing repos will not see these rules.
Keep whatever they need in that repo's committed `AGENTS.md`.

## CI

`sync.yml` runs on push to `rules/**` or `manifest.toml`, clones each
publishing target, runs `bin/sync --apply`, and commits as
`github-actions[bot]`. It needs a repo secret **`SYNC_TOKEN`** — a fine-grained
PAT owned by engels74 with **Contents: Read and write** on the 14 target repos.
Fine-grained PATs expire within a year; the failure mode is a silent stop.

## Adding a rule

1. Write `rules/<name>.md`.
2. Add `<name>` to the relevant `[[repo]]` entries in `manifest.toml`.
3. `bin/sync` to preview, `bin/sync --apply` to land.

Reconcile the toolchain floor in `docs/toolchain-floors.md` first — see that
file for why a floor mismatch is a silent CI outage rather than a warning.

## License

AGPL-3.0-or-later. See [LICENSE](LICENSE).
