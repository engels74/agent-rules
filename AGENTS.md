# agent-rules — working notes

## What this repo is

The single source of truth for `.agents/rules/*.md` across my projects.
Content only, plus one shell script. There is no package manifest,
lockfile, build step, test suite, or CI workflow, and none should be added
without being asked.

## Hard rules

- **`publish` defaults to `false`.** A repo commits its rule files only by
  opting in. Repos that must not appear in a public target list belong in
  `manifest.local.toml`, which is gitignored.
- **Key on the origin remote slug.** Local directory names do not match slugs
  (`engels74/afisharr` lives at `afisharr-project/afisharr`; `poyo-studio` has a
  second checkout at `poyo-local`). Never infer a repo from its path.
- **Prune in the same pass as the write.** Every consumer's old filename differs
  from its new one. A repo left holding both gets contradictory guidance.
- **De-publish before excluding.** Git ignores exclude rules for already-tracked
  paths, so `git rm --cached` must come first or the guard silently does nothing.
- **Verify the copy.** Compare checksums after writing. A truncated or
  re-encoded copy is otherwise invisible.
- **Reconcile toolchain floors before syncing**, not after. See
  `docs/toolchain-floors.md`.

## Editing a rule file

One flat file per stack. Do not split into `references/` — consumers read a
single `.md` from `.agents/rules/`, and a split would require a concatenation
step this repo deliberately does not have.

Check dependency pins against the registry before trusting them. The canonical
files were generated in bulk; one shipped a package that has never existed and
four pins a full major generation stale. A drifted repo copy affects one repo;
a drifted canonical file affects every consumer at once.
