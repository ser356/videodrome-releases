# Videodrome releases

Public release artifacts for **[Videodrome](https://github.com/ser356/videodrome)**.
The source code is private; this repo hosts the built binaries so that
Homebrew and Scoop can download them anonymously.

## Install

**macOS (arm64):**

```bash
brew tap ser356/cask
brew install --cask videodrome
```

**Windows (x86_64):**

```powershell
scoop bucket add ser356 https://github.com/ser356/scoop-bucket
scoop install ser356/videodrome
```

**Linux (x86_64, CLI):**

```bash
curl -sSfL https://github.com/ser356/videodrome/raw/main/docs/setup.sh | bash
```

**Linux GUI:** descarga el `.AppImage` o `.deb` de la última release
en la pestaña [Releases](https://github.com/ser356/videodrome-releases/releases).

## How this works

- Code lives in [ser356/videodrome](https://github.com/ser356/videodrome) (private).
- On tag push there (`v*`), a tiny workflow in the private repo
  dispatches an event here that runs `.github/workflows/release.yml`.
- CI checks out the private source with a fine-grained PAT
  (`SOURCE_REPO_PAT`, `contents:read` only), builds mac/win/linux, and
  publishes the release + updates the cask/scoop bucket.
- Manual dispatch is also supported:
  ```bash
  gh workflow run release.yml --repo ser356/videodrome-releases -f tag=v1.16.0
  ```

## Bootstrap

See `RELEASE.md` in the private repo for the full runbook.
