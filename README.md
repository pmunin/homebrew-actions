# homebrew-actions

Reusable GitHub Actions for Homebrew formula management — plus a Claude Code skill to wire them up.

## Actions

### [upsert-formula](./upsert-formula)

Renders a formula template and upserts `Formula/<name>.rb` in a target Homebrew tap repository on release.

See [upsert-formula/README.md](./upsert-formula/README.md) for usage and setup details.

## Claude Code skill

This repo doubles as a Claude Code [marketplace](.claude-plugin/marketplace.json). The
[`homebrew-formula-release`](./plugins/homebrew-formula-release) plugin adds a skill that sets up the
`upsert-formula` action on any repo (formula template + release workflow + auth) through a guided flow.

### Install

```
/plugin marketplace add pmunin/homebrew-actions
/plugin install homebrew-formula-release@homebrew-actions
```

Then just ask Claude something like "make this repo installable with brew on release".

### Update

```
/plugin marketplace update homebrew-actions
```

The skill lives in the same repo as the action, so it stays in sync with the action's inputs.
