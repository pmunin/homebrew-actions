# Formula template gallery

All templates use `envsubst` `${VAR}` placeholders. The only variable substituted by default is the
one(s) you pass in `extra-envs` — usually `TAG`. The Ruby `class` name is the formula name in
CamelCase (`gh-goto` → `GhGoto`, `my_tool` → `MyTool`).

## 1. Minimal — install a single binary from a git tag

```ruby
class MyTool < Formula
  desc "One-line description"
  homepage "https://github.com/<owner>/<repo>"
  url "https://github.com/<owner>/<repo>.git", tag: "${TAG}"
  head "https://github.com/<owner>/<repo>.git", branch: "main"

  def install
    bin.install "bin/my-tool"
  end
end
```

## 2. Binary + zsh completion + shared shell script + deps + caveats

Based on a real self-tap formula (`gh-goto`). Shows `depends_on`, completions, and `caveats`.

```ruby
class GhGoto < Formula
  desc "Resolve and jump to GitHub repo directories via GitHub CLI"
  homepage "https://github.com/<owner>/homebrew-gh-goto"
  url "https://github.com/<owner>/homebrew-gh-goto.git", tag: "${TAG}"
  head "https://github.com/<owner>/homebrew-gh-goto.git", branch: "main"

  depends_on "<owner>/source-into-shell/source-into-shell"
  depends_on "gh"

  def install
    bin.install "bin/gh-goto"
    (share/"source-into-shell"/"gh-goto").install "bin/gh-goto.source.sh"
    zsh_completion.install "bin/_gh-goto"
  end

  def caveats
    <<~EOS
      Ensure your ~/.zshrc contains:
        source "$(brew --prefix)/share/source-into-shell/source-this.sh"
    EOS
  end
end
```

## 3. Tarball + sha256 (bottle-style, no build from source)

Use when you ship a prebuilt release asset rather than building from a git checkout. Requires a
checksum, so pass it via `extra-envs` too. Compute it in the release workflow before the action runs
(e.g. `SHA256=$(shasum -a 256 dist/mytool.tar.gz | cut -d' ' -f1)`).

```ruby
class MyTool < Formula
  desc "One-line description"
  homepage "https://github.com/<owner>/<repo>"
  url "https://github.com/<owner>/<repo>/releases/download/${TAG}/mytool-${TAG}.tar.gz"
  sha256 "${SHA256}"
  version "${VERSION}"

  def install
    bin.install "mytool"
  end
end
```

Corresponding `extra-envs`:

```yaml
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
            VERSION=${{ github.event.release.tag_name }}
            SHA256=${{ steps.checksum.outputs.sha256 }}
```

## 4. Private repo — prebuilt asset via the gh CLI (no token env var)

For a PRIVATE source repo, the public `releases/download/...` URL 404s without credentials. A small
download strategy that shells out to `gh release download` reuses the user's existing `gh auth` — no
`HOMEBREW_GITHUB_API_TOKEN` to manage. Requires `depends_on "gh"`.

```ruby
require "download_strategy"

# gh-release://<owner>/<repo>/<tag>/<asset-filename>
class GhReleaseDownloadStrategy < AbstractDownloadStrategy
  def fetch(timeout: nil, **_options)
    gh = which("gh") || HOMEBREW_PREFIX/"bin/gh"
    raise "GitHub CLI (gh) is required. Run: brew install gh && gh auth login" unless File.executable?(gh.to_s)

    repo, tag, filename = url.match(%r{gh-release://([^/]+/[^/]+)/([^/]+)/(.+)}).captures
    system(gh.to_s, "release", "download", tag, "-R", repo, "-p", filename,
           "-O", cached_location.to_s, "--clobber") || raise("gh download failed. Run: gh auth login")
  end

  def cached_location
    @cached_location ||= HOMEBREW_CACHE/"downloads/#{name}-#{version}"
  end

  def clear_cache
    cached_location.unlink if cached_location.exist?
  end
end

class MyTool < Formula
  desc "One-line description"
  homepage "https://github.com/<owner>/<repo>"
  version "${VERSION}"
  depends_on "gh"

  on_macos do
    if Hardware::CPU.arm?
      url "gh-release://<owner>/<repo>/${TAG}/mytool-darwin-arm64", using: GhReleaseDownloadStrategy
      sha256 "${ARM_SHA}"
    else
      url "gh-release://<owner>/<repo>/${TAG}/mytool-darwin-x64", using: GhReleaseDownloadStrategy
      sha256 "${X64_SHA}"
    end
  end

  def install
    bin.install cached_download => "mytool"            # single raw binary
    # Multiple binaries shipped in one tar.gz instead:
    #   system "tar", "-xzf", cached_download, "-C", buildpath
    #   bin.install "tool-a", "tool-b"
  end
end
```

`extra-envs` — compute the checksums in the **same job that builds the assets**, then pass them:

```yaml
          extra-envs: |
            TAG=${{ github.event.release.tag_name }}
            VERSION=${{ steps.meta.outputs.version }}
            ARM_SHA=${{ steps.meta.outputs.arm_sha }}
            X64_SHA=${{ steps.meta.outputs.x64_sha }}
```

Because the shas come from the built assets, run `upsert-formula` **inside the build job** (after the
upload step) rather than as a standalone `bump-formula.yml` — otherwise the checksums aren't available.

## Tips

- Add a `test do ... end` block if you want `brew test <formula>` to work; e.g.
  `assert_match "usage", shell_output("#{bin}/my-tool --help")`.
- For language-specific builds (Go, Rust, Node), keep the build commands inside `def install` exactly
  as you'd run them locally; the runner has the toolchains available via `depends_on`.
- Keep version-specific values out of the template except through `${...}` placeholders so the same
  file renders correctly for every release.
