# `upsert-formula` action inputs

Action: `pmunin/homebrew-actions/upsert-formula@main`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `formula-name` | No | Repo name (strips leading `homebrew-`) | Name of the formula file: `Formula/<name>.rb`. |
| `formula-template-path` | **Yes** | | Path to the `.rb` template in the calling repo workspace (checked out). |
| `extra-envs` | No | | Extra env vars for `envsubst`, multiline `KEY=VALUE`. ONLY these keys are substituted. |
| `homebrew-repo` | No | Current repo (`${{ github.repository }}`) | Target tap repo, e.g. `owner/homebrew-tap`. |
| `branch` | No | Tap repo's default branch | Branch in `homebrew-repo` to commit the formula to. |
| `formula-commit-message` | No | `Upsert <name> formula` | Commit message for the formula upsert (`[skip ci]` is appended automatically). |
| `formula-commit-tag` | No | | If set, (re)points this tag at the formula commit in the target repo. Use for self-tap repos. |
| `github-write-token` | No | `${{ github.token }}` | Token with write access to `homebrew-repo`. Default works same-repo; use a PAT cross-repo. Passing an unset secret sends `""`; the action then warns and falls back to `GITHUB_TOKEN`. |

## Behavior notes

- **Formula name default:** repo name with a leading `homebrew-` removed (`homebrew-gh-goto` → `gh-goto`).
- **Substitution is opt-in:** the action builds the `envsubst` allowlist only from `extra-envs` keys,
  so Ruby string interpolation and shell-like `$()` in the template are left untouched. Always pass
  `TAG=${{ github.event.release.tag_name }}` if the template uses `${TAG}`.
- **No-op safe:** if the rendered formula is byte-identical to what's already committed, GitHub returns
  422 and the action treats it as success (no empty commit).
- **Tagging:** with `formula-commit-tag`, the action creates the tag if missing or force-updates it to
  the new commit SHA (or to the tap branch HEAD if the content was unchanged).
- **Cross-repo access:** besides the PAT, the target tap repo must allow Actions access
  (tap repo → Settings → Actions → General → Access).
