# Formula Update Workflow

Reference for how a release gets from a tool repo into this tap automatically.

## Overview

```
tool repo (tag push v*) ──▶ build + publish release ──▶ repository_dispatch ──▶ tap update workflow
                                                                              │
                                                                fetch tarballs + sha256
                                                                generate Formula/<name>.rb
                                                                brew audit --strict
                                                                commit to main
```

Two halves. The **tool repo** owns building and publishing. The **tap** owns
generating and committing the formula. The bridge between them is a
`repository_dispatch` event carrying the tag.

## Side 1: Tool repo (release workflow)

- Trigger: `push` to tags matching `v*.*.*`.
- `permissions: contents: write` (needed to publish the release).
- Steps:
  1. Build the single static binary per platform using the project's native
     toolchain (bun / cargo / go — whatever the project uses).
  2. Package as gzipped tar archives, flat layout with the binary name inside:
     - `<name>-<version>-darwin-arm64.tar.gz`
     - `<name>-<version>-darwin-amd64.tar.gz`
     - `<name>-<version>-linux-amd64.tar.gz`
     Each archive contains a single file named `<name>` at its root (so
     `bin.install "<name>"` works unchanged in the formula).
  3. Publish with `softprops/action-gh-release` (`files: <name>-*.tar.gz`,
     `generate_release_notes: true`).
  4. Fire the dispatch to the tap:
     ```sh
     gh api repos/fbcgulfport/homebrew-tap/dispatches \
       --method POST \
       -f event_type=<name>_release \
       -F client_payload[repo]=fbcgulfport/<name> \
       -F client_payload[tag]="$TAG"
     ```
     This requires a fine-grained PAT with `contents: write` on this tap repo,
     stored as a secret in the tool repo (e.g. `TAP_DISPATCH_TOKEN`).

## Side 2: This tap (per-formula update workflow)

One workflow per formula: `.github/workflows/update-<name>.yaml`.

- Triggers:
  - `repository_dispatch` with `types: [<name>_release]` (from the tool repo).
  - `workflow_dispatch` with a `tag: v0.1.0` input (manual backfills).
- `permissions: contents: write`.
- `concurrency: group: update-<name>-formula, cancel-in-progress: false` so two
  rapid releases can't race a commit.
- Steps:
  1. Resolve `TAG` and `REPO` from `client_payload`, falling back to
     `workflow_dispatch` inputs / the default repo. Validate
     `TAG =~ ^v[0-9]+\.[0-9]+\.[0-9]+$` before using it.
  2. Download each platform tarball (`curl -fsSL <release-url>`) and compute
     sha256 in-repo:
     `sha="$(curl -fsSL "$url" | shasum -a 256 | cut -d ' ' -f 1)"`.
  3. Generate `Formula/<name>.rb` via a shell heredoc, templated with the URLs
     and sha256s. Shape:
     - `class <Name> < Formula` + `desc` / `homepage` / `license`
     - `on_macos` → `on_arm` (darwin-arm64) and `on_intel` (darwin-amd64) blocks
     - `on_linux` → `on_intel` (linux-amd64) block, plus
       `on_arm { disable! date: "<today>", because: "<name> does not provide Linux ARM builds" }`
     - `def install; bin.install "<name>"; end`
     - `test do; assert_path_exists bin/"<name>"; end`
  4. Link the checkout as a live tap so audit runs against it:
     ```sh
     tap_dir="$(brew --repository)/Library/Taps/fbcgulfport/homebrew-tap"
     mkdir -p "$(dirname "$tap_dir")"
     ln -s "$GITHUB_WORKSPACE" "$tap_dir"
     ```
     Homebrew set up first via `Homebrew/actions/setup-homebrew@main`.
  5. `brew audit --strict --formula fbcgulfport/tap/<name>` with
     `HOMEBREW_NO_INSTALL_FROM_API: 1` (third-party taps don't live in the
     homebrew API).
  6. Skip if unchanged (`git diff --cached --quiet -- Formula/<name>.rb` → exit 0).
     Otherwise commit to `main` **via the GitHub GraphQL API**
     (`createCommitOnBranch` with `expectedHeadOid`), which produces a
     GitHub-signed commit without pushing tokens to git remotes. Verify the
     returned `signature.isValid` is `true` before declaring success. A plain
     PAT push (`git push https://x-access-token:${PAT}@...`) is the simpler
     fallback but yields unsigned commits.

## Repository dispatch token setup

- Create a fine-grained PAT scoped to `fbcgulfport/homebrew-tap`,
  repository access read+write contents (and metadata).
- Tool repo secret `TAP_DISPATCH_TOKEN` = that PAT.
- Alternatively org-scoped: one PAT + secret shared by every tool repo,
  mapping to a single shared dispatch token.
- The tap itself only needs its own `GITHUB_TOKEN` for the audit/commit steps,
  plus optionally a bot PAT (`BOT_PAT`) if committing via git push instead of
  the GraphQL method.

## Manual usage

Backfill a formula for an already-published release:

```
gh workflow run update-<name>.yaml --repo fbcgulfport/homebrew-tap -f tag=v0.1.0
```

## Adding a new formula

1. Tool repo: add the release workflow (Side 1), archives named
   `<name>-<version>-<platform>.tar.gz`, dispatch at the end.
2. Tap: add `.github/workflows/update-<name>.yaml` from the per-formula
   template, set `desc`, event type, and archive names.
3. Push a tag in the tool repo; the formula commit should land on main
   automatically, then users get it via `brew install fbcgulfport/tap/<name>`.
