---
name: homebrew-formula-release
description: Set up a GitHub Actions release workflow that automatically updates a Homebrew formula whenever a GitHub release is published, using the pmunin/homebrew-actions/upsert-formula GitHub Action. Use when the user wants to publish a CLI/tool to Homebrew, create or update a brew tap, add a GitHub Actions workflow on release, or auto-bump/upsert a formula on each GitHub release.
license: MIT
metadata:
  author: Philipp Munin
  version: "0.0.1"
---

# Homebrew Formula Release

Configure a GitHub repo so that publishing a release automatically renders a Homebrew
formula template and upserts `Formula/<name>.rb` in a tap repository — via the
[`pmunin/homebrew-actions/upsert-formula`](https://github.com/pmunin/homebrew-actions) composite action.

Use this skill when the user says things like "make this installable with brew", "publish my CLI
to Homebrew", "auto-update my formula on release", or "set up a brew tap for this tool".

## How the action works (mental model)

On `release: published`, the workflow:
1. Renders a formula template (`*.rb` with `envsubst` `${VAR}` placeholders — typically `${TAG}`).
2. Upserts `Formula/<formula-name>.rb` into the target tap repo via the GitHub API.
3. Optionally moves the release tag onto the new formula commit (self-tap repos only).

Only variables you pass in `extra-envs` are substituted — everything else in the template is left
verbatim. `formula-name` defaults to the repo name with a leading `homebrew-` stripped.

## Step 1 — Identify the scenario

Detect the current repo with `gh repo view --json name,nameWithOwner,defaultBranchRef` and ask the
user (or infer) which layout applies:

- **A. Self-tap source repo** — the repo holds the tool's source code AND is the tap (its name
  usually starts with `homebrew-`, and it has, or will have, a `Formula/` dir). The formula's `url`
  points at this same repo's release tag. Example: `pmunin/homebrew-gh-goto`.
- **B. Separate tap repo** — source code lives here, but the formula is written to a different tap
  repo (e.g. `owner/homebrew-tap`). The formula's `url` points at THIS repo's release tag.

The scenario determines auth (Step 4) and a couple of workflow inputs (Step 3).

## Step 2 — Create the formula template

Create `homebrew-formula.rb` in the repo root (or another path — remember it for `formula-template-path`).

Gather what the formula needs from the user / repo: install targets (binaries, completions, shared
scripts), runtime `depends_on`, homepage, description, and any `caveats`. Use `${TAG}` for the
version so the same template works for every release.

Minimal tag-based template:

```ruby
class MyTool < Formula
  desc "One-line description of the tool"
  homepage "https://github.com/<owner>/<repo>"
  url "https://github.com/<owner>/<repo>.git", tag: "${TAG}"
  head "https://github.com/<owner>/<repo>.git", branch: "main"

  def install
    bin.install "bin/my-tool"
  end
end
```

The Ruby `class` name is the formula name in CamelCase (e.g. `gh-goto` → `GhGoto`).
See `references/templates.md` for completions, shared scripts, deps, caveats, and bottle/sha256 variants.

## Step 3 — Create the release workflow

Create `.github/workflows/bump-formula.yml`. Pick the variant matching the scenario.

**Scenario A — self-tap source repo** (default token, re-tag the formula commit):

```yaml
name: Bump Homebrew Formula
on:
  release:
    types: [published]

jobs:
  bump:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: pmunin/homebrew-actions/upsert-formula@main
        with:
          formula-template-path: homebrew-formula.rb
          formula-commit-tag: ${{ github.event.release.tag_name }}
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
```

**Scenario B — separate tap repo** (needs a PAT, see Step 4):

```yaml
name: Bump Homebrew Formula
on:
  release:
    types: [published]

jobs:
  bump:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pmunin/homebrew-actions/upsert-formula@main
        with:
          formula-template-path: homebrew-formula.rb
          homebrew-repo: <owner>/homebrew-<tap>
          formula-commit-message: "Bump <tool> to ${{ github.event.release.tag_name }}"
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
          github-write-token: ${{ secrets.HOMEBREW_PAT }}
```

Full input reference: `references/inputs.md`.

## Step 4 — Auth

- **Scenario A:** the workflow's `GITHUB_TOKEN` is enough — just keep `permissions: contents: write`
  (already in the snippet). No secret needed.
- **Scenario B:** `GITHUB_TOKEN` can't write to another repo, so create a fine-grained PAT and add it
  as a secret:
  1. Create the PAT at https://github.com/settings/personal-access-tokens — **Repository access** =
     the target tap repo; **Permissions** → Contents → Read and write.
  2. `gh secret set HOMEBREW_PAT -R <owner>/<this-repo>` (paste the token when prompted).
  3. Reference it as `github-write-token: ${{ secrets.HOMEBREW_PAT }}` (already in the snippet).
  4. Ensure the tap repo allows the action: tap repo → Settings → Actions → General → Access.

## Step 5 — Ship & verify

1. Have the user commit and push the template + workflow (do not commit on their behalf unless asked).
2. Publish a GitHub release with a semver tag (e.g. `gh release create v0.1.0 --generate-notes`).
3. Confirm the workflow ran (`gh run list --workflow bump-formula.yml`) and that
   `Formula/<name>.rb` was created/updated in the tap repo.
4. Install: `brew install <owner>/<tap>/<formula-name>` (tap repo name minus `homebrew-` is the tap).

## Notes & gotchas

- Template placeholders are ONLY filled for keys present in `extra-envs`. If `${TAG}` stays literal in
  the output, the `TAG=...` line is missing.
- Identical content is a no-op — the action treats GitHub's "no changes" 422 as success.
- Tag-based `url` (`tag: "${TAG}"`) needs no checksum and is the simplest path. Use a tarball + sha256
  only if you need bottle-style installs; see `references/templates.md`.
- This skill pins the action with `@main`. Pin to a release tag instead if the user wants stability.
