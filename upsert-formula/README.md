# Upsert Formula Action

Reusable composite action that renders a formula template and creates or updates `Formula/<name>.rb` in a target Homebrew tap repository.

## Usage

In your repo's workflow (e.g. `.github/workflows/bump-formula.yml`):

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
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
          github-write-token: ${{ secrets.HOMEBREW_PAT }}
```

### Updating a formula in another repo

```yaml
      - uses: pmunin/homebrew-actions/upsert-formula@main
        with:
          formula-template-path: homebrew-formula.rb
          homebrew-repo: <owner>/homebrew-<tap>
          formula-commit-message: "Bump my-tool to ${{ github.event.release.tag_name }}"
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
          github-write-token: ${{ secrets.HOMEBREW_PAT }}
```

### Tagging the formula commit

Useful when the repo is itself a Homebrew tap and you want the release tag to point at the commit that includes the updated formula:

```yaml
      - uses: pmunin/homebrew-actions/upsert-formula@main
        with:
          formula-template-path: homebrew-formula.rb
          formula-commit-tag: ${{ github.event.release.tag_name }}
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
          github-write-token: ${{ secrets.HOMEBREW_PAT }}
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `formula-name` | No | Repo name (strips `homebrew-` prefix) | Name of the formula file (`Formula/<name>.rb`) |
| `formula-template-path` | Yes | | Path to formula template in the calling repo |
| `extra-envs` | No | | Additional env vars for `envsubst`, multiline `KEY=VALUE` |
| `homebrew-repo` | No | Current repository | Target Homebrew tap repository (e.g. `owner/homebrew-tap`) |
| `formula-commit-message` | No | `Upsert <formula-name> formula` | Custom commit message for the formula upsert |
| `formula-commit-tag` | No | | If set, upsert this tag to the formula commit in the target repo |
| `github-write-token` | Yes | | GitHub PAT with write access to the target `homebrew-repo` |

## Template

The template file uses `envsubst` syntax (`${VAR_NAME}`). Only variables from `extra-envs` are substituted.

Example template (`homebrew-formula.rb`):

```ruby
class MyTool < Formula
  desc "My tool description"
  homepage "https://github.com/<owner>/my-tool"
  url "https://github.com/<owner>/my-tool.git", tag: "${TAG}"

  def install
    bin.install "bin/my-tool"
  end
end
```

## Setting up the PAT

The action needs a GitHub Personal Access Token with write access to the target homebrew tap repo, because `GITHUB_TOKEN` is scoped to the calling repo only.

1. Create a fine-grained PAT at [https://github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens):
   - **Repository access:** select your target homebrew tap repository
   - **Permissions:** Contents → Read and write
2. Add the PAT as a secret in your repo:

   ```sh
   gh secret set HOMEBREW_PAT -R <owner>/<your-repo>
   ```

3. Reference it in your workflow as `${{ secrets.HOMEBREW_PAT }}`.

## Access settings

Ensure the target homebrew tap repository allows actions from your repos:

**Target repo** → Settings → Actions → General → "Access" → set to allow access from repositories that need it.
