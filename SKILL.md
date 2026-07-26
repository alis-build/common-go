---
name: migrate-to-go-alis-build-common
description: Migrate Go code from github.com/alis-build/public-go (or the legacy go.alis.build/common root module / open.alis.services/protobuf copies) to the go.alis.build/common vanity module paths, and fix the resulting duplicate-proto-registration panics, mixed-stub type mismatches, and module-resolution errors.
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
  with their own versions (tags like `alis/open/iam/v1.8.0`). There is no
  single umbrella module to require.
- **Old paths are frozen, not broken.** Versions already published as
  `github.com/alis-build/public-go/...` keep resolving (GitHub redirects the
  renamed repo), but **no new versions** will appear under that path. All new
  versions are published only under `go.alis.build/common/...`.
- **Both paths contain identical proto types.** The generated `.pb.go` files
  register the same proto file names (e.g. `alis/open/iam/iam.proto`). If one
  binary links BOTH copies — even one directly and one through a dependency —
  it panics at startup:
  `panic: proto: file "..." is already registered`. Migration is therefore
  all-or-nothing per binary, including everything in its dependency graph.
- **Two legacy lineages are deprecated**, and both collide with the vanity
  copies the same way:
  - the old `go.alis.build/common` *root module* (from `alis-exchange/common-go`,
    root tags `v0.x.y`); pinned versions keep resolving via the Go proxy cache;
  - the `open.alis.services/protobuf/...` copies from the old artifact
    pipeline (still imported by some shared libraries — see the last section).

## Migrating a plain Go module

1. Find affected imports and requires:

   ```sh
   grep -rn "github.com/alis-build/public-go" --include="*.go" --include=go.mod .
   ```

2. Rewrite the import prefix in `.go` files and `go.mod`:

   ```sh
   # macOS sed shown; drop the '' on Linux
   grep -rl "github.com/alis-build/public-go" --include="*.go" --include=go.mod . \
     | xargs sed -i '' 's|github.com/alis-build/public-go|go.alis.build/common|g'
   ```

3. Pin each rewritten require to a **post-rename version** (check with
   `go list -m -versions go.alis.build/common/alis/open/iam`; when in doubt use
   `@latest`). Pre-rename versions listed under the new path fail to download —
   their committed `go.mod` still declares the old path. The reverse also
   holds: after the first vanity publish, `@latest` on the OLD path resolves to
   a new tag and fails with a module-path mismatch, because both paths share
   one tag namespace.

4. `go mod tidy`, then `go build ./...`. If tidy fails on an unrelated
   `google.golang.org/genproto` split error pulled in by a dependency's tests,
   skip tidy — `go get` the modules explicitly and verify by building.

5. **Verify the whole graph, not just your code.** Both of these must return
   nothing (the second covers test binaries):

   ```sh
   go list -deps ./... | grep github.com/alis-build/public-go
   go list -test -deps ./... | grep github.com/alis-build/public-go
   ```

   A cheap runtime probe that triggers all proto registrations:
   `go test -run NONE -count=1 .` — it must not panic.

## Migrating an Alis Build service (product repo)

Services on the Alis Build platform rarely import these stubs only directly —
they also consume **generated `alis.build/<org>/<product>/<neuron>` modules**
from the private registry, whose own `.pb.go` files import whichever path was
current when that package was last defined. Old generated modules therefore
drag the old copies back into your binary even after you rewrite your imports.

Symptoms of a half-migrated graph:

- startup/test panic: `proto: file "..." is already registered — previously
  from "github.com/alis-build/public-go/...", currently from
  "go.alis.build/common/..."`;
- compile error where your handler type "does not implement" the generated
  server interface — `have ...(*"github.com/alis-build/public-go/..."` vs
  `want ...(*"go.alis.build/common/..."` — your code and the regenerated
  stubs disagree on which copy of a request type they mean.

Recipe (order matters):

1. **Re-define every stale package first.** Any `alis.build/...` module in
   your graph whose generated code still imports the old path must be
   regenerated: run `alis define <package-id>` for it (no proto changes
   needed). Find the stale set by checking each `alis.build/...` module
   version in `go list -deps -f '{{if .Module}}{{.Module.Path}} {{.Module.Version}}{{end}}' ./...`
   for `public-go` imports in its module-cache source. Defines of one
   package can expose the next stale one (deps of deps) — iterate until the
   scan is clean. If common packages themselves are re-defined, do leaves
   before dependents (e.g. `validation`/`options` before `iam`) or the
   dependent's fresh stubs will still reference an old-path leaf.
2. **`alis packages upgrade --all` BEFORE hand-editing anything.**
   `alis packages install`/`upgrade` re-sync `go.mod` server-side and will
   revert hand edits — CLI first, edits after. If an upgrade run immediately
   follows a define, it can resolve a stale "latest"; re-run it.
3. Rewrite your own imports (sed sweep above) and `go get` the vanity pins.
4. Bump any remaining stale `alis.build/...` pins. If direct fetches fail
   (private registry), use the module cache:
   `GOPRIVATE='alis.build/*' GOFLAGS=-mod=mod GOPROXY=off go get alis.build/...@<version>`.
   Generated modules have an empty `require ()`, so add their vanity
   requirements first (step 3) or cache-only resolution can't discover them.
5. Verify with the graph checks and probe from the previous section —
   including the `-test` variant: stale pins sometimes hide in test-only
   dependencies.

## Fixing common errors

- **`module declares its path as: go.alis.build/common/... but was required as: github.com/alis-build/public-go/...`**
  A post-rename version was pulled through the old path (often via
  `go get -u`, `@latest`, or `go mod tidy` inventing a requirement). Rewrite
  the import/require to `go.alis.build/common/...`.

- **`module declares its path as: github.com/alis-build/public-go/... but was required as: go.alis.build/common/...`**
  A pre-rename version pinned on the new path. Bump to the latest version.

- **`panic: proto: file ... is already registered`** — two copies of the same
  proto package in one binary. Read the panic: it names both import paths.
  Use `go mod why <old-path-package>` to find which dependency drags the old
  copy in, then re-define/bump that dependency (see the service recipe).
  If the old copy arrives via `open.alis.services/protobuf/...`, the culprit
  is a shared library still on the legacy artifact pipeline (at the time of
  writing, `go.alis.build/authz` / `go.alis.build/testing`) — if it only
  enters through test helpers your deployed binary is unaffected, but
  `go test` will panic until that library migrates.

- **`unknown revision` / `no matching versions` for `go.alis.build/common/<pkg>`**
  Either the package has no post-rename version yet (define it), or the path
  is wrong — module paths mirror the proto package with the trailing `.v1`
  dropped (proto `alis.open.iam.v1` → `go.alis.build/common/alis/open/iam`;
  `v2+` majors keep their suffix: `alis.open.agent.v2` → `.../alis/open/agent/v2`).

- **sum.golang.org 404/500 on a freshly tagged version** — the checksum
  database is fetching or briefly cached a failed lookup; retry after a
  minute or two.
