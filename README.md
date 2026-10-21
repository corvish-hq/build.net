# build.net

Shared build infrastructure for Corvish .NET projects, consumed as a Git submodule.

## Contents

| Path | Purpose |
|---|---|
| `build.net.props` | Shared MSBuild defaults (nullable, warnings as errors, analyzers, XML docs). **Mandatory** import. |
| `build.net.targets` | Shared MSBuild logic. Currently syncs the shared `*.instructions.md` files into the consumer repo on every local build. |
| `.github/instructions/*.instructions.md` | Shared Copilot/agent rulebook: `base` (always loaded) plus topic files (`csharp`, `ui`, `testing`, `data`) loaded via `applyTo`. Consumer repos extend it with their own `.github/instructions/repo.instructions.md`. |
| `templates/repo.instructions.md` | Starting point for a consumer's `repo.instructions.md`. Copied once, then owned by the consumer. |
| `ReSharper.DotSettings` | Shared ReSharper/Rider code-style settings, injected as a layer by consumer solutions. |

## Usage

1. Add the submodule at the repository root:

   ```powershell
   git submodule add https://github.com/corvish-hq/build.net build.net
   ```

2. Import the defaults **unguarded** at the top of the consumer's `Directory.Build.props`, right after `RepoRoot` is
   defined. Without the submodule the build must stop (`MSB4019`) instead of silently building without the shared
   standards:

   ```xml
   <Import Project="$(RepoRoot)build.net\build.net.props" />
   ```

   Keep repository-specific values in the consumer: `TargetFramework`, `LangVersion`, `AnalysisLevel` (pinned per
   repository), metadata. Extend lists by appending, e.g.
   `<WarningsNotAsErrors>$(WarningsNotAsErrors);CA1716</WarningsNotAsErrors>`.

   Fix a missing submodule with `git submodule update --init --recursive`. CI must check out submodules
   (`actions/checkout` with `submodules: true`) in **every** job that builds.

3. Import the targets **unguarded** from the consumer's `Directory.Build.targets`, for the same reason as the props:

   ```xml
   <Import Project="$(RepoRoot)build.net\build.net.targets" />
   ```

4. Copy `templates/repo.instructions.md` to `.github/instructions/repo.instructions.md` and replace every
   `__PLACEHOLDER__` (product name, UI prefix, paths, file header, domain rules); delete sections that do not apply.
   Then delete all `Hint:` comments. Afterwards,
   `rg "__[A-Z_]+__|Hint:" .github/instructions/repo.instructions.md` must find nothing. The file is owned by the
   consumer from then on — never synced — and wins on conflicts with the baseline.

### Properties

| Property | Default | Meaning |
|---|---|---|
| `RepoRoot` | Parent directory of the submodule | Consumer repository root (trailing slash). Set it in `Directory.Build.props` if the submodule lives elsewhere. |

### Behavior

- All `*.instructions.md` files are copied to `$(RepoRoot).github/instructions/` before each local build and **always
  overwritten** when they differ from the source (size or timestamp) — local edits to the copy are discarded. Never
  edit the copies; change them here. A file removed from build.net is **not** deleted in the consumer — remove the
  stale copy by hand.
- **Commit the copy** in the consumer repo. Copilot on github.com (coding agent, PR review) reads
  `.github/instructions/` directly from the repository and never builds, so only a committed copy reaches it.
- Skipped on CI (`CI=true`, set by GitHub Actions).
- A failed copy (e.g. a sharing violation during parallel builds) is retried and then reported as a warning; it never
  breaks the build.

### Warnings

| Code | Meaning |
|---|---|
| `BUILDNET001` | `build.net.targets` was found, but the shared `*.instructions.md` files are missing (incomplete checkout). |

## CI drift check (recommended)

Local builds keep the committed copy in sync, but nothing forces a contributor to build before committing. Add a job
to the consumer's workflow that fails when the copy drifts from the submodule.

The snippet below is a **GitHub Actions** job (`::error` is a GitHub workflow command). On other CI systems, run the
same `git diff --no-index --exit-code` comparison as a build step; a non-zero exit code means the copy has drifted.

```yaml
instructions-drift:
  name: Copilot Instructions Up To Date
  runs-on: ubuntu-latest
  steps:
  - name: Checkout Repository
    uses: actions/checkout@v7
    with:
      submodules: true

  - name: Compare shared instructions with build.net
    run: |
      failed=0
      for src in build.net/.github/instructions/*.instructions.md; do
        dest=".github/instructions/$(basename "$src")"
        if ! git diff --no-index --exit-code "$src" "$dest"; then
          echo "::error file=$dest::Copy differs from build.net. Build locally to resync (never edit the copy; change build.net instead) and commit the result."
          failed=1
        fi
      done
      exit $failed
```