---
name: go-mockery
description: Generate mocks for Go interfaces using mockery v3. Use when creating mocks, configuring .mockery.yaml, setting up mock generation for interfaces in Go projects, writing tests with mockery-generated mocks and testify (EXPECT, mock.Anything, RunAndReturn), or migrating from mockery v2 to v3. Not for gomock, counterfeiter, or hand-written mocks.
---

# Mockery v3 — Go Mock Generation

Generate type-safe mocks for Go interfaces using [mockery](https://github.com/vektra/mockery) v3 with `stretchr/testify/mock`. Covers mockery v3.x (v3.6.0+ for `//mockery:generate` directives). Check version: `mockery --version`.

## Convention: mocks/ as Child Directory

Always place generated mocks in a `mocks/` subdirectory of the package defining the interface:

```
internal/
├── repo/
│   ├── user.go              # defines UserRepo interface
│   └── mocks/
│       └── mock_UserRepo.go
├── service/
│   ├── order.go             # defines OrderService interface
│   └── mocks/
│       └── mock_OrderService.go
```

In hexagonal architecture, `mocks/` lives under `ports/`, `in/`, `out/`:

```
app/
├── ports/
│   ├── repository.go        # defines Repository interface
│   └── mocks/
│       └── mock_Repository.go
├── in/
│   ├── service.go           # defines input port interfaces
│   └── mocks/
│       └── mock_Service.go
└── out/
    ├── gateway.go           # defines output adapter interfaces
    └── mocks/
        └── mock_Gateway.go
```

The config that produces this layout:

```yaml
dir: "{{.InterfaceDir}}/mocks"
pkgname: "mocks"
filename: "mock_{{.InterfaceName}}.go"
```

## Generating .mockery.yaml

When a project needs mockery config, generate a `.mockery.yaml` at the repo root. Start from this base and adapt to the project's package paths:

```yaml
all: false
template: testify
formatter: goimports
dir: "{{.InterfaceDir}}/mocks"
pkgname: "mocks"
filename: "mock_{{.InterfaceName}}.go"
structname: "Mock{{.InterfaceName}}"

packages:
  github.com/org/repo/internal/ports:
    config:
      all: true
```

### Adjusting for the project

- Replace the package path with actual module paths from `go.mod`
- Add multiple package entries if interfaces live in several packages
- Use `all: true` per-package to mock every interface, or list specific interfaces
- Use `recursive: true` if a package tree has interfaces at multiple depths

## Configuration Reference

### Top-Level Parameters

| Parameter | Templated | Default | Description |
|---|---|---|---|
| `all` | No | `false` | Mock all interfaces in specified packages |
| `dir` | Yes | `"{{.InterfaceDir}}"` | Output directory for mock files |
| `filename` | Yes | `"mocks_test.go"` | Output filename (we override to `"mock_{{.InterfaceName}}.go"`) |
| `pkgname` | Yes | `"{{.SrcPackageName}}"` | Package name for generated files |
| `structname` | Yes | `"{{.Mock}}{{.InterfaceName}}"` | Name of generated mock struct |
| `template` | No | `"testify"` | Template: `testify`, `matryer`, `file://`, `https://` |
| `formatter` | No | `"goimports"` | Code formatter: `goimports`, `gofmt`, `noop` |
| `force-file-write` | No | `true` | Overwrite existing files |
| `recursive` | No | `false` | Recursively discover sub-packages |
| `log-level` | No | `"info"` | Logger verbosity |
| `include-interface-regex` | No | `""` | Only mock matching interfaces |
| `exclude-interface-regex` | No | `""` | Exclude matching interfaces (requires `include-interface-regex`) |
| `exclude-subpkg-regex` | No | `[]` | Exclude sub-packages when `recursive: true` |
| `build-tags` | No | `""` | Additional build tags |
| `inpackage` | No | `nil` | Override auto-detection of same-package placement |
| `include-auto-generated` | No | `false` | Parse auto-generated source files |
| `template-data` | No | `{}` | Arbitrary options passed to the template |
| `replace-type` | No | `{}` | Type replacement mappings |
| `_anchors` | No | `{}` | YAML anchors for DRY config (ignored by mockery) |

### Template Variables

Available in any templated parameter (`dir`, `filename`, `pkgname`, `structname`):

| Variable | Description |
|---|---|
| `{{.InterfaceName}}` | Name of the interface |
| `{{.InterfaceDir}}` | Absolute directory of the source interface |
| `{{.InterfaceDirRelative}}` | Interface directory relative to working directory |
| `{{.InterfaceFile}}` | File where the interface is defined |
| `{{.Mock}}` | `"Mock"` if exported, `"mock"` if unexported |
| `{{.MockName}}` | Resolved mock struct name |
| `{{.SrcPackageName}}` | Source `package` name |
| `{{.SrcPackagePath}}` | Fully qualified package path |
| `{{.ConfigDir}}` | Directory of the config file |
| `{{.Template}}` | Template name |

Template functions: `camelcase`, `snakecase`, `kebabcase`, `lower`, `upper`, `firstLower`, `firstUpper`, `trimPrefix`, `trimSuffix`, `replace`, `replaceAll`, `split`, `join`, `contains`, `hasPrefix`, `hasSuffix`, `matchString`, `base`, `dir`, `expandEnv`, `getenv`.

### Template-Data Keys (testify)

| Key | Type | Description |
|---|---|---|
| `with-expecter` | bool | Expecter structs (always true in v3, listed for v2 compat reference) |
| `unroll-variadic` | bool | Expand variadic args |
| `boilerplate-file` | string | Path to header comment file |
| `mock-build-tags` | string | Build tags for generated mocks |

### Configuration Hierarchy

Config merges across three levels — lower overrides higher:

```yaml
# Level 1: Top-level defaults
dir: "{{.InterfaceDir}}/mocks"
pkgname: "mocks"

packages:
  # Level 2: Package-level overrides
  github.com/org/repo/internal/ports:
    config:
      all: true
    interfaces:
      # Level 3: Interface-level overrides
      Repository:
        config:
          structname: "MockRepo"
```

Use `config:` (singular) for one mock per interface. Use `configs:` (plural, a list) for multiple mock variants:

```yaml
interfaces:
  Handler:
    configs:
      - structname: "MockHandlerStrict"
        template-data:
          unroll-variadic: true
      - structname: "MockHandlerRelaxed"
```

### Interface Selection

| Method | When to use |
|---|---|
| `all: true` on a package | Mock everything in the package |
| Explicit `interfaces:` list | Cherry-pick specific interfaces |
| `include-interface-regex` | Pattern-match interface names |
| `recursive: true` | Discover interfaces in sub-packages |
| `//mockery:generate` directive | Per-interface control via doc comments |

The `//mockery:generate` directive (v3.6.0+) goes in the interface's doc comment:

```go
// Repository handles data persistence.
//
//mockery:generate: true
//mockery:structname: MockRepo
type Repository interface {
    Find(ctx context.Context, id string) (*Entity, error)
}
```

### replace-type (v3 format)

```yaml
replace-type:
  github.com/original/pkg:
    OriginalType:
      pkg-path: github.com/replacement/pkg
      type-name: ReplacementType
```

### YAML Anchors for DRY Config

```yaml
_anchors:
  mocks-config: &mocks-config
    dir: "{{.InterfaceDir}}/mocks"
    pkgname: "mocks"
    filename: "mock_{{.InterfaceName}}.go"

packages:
  github.com/org/repo/internal/ports:
    config:
      <<: *mocks-config
      all: true
  github.com/org/repo/internal/adapters/out:
    config:
      <<: *mocks-config
      all: true
```

## Running Mockery

```bash
# Generate all configured mocks
mockery

# With debug logging
mockery --log-level=debug

# Bootstrap config for a new project
mockery init

# Migrate v2 config to v3
mockery migrate --config .mockery_v2.yaml
```

Add a `//go:generate` directive in the file that defines your interfaces:

```go
//go:generate mockery
package ports
```

Then run with `go generate ./...`.

## Usage Examples

These are reference examples, not prescribed patterns. Use whatever fits the test.

### Basic expectation

```go
func TestGetUser(t *testing.T) {
    repo := mocks.NewMockRepository(t)
    repo.EXPECT().
        FindByID(mock.Anything, "user-123").
        Return(&User{ID: "user-123", Name: "Alice"}, nil).
        Once()

    svc := NewUserService(repo)
    user, err := svc.GetUser(context.Background(), "user-123")

    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

### Dynamic return values

```go
repo.EXPECT().
    FindByID(mock.Anything, mock.AnythingOfType("string")).
    RunAndReturn(func(ctx context.Context, id string) (*User, error) {
        return &User{ID: id, Name: "User " + id}, nil
    })
```

### Side effects with Run

```go
var captured string
repo.EXPECT().
    Save(mock.Anything, mock.Anything).
    Run(func(ctx context.Context, u *User) {
        captured = u.ID
    }).
    Return(nil)
```

### Call frequency

```go
repo.EXPECT().FindByID(mock.Anything, "id").Return(&User{}, nil).Once()
repo.EXPECT().FindByID(mock.Anything, "id").Return(&User{}, nil).Times(3)
repo.EXPECT().FindAll(mock.Anything).Return([]*User{}, nil).Twice()
```

### Error case

```go
repo.EXPECT().
    FindByID(mock.Anything, "missing").
    Return(nil, ErrNotFound).
    Once()
```

### Ordered expectations

```go
repo.EXPECT().Save(mock.Anything, mock.Anything).Return(nil).Once()
repo.EXPECT().FindByID(mock.Anything, "id").Return(&User{ID: "id"}, nil).Once()
```

### Argument matchers

```go
mock.Anything                          // matches any value
mock.AnythingOfType("string")          // matches by type name
mock.MatchedBy(func(u *User) bool {    // custom matcher
    return u.Age > 18
})
```

## v2 to v3 Migration Notes

Key changes when upgrading:

| v2 | v3 |
|---|---|
| `with-expecter: true` (top-level) | Always on for testify; remove it |
| `unroll-variadic` (top-level) | Moved to `template-data.unroll-variadic` |
| `tags` | Moved to `template-data.mock-build-tags` |
| `replace-type` string format | Key-value map format |
| `keeptree` | Removed |
| Separate `mocks/` dir default | Adjacent to interface by default |

Run `mockery migrate` to auto-convert config, then fix items in the deprecation table it outputs.

## Troubleshooting

| Problem | Fix |
|---|---|
| "interface not found" | Run `mockery --log-level=debug`. Check package path matches `go.mod`. For external packages, `go get` them first. |
| Import cycle | Mocks are in a separate `mocks/` package — this shouldn't happen. If it does, check `inpackage` isn't set. |
| Stale mocks | Re-run `mockery` after changing interfaces. Wire into `go generate`. |
| Wrong package name | Check `pkgname` template resolves correctly. Use `--log-level=debug`. |
