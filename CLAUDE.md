# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **GitHub MCP Server** - a Model Context Protocol server that connects AI tools to GitHub's platform. It enables AI agents to manage repositories, issues, pull requests, workflows, and more through natural language.

- **Language:** Go 1.24+
- **Framework:** modelcontextprotocol/go-sdk for MCP protocol
- **GitHub API:** google/go-github v79 (REST) + shurcooL/githubv4 (GraphQL)
- **Entry Point:** `cmd/github-mcp-server/main.go` - stdio MCP server (PRIMARY FOCUS)

## High-Level Architecture

```
cmd/github-mcp-server/  → CLI entry point (cobra/viper)
internal/ghmcp/         → Core server logic, GitHub client creation
pkg/github/             → 50+ MCP tools organized by domain (60+ Go files):
                        ├── tools.go, inventory.go, server.go (core registration)
                        ├── repositories.go, issues.go, pullrequests.go
                        ├── actions.go, code_scanning.go, dependabot.go
                        ├── search.go, git.go, gists.go, discussions.go
                        ├── notifications.go, projects.go, labels.go
                        ├── resources.go, prompts.go, scope_filter.go
                        └── __toolsnaps__/ (JSON schema snapshots)
pkg/inventory/          → Tool/resource/prompt registration system (builder pattern)
pkg/                    → Shared utilities: errors, sanitize, scopes, log, lockdown
internal/toolsnaps/     → Tool schema snapshot validation
```

The server creates GitHub clients (REST + GraphQL + Raw), registers toolsets via the inventory builder pattern, and handles stdio transport for MCP protocol.

**Key architectural pattern:** `pkg/` is public API (used as library by remote server), `internal/` is private to local server. Export functions (capitalize) if potentially used externally.

## Required Pre-Commit Commands

**Before committing, ALWAYS run in this order:**

```bash
script/lint              # gofmt -s -w . && golangci-lint (~1s)
script/test              # go test -race ./... (~1s cached)
```

**If modifying MCP tools/toolsets, ALSO run:**

```bash
UPDATE_TOOLSNAPS=true go test ./...   # Update tool schema snapshots
script/generate-docs                  # Update README.md
```

Commit the updated `.snap` files from `pkg/github/__toolsnaps__/` along with your changes.

## Common Development Commands

```bash
# Build
go build -v ./cmd/github-mcp-server

# Run server (requires GITHUB_PERSONAL_ACCESS_TOKEN)
./github-mcp-server stdio

# Run specific test
go test ./pkg/github -run TestGetMe

# Tool search CLI
./github-mcp-server tool-search "issue" --max-results 5
```

## Environment Variables

- `GITHUB_PERSONAL_ACCESS_TOKEN` - Required authentication
- `GITHUB_HOST` - GitHub Enterprise hostname (prefix with `https://` for GHES)
- `GITHUB_TOOLSETS` - Comma-separated toolset list (overrides --toolsets flag)
- `GITHUB_TOOLS` - Specific tools to enable (additive with toolsets)
- `GITHUB_READ_ONLY` - Set to "1" for read-only mode
- `GITHUB_LOCKDOWN_MODE` - Set to "1" to restrict public repository content access
- `GITHUB_DYNAMIC_TOOLSETS` - Set to "1" for runtime toolset discovery
- `UPDATE_TOOLSNAPS` - Set to "true" when updating snapshots

## Code Patterns

- **Toolsnaps:** Every MCP tool has a JSON schema snapshot in `pkg/github/__toolsnaps__/*.snap` - these document API changes and must be updated when schemas change
- **Inventory Builder:** Tools are registered via `pkg/inventory` builder pattern with chainable methods (`SetTools()`, `WithToolsets()`, `WithReadOnly()`)
- **Feature Flags:** `FeatureFlags` struct includes `LockdownMode` and `InsiderMode` for runtime behavior adjustment
- **Dependency Injection:** `BaseDeps` struct injected via context, not closures (performance-critical for 50+ tools)
- **Testing:** Use testify, mock with go-github-mock (REST) or githubv4mock (GraphQL)
- **Library usage:** This repo is used as a library by the remote server - export functions (capitalize) if potentially used externally
- **Naming:** Use `ID`, `API`, `URL`, `HTTP` (not `Id`, `Api`, `Url`, `Http`)

## CI/CD

Located in `.github/workflows/`:
- `go.yml` - Build/test on ubuntu/windows/macos
- `lint.yml` - golangci-lint
- `docs-check.yml` - Verifies README.md is up-to-date
- `license-check.yml` - Validates license compliance
- `docker-publish.yml` - Publishes to ghcr.io
- `goreleaser.yml` - Creates releases (main branch only)

All must pass for PR merge.

## Key Files

- `/internal/ghmcp/server.go` - Server factory, creates GitHub clients (REST/GraphQL/Raw)
- `/pkg/github/server.go` - MCP server wrapper, provides helpers for tools
- `/pkg/github/tools.go` - `AllTools()` returns all tool definitions
- `/pkg/github/inventory.go` - `NewBuilder()` for tool registration with filtering
- `/go.mod` - Go 1.24.0 dependencies
- `/.golangci.yml` - Linter config (15+ linters enabled)
- `/Dockerfile` - Multi-stage container build
- `/.github/copilot-instructions.md` - Detailed AI assistant guidelines

## Secondary Package

`cmd/mcpcurl/` is a testing utility for MCP debugging. Don't break it, but it's not the primary focus.
