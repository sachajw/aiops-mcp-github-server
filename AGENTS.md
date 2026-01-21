# GitHub MCP Server - Agent Guidelines

This is a Go-based MCP server that connects AI tools to GitHub. Use this guide when making changes.

## Project Overview

- **Language:** Go 1.24+
- **Entry Point:** `cmd/github-mcp-server/main.go` (stdio MCP server - PRIMARY)
- **Framework:** modelcontextprotocol/go-sdk for MCP protocol, google/go-github for GitHub API
- **Structure:** `pkg/github/` (50+ MCP tools), `internal/ghmcp/` (core logic), `script/` (build tools)

## Required Commands Before Committing

**Always run in this order:**
```bash
script/lint              # gofmt -s -w . && golangci-lint (~1s)
script/test              # go test -race ./... (~1s cached)
```

**If modifying MCP tools/toolsets, ALSO run:**
```bash
UPDATE_TOOLSNAPS=true go test ./...   # Update tool schema snapshots
script/generate-docs                  # Update README.md
```

Commit the updated `.snap` files from `pkg/github/__toolsnaps__/` with your changes.

## Common Commands

```bash
# Build
go build -v ./cmd/github-mcp-server

# Run specific test
go test ./pkg/github -run TestGetMe

# Run specific package tests
go test ./pkg/github -v

# Run server (requires GITHUB_PERSONAL_ACCESS_TOKEN)
./github-mcp-server stdio
```

## Code Style Guidelines

### Formatting & Linting
- **Formatting:** `gofmt -s` (simplify flag required) - auto-run by `script/lint`
- **Linter:** golangci-lint with bodyclose, gocritic, gosec, makezero, misspell, nakedret, revive, errcheck, staticcheck, govet, ineffassign, unused
- **Exclusions:** third_party/, builtin/, examples/, generated code

### Import Organization
- Group stdlib imports first, then third-party
- Use blank line between groups
- Use aliases for clarity: `ghErrors "github.com/..."`, `buffer "github.com/..."`

### Naming Conventions
- **Acronyms:** Use `ID`, `API`, `URL`, `HTTP` (not `Id`, `Api`, `Url`, `Http`)
- Examples: `userID`, `getAPI`, `parseURL`, `HTTPClient`
- Functions: PascalCase for exported, camelCase for internal
- Variables: camelCase, avoid underscores

### Types & Errors
- Use struct field tags for JSON/other serialization
- Return errors explicitly, never wrap nil errors
- Use `fmt.Errorf` with `%w` for error wrapping
- Check errors immediately after function calls
- Use `github.Ptr()` for pointer conversions

### Testing Patterns
- Use table-driven tests for behavioral scenarios
- Use testify: `require` for critical checks, `assert` for non-blocking
- Mock GitHub API: `go-github-mock` (REST) or `githubv4mock` (GraphQL)
- Test files: `*_test.go` alongside implementation
- Test structure: 1) Verify tool snapshot, 2) Check critical properties, 3) Behavioral tests

### Function & Tool Definitions
- Tools use `NewTool()` helper with: Toolset, mcp.Tool, required scopes, handler function
- InputSchema: Use `jsonschema.Schema` with map[string]*Schema for Properties
- Required parameters in Required slice
- Handler signature: `func(ctx context.Context, deps ToolDependencies, _ *mcp.CallToolRequest, args map[string]any) (*mcp.CallToolResult, any, error)`
- Use `utils.NewToolResultError(err.Error())` for error responses
- Use `utils.NewToolResultErrorFromErr(msg, err)` for errors with context

### Constants & Comments
- Define constants for repeated strings (especially descriptions)
- Comment sparingly - prefer self-documenting code
- Comments only when: explaining WHY, not WHAT; complex algorithms; public API docs

### Error Handling Patterns
```go
// Get client with error
client, err := deps.GetClient(ctx)
if err != nil {
    return utils.NewToolResultErrorFromErr("failed to get GitHub client", err), nil, nil
}

// Required parameter check
owner, err := RequiredParam[string](args, "owner")
if err != nil {
    return utils.NewToolResultError(err.Error()), nil, nil
}

// Optional parameter with default
perPage := OptionalParam[int](args, "perPage", 30)
```

## Architecture Notes

- **Library Usage:** This repo is used as library by remote server - export functions (capitalize) if potentially used externally
- **Toolsnaps:** Every MCP tool has JSON schema snapshot in `pkg/github/__toolsnaps__/*.snap` - these document API changes
- **Toolsets:** Organized groups of tools (repos, issues, pull_requests, actions, etc.) - see `pkg/github/tools.go` for registration
- **Scope System:** All tools declare required GitHub OAuth scopes in `pkg/scopes/`

## Environment Variables

- `GITHUB_PERSONAL_ACCESS_TOKEN` - Required authentication
- `GITHUB_HOST` - GitHub Enterprise hostname
- `GITHUB_TOOLSETS` - Comma-separated toolset list
- `GITHUB_TOOLS` - Specific tools to enable
- `GITHUB_READ_ONLY` - Set to "1" for read-only mode
- `UPDATE_TOOLSNAPS` - Set to "true" when updating snapshots

## CI Requirements

All workflows in `.github/workflows/` must pass for PR merge:
- `go.yml` - Build/test on ubuntu/windows/macos
- `lint.yml` - golangci-lint
- `docs-check.yml` - Verifies README.md is up-to-date
- `license-check.yml` - Validates license compliance

## Common Pitfalls

- Forgot to run `UPDATE_TOOLSNAPS=true` after changing tool schema → CI fails with toolsnap diff
- Changed README manually without `script/generate-docs` → docs-check fails
- Used lowercase acronyms (Id, Api, Url) → linter errors
- Skipped `script/lint` before commit → lint.yml fails in CI
