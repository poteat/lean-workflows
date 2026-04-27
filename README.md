# lean-workflows

Reusable GitHub Actions workflows for Lean 4 packages.

Three workflows, all invoked via `workflow_call`:

- **`ci.yml`** — checkout, install the pinned toolchain via
  `leanprover/lean-action`, build, then run `lake test`.
- **`bump-toolchain.yml`** — resolve the latest `leanprover/lean4` release,
  update `lean-toolchain`, build, run `lake test`, then commit+push. After
  every run (bump or no-op), idempotently ensure a `v<toolchain>.0` tag exists
  for the currently-pinned toolchain — so a fresh package gets its first tag
  without manual bootstrap. Hotfix patches (`.1`, `.2`, …) stay manual.
- **`docs.yml`** — build API docs with `doc-gen4` via
  `leanprover-community/docgen-action` and deploy to GitHub Pages. The
  package's own `lakefile` stays free of any `doc-gen4` require; the action
  handles the inverted dependency in CI.

Both default to `lake test`, which runs whichever target is tagged
`@[test_driver]` in your `lakefile.lean`. Override `test-command` if you need
something else; pass `''` to skip.

## Usage

### 1. Mark your test target

In `lakefile.lean`:

```lean
@[test_driver]
lean_exe «my-smoke» where
  root := `MyPackageTest.Smoke
```

### 2. Add two thin wrappers under `.github/workflows/`

### `ci.yml`

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  ci:
    uses: poteat/lean-workflows/.github/workflows/ci.yml@main
```

### `bump-toolchain.yml`

```yaml
name: Bump Lean toolchain

on:
  schedule:
    # 18:00 UTC daily — Lean RCs typically publish 02:00–16:00 UTC,
    # so this catches new releases the same day.
    - cron: "0 18 * * *"
  workflow_dispatch:

jobs:
  bump:
    permissions:
      contents: write
    uses: poteat/lean-workflows/.github/workflows/bump-toolchain.yml@main
```

### `docs.yml`

```yaml
name: Docs

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  docs:
    permissions:
      contents: read
      pages: write
      id-token: write
    uses: poteat/lean-workflows/.github/workflows/docs.yml@main
    with:
      package-name: my-package    # the `package «...»` name in the lakefile
      docs-targets: MyPackage:docs # one or more `<Target>:docs` facets
```

Before the first run, enable Pages on the repo:
**Settings → Pages → Source: GitHub Actions**.

Docs land at `https://<owner>.github.io/<repo>/docs` (or, if the user-site
has a custom domain, `https://<custom>/<repo>/docs`).

Unlike `leanprover-community/docgen-action`, this workflow accepts both
`lakefile.toml` and `lakefile.lean` packages — the throwaway `docbuild/`
lakefile is generated from the inputs above rather than parsed from the
project's lakefile.

## Inputs

| Workflow                | Input            | Default         | Notes                                                                          |
| ----------------------- | ---------------- | --------------- | ------------------------------------------------------------------------------ |
| `ci`, `bump-toolchain`  | `test-command`   | `lake test`     | Shell command to validate after build. `''` skips.                             |
| all                     | `runs-on`        | `ubuntu-latest` | Override runner image.                                                         |
| `bump-toolchain`        | `tag-release`    | `true`          | Idempotently push `v<toolchain>.0` for Reservoir.                              |
| `docs`                  | `package-name`   | _required_      | Lake package name (from `package «...»` in the lakefile).                      |
| `docs`                  | `docs-targets`   | _required_      | Space-separated `:docs` facets to build (e.g. `MyLib:docs`).                   |
| `docs`                  | `landing-page`   | `''`            | If set (e.g. `MyLib.html`), redirects `/docs/` to this page (skips the austere `index.html`). |
| `docs`                  | `header-title`   | `''`            | Replaces "Documentation" in the page header. Accepts HTML (e.g. a breadcrumb with `<a>`). |
| `docs`                  | `hidden-modules` | `Init Lake Lean Std` | Top-level modules to strip from the sidebar (HTML files kept so cross-refs resolve). `''` shows everything. |

## Repo settings required

For the bump workflow to push commits and tags using `GITHUB_TOKEN`:

> Settings → Actions → General → Workflow permissions → **Read and write
> permissions**

If the consuming repo enforces branch protection on `main`, the bot will need a
bypass or the wrapper must be adapted to open a PR instead.
