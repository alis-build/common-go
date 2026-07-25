# common-go

Public Go packages generated from public protocol buffers on Alis Build.

Import these modules via their canonical vanity paths under
**`go.alis.build/common`** — one module per proto package, e.g.:

```go
import (
    openIam "go.alis.build/common/alis/open/iam"
    openValidation "go.alis.build/common/alis/open/validation"
)
```

```sh
go get go.alis.build/common/alis/open/iam@latest
```

> This repository was previously named `public-go` and its modules were
> imported as `github.com/alis-build/public-go/...`. Those paths are frozen —
> new versions are published only under `go.alis.build/common/...`.
> See [SKILL.md](SKILL.md) for a migration guide your coding agent can follow.
