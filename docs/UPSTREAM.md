# Upstream tracking

This repository is a maintained Xalterra distribution of Casys AI's
`mcp-erpnext`. It is not an independent implementation — application code is
reused as-is wherever possible, per `docs/XALTERRA.md`.

## Source

- **Repository:** https://github.com/Casys-AI/mcp-erpnext
- **License:** MIT, `Copyright (c) 2026 Casys AI` (see `LICENSE`, unmodified)

## Selected baseline

- **Tag:** `v3.0.3`
- **Commit SHA:** `2a88ac5807c7869794f0635a5422221b5360a590`
- **Commit date:** 2026-08-30 (`chore: release 3.0.3`)
- **GitHub release status at selection time:** marked **"Latest"** — the most
  recent non-prerelease tag.

### Why this tag and not something newer

At the time of writing, upstream's tag list (newest first) was:

```
v3.1.0-beta.6   (prerelease)
v3.1.0-beta.5   (prerelease)
v3.0.3          <- selected — latest stable release
v3.1.0-beta.4   (prerelease)
v3.1.0-beta.3   (prerelease)
v3.1.0-beta.2   (prerelease)
v3.1.0-beta.1   (prerelease)
v3.0.2          (stable, superseded by v3.0.3)
...
```

`v3.1.0-beta.*` is an active, unreleased development line (viewer/attachment
work, per its own changelog entries) sitting ahead of `v3.0.3` in commit
history. Per our own policy, a production fork must not be based on a beta or
release-candidate merely because it carries a higher version number. `v3.0.3` is
upstream's own designated "Latest" stable release and its changelog entry ("adds
a deny-by-default escape hatch for allowlisted Frappe method calls") describes a
security-relevant fix consistent with a stable patch release, not experimental
work.

## Remote layout

```
origin    git@github.com:xalterra/mcp-erpnext.git      (this repository)
upstream  https://github.com/Casys-AI/mcp-erpnext.git  (Casys AI, read-only)
```

Local `main` was created from the `v3.0.3` tag (`git checkout -b main
v3.0.3`),
so it carries upstream's full commit history up to that point. `upstream/main`
(and upstream's other branches) remain fetchable but are **ahead** of our
baseline with the unreleased 3.1 beta line — do not merge or rebase onto
`upstream/main` directly; pull a specific future stable tag instead.

## Pulling future upstream changes

1. `git fetch upstream --tags`
2. Review upstream's `CHANGELOG.md` and release notes for the candidate tag —
   apply the same stable-vs-prerelease judgement used above.
3. `git merge <tag>` (preferred, keeps history honest) or `git rebase <tag>` if
   the Xalterra diff is still small enough that a rebase stays clean.
4. Resolve conflicts, re-run the full validation in `AGENTS.md` /
   `docs/tools.md`'s "Build, Test, and Development Commands" section
   (`deno
   fmt --check`, `deno lint`, `deno task check`,
   `deno test --allow-all
   src/`, Docker build), then commit.
5. Update this file's "Selected baseline" section to the new tag/SHA/date and
   reason.

Keep the Xalterra diff against upstream as small as practical (see
`docs/XALTERRA.md`) — the smaller it is, the more of these updates a `merge` can
absorb with no manual work at all.
