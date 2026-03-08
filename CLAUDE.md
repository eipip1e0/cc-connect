# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

cc-connect is a Go application that bridges AI coding assistants (Claude Code, Codex, Cursor, Gemini, Qoder, OpenCode) to messaging platforms (Feishu, DingTalk, Slack, Telegram, Discord, LINE, WeChat Work, QQ). It enables users to interact with local AI agents from chat apps.

## Build Commands

```bash
# Build the binary
make build

# Build and run
make run

# Run tests (unit tests only; integration tests require env flags)
make test

# Run linter
make lint

# Build release binaries for all platforms
make release-all

# Build for specific platform
make release TARGET=linux/amd64

# Clean build artifacts
make clean
```

## Running Tests

```bash
# Run all unit tests
go test -v ./...

# Run integration tests (requires respective CLI installed)
QODER_INTEGRATION=1 go test -v ./agent/qoder/...
```

## Architecture

### Core Abstractions (`core/`)

The codebase uses a plugin-based architecture with factory registration:

- **`core.Platform`** interface (`core/interfaces.go:9-15`): Messaging platform adapters implement this. Key methods: `Start(handler MessageHandler)`, `Reply()`, `Send()`, `Stop()`.
- **`core.Agent`** interface (`core/interfaces.go:103-110`): AI agent adapters implement this. Key methods: `StartSession()`, `ListSessions()`, `Stop()`.
- **`core.Engine`** (`core/engine.go`): Routes messages between platforms and agents, handles slash commands, session management.
- **Registration pattern**: Agents and platforms register themselves via `init()` functions using `core.RegisterAgent()` and `core.RegisterPlatform()` (see `core/registry.go`).

### Project Structure

```
cmd/cc-connect/    # Entrypoint with agent/platform imports
agent/             # Agent implementations (claudecode, codex, cursor, gemini, qoder, opencode)
platform/          # Platform implementations (feishu, dingtalk, telegram, slack, discord, line, wecom, qq)
core/              # Core abstractions and engine
core/engine.go     # Main message routing logic (~1700 lines)
core/session.go    # Session management
core/cron.go       # Scheduled task support
config/            # TOML config loading and persistence
daemon/            # System daemon/service support
```

### Adding New Components

**New Agent**: Implement `core.Agent` interface, register in `init()`:
```go
func init() { core.RegisterAgent("myagent", New) }
```
Then add blank import in `cmd/cc-connect/main.go`.

**New Platform**: Implement `core.Platform` interface, register similarly.

### Key Implementation Patterns

1. **Agent Sessions**: Agents spawn persistent processes (e.g., `claude --input-format stream-json`) and communicate via stdin/stdout. See `agent/claudecode/session.go` for the canonical implementation.

2. **Event-Driven Architecture**: Agent sessions emit `core.Event` types (Text, ToolUse, Result, Error, PermissionRequest) consumed by the engine.

3. **Optional Interfaces**: Agents/platforms implement optional capability interfaces:
   - `ProviderSwitcher`: Runtime API provider switching
   - `ModelSwitcher`: Runtime model switching
   - `ModeSwitcher`: Permission mode switching
   - `HistoryProvider`: Retrieve conversation history
   - `ToolAuthorizer`: Pre-allow specific tools
   - `MemoryFileProvider`: Agent instruction files (CLAUDE.md, etc.)
   - `TypingIndicator`, `MessageUpdater`, `InlineButtonSender`: Platform UI features

4. **Config Persistence**: `config/` package handles atomic read-modify-write with mutex protection for runtime config changes.

## Configuration

TOML-based config at `~/.cc-connect/config.toml` or `./config.toml`:

```toml
[[projects]]
name = "my-project"

[projects.agent]
type = "claudecode"  # or codex, cursor, gemini, qoder, opencode

[projects.agent.options]
work_dir = "/path/to/project"
mode = "default"  # permission mode

[[projects.platforms]]
type = "feishu"  # or telegram, slack, discord, etc.

[projects.platforms.options]
# Platform-specific credentials
```

## Development Notes

- **Go version**: 1.24.2 (specified in `go.mod`)
- **No external build dependencies** beyond Go toolchain
- **Integration tests** require environment flags (e.g., `QODER_INTEGRATION=1`) and the respective CLI tool installed
- **Session storage**: Stored in `~/.cc-connect/sessions/` as JSON files
- **Log levels**: Controlled via config `[log] level = "info"` (debug, info, warn, error)

## Common File Locations

- Agent implementations: `agent/<name>/<name>.go` and `session.go`
- Platform implementations: `platform/<name>/<name>.go`
- Engine routing logic: `core/engine.go`
- Config structures: `config/config.go`
- CLI entrypoint: `cmd/cc-connect/main.go`
