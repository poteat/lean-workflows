# lean-workflows

Reusable GitHub Actions workflows for Lean 4 packages.

Two workflows, both invoked via `workflow_call`:

- **`ci.yml`** — checkout, install the pinned toolchain via
  `leanprover/lean-action`, build, then run `lake test`.
- **`bump-toolchain.yml`** — resolve the latest `leanprover/lean4` release,
  update `lean-toolchain`, build, run `lake test`, then commit+push and
  (optionally) tag `vX.Y.Z[-rcN]`.

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

## Inputs

| Workflow         | Input          | Default         | Notes                                              |
| ---------------- | -------------- | --------------- | -------------------------------------------------- |
| both             | `test-command` | `lake test`     | Shell command to validate after build. `''` skips. |
| both             | `runs-on`      | `ubuntu-latest` | Override runner image.                             |
| `bump-toolchain` | `tag-release`  | `true`          | Push `vX.Y.Z[-rcN]` tag for Reservoir to pick up.  |

## Repo settings required

For the bump workflow to push commits and tags using `GITHUB_TOKEN`:

> Settings → Actions → General → Workflow permissions → **Read and write
> permissions**

If the consuming repo enforces branch protection on `main`, the bot will need a
bypass or the wrapper must be adapted to open a PR instead.
