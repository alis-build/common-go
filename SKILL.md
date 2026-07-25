---
name: migrate-to-go-alis-build-common
description: Migrate Go code from github.com/alis-build/public-go (or the legacy go.alis.build/common root module from alis-exchange/common-go) to the new go.alis.build/common/... vanity module paths, and fix related module-resolution errors.
---

# Migrate to go.alis.build/common import paths

This repository (`github.com/alis-build/common-go`, previously named
`github.com/alis-build/public-go`) publishes the public Go stubs generated from
Alis Build's public protocol buffers. As of July 2026 the canonical import
paths are **vanity paths under `go.alis.build/common`**:

| Before | After |
|---|---|
| `github.com/alis-build/public-go/alis/open/iam` | `go.alis.build/common/alis/open/iam` |
| `github.com/alis-build/public-go/alis/open/validation` | `go.alis.build/common/alis/open/validation` |
| `github.com/alis-build/public-go/<pkg-path>` | `go.alis.build/common/<pkg-path>` |

Key facts:

- **One Go module per proto package.** `go.alis.build/common/alis/open/iam`,
  `go.alis.build/common/alis/open/validation`, etc. are each their own module
  with their own versions (tags like `alis/open/iam/v1.7.0`). There is no
  single umbrella module to require.
- **Old paths are frozen, not broken.** Versions already published as
  `github.com/alis-build/public-go/...` keep resolving (GitHub redirects the
  renamed repo), but **no new versions** will appear under that path. All new
  versions are published only under `go.alis.build/common/...`.
- **The legacy `go.alis.build/common` root module is a different thing.** It
  was a single module (from `alis-exchange/common-go`) with root tags like
  `v0.x.y`. It is deprecated; already-pinned versions keep resolving through
  the Go module proxy cache, but do not add new requires on it. Replace its
  packages with the per-package modules above.

## How to migrate a Go module

1. Find affected imports and requires:

   ```sh
   grep -rn "github.com/alis-build/public-go" --include="*.go" --include=go.mod .
   grep -rn '"go.alis.build/common"' go.mod   # legacy root module, if present
   ```

2. Rewrite the import prefix in `.go` files and `go.mod`:

   ```sh
   # macOS sed shown; drop the '' on Linux
   grep -rl "github.com/alis-build/public-go" --include="*.go" --include=go.mod . \
     | xargs sed -i '' 's|github.com/alis-build/public-go|go.alis.build/common|g'
   ```

3. Pin each rewritten require to a version that exists under the new module
   path — the **first version published after the rename** or later. Check with:

   ```sh
   go list -m -versions go.alis.build/common/alis/open/iam
   ```

   Old versions listed for the new path that predate the rename will fail to
   download (their `go.mod` still declares the old path) — always pick the
   latest, or at minimum the first post-rename version.

4. Refresh dependencies:
   - On the Alis Build platform (service repos): run `alis packages install`
     — do not hand-roll `go mod tidy` with custom `GOPROXY` settings, the CLI
     configures the private registries for you.
   - Plain public modules elsewhere: `go mod tidy` is fine; the modules are
     public and served through the standard Go proxy and checksum database.

5. Build to verify: `go build ./...`.

Do not mix both prefixes for the same package in one module — the old and new
paths contain identical duplicate types (`protoreflect` full names collide at
runtime registration), so finish the rewrite module-wide.

## Fixing common errors

- **`module declares its path as: go.alis.build/common/... but was required as: github.com/alis-build/public-go/...`**
  You (or `go get -u`/`@latest`) pulled a post-rename version through the old
  path. Rewrite the import/require to `go.alis.build/common/...` (steps above).

- **`module declares its path as: github.com/alis-build/public-go/... but was required as: go.alis.build/common/...`**
  You pinned a pre-rename version on the new path. Bump to the latest version
  of that module.

- **`unknown revision` / `no matching versions` for `go.alis.build/common/<pkg>`**
  Either the package has not had a version published under the vanity path yet
  (trigger or wait for the next define of that proto package), or the path is
  wrong — module paths mirror the proto package with the trailing `.v1`
  dropped (e.g. proto `alis.open.iam.v1` → module
  `go.alis.build/common/alis/open/iam`; `v2+` majors keep their suffix:
  `alis.open.agent.v2` → `go.alis.build/common/alis/open/agent/v2`).

- **Ambiguous import / duplicate proto registration panics** (`proto: file
  ... is already registered`): both the old and new path of the same package
  are in the build graph. Find the stragglers with the greps in step 1 —
  including in your own dependencies — and finish the migration everywhere.
