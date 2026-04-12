# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!-- AUTO-MANAGED: project-description -->
## Overview

**Claude Agent SDK for Go** - Unofficial Go SDK for Claude Code CLI integration. Provides programmatic interaction through `Query()` (one-shot) and `Client` (streaming) APIs with 100% Python SDK parity.

- **Module**: `github.com/severity1/claude-agent-sdk-go`
- **Package**: `claudecode`
- **Go Version**: 1.18+

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: build-commands -->
## Build & Development Commands

```bash
# Build and test
go build ./...                    # Build all packages
go test ./...                     # Run all tests
go test -race ./...               # Race condition detection
go test -cover ./...              # Coverage analysis
make test-cover                   # Tests with coverage + HTML report

# Specific test patterns
go test -v -run TestClient        # Run client tests (verbose)
go test -count=3 -run TestClient  # Run tests multiple times for consistency
make bench                        # Run benchmarks

# Code quality (run before commits)
go fmt ./...                      # Format code
go vet ./...                      # Static analysis
golangci-lint run                 # Comprehensive linting
gocyclo -over 15 .                # Cyclomatic complexity check

# Makefile targets (recommended)
make check                        # Run all checks (fmt, vet, lint, cyclo)
make cyclo                        # Show complex functions (threshold: 15)
make cyclo-check                  # Fail if complexity exceeds threshold (CI)
make fmt-check                    # Verify code formatting
make security                     # Run security vulnerability checks
make sdk-test                     # Test SDK as consumer would use it
make release-check                # Pre-release validation
make ci                           # Run full CI pipeline locally
```

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: architecture -->
## Architecture

```
.
├── client.go              # Client interface and WithClient context manager
├── query.go               # Query API (one-shot operations)
├── errors.go              # Structured error types
├── transport.go           # Transport interface abstraction
├── options.go             # Options types and functional options
├── options_bench_test.go  # Options performance benchmarks
├── internal/
│   ├── cli/               # CLI discovery and command building
│   ├── control/           # Bidirectional control protocol (hooks, permissions, MCP)
│   ├── parser/            # JSON message parsing with speculative parsing
│   ├── shared/            # Shared types (Message, ContentBlock interfaces)
│   └── subprocess/        # Subprocess management and protocol adapter
├── examples/              # Usage examples (numbered by complexity)
└── docs/
    ├── architecture/      # Detailed architecture documentation
    └── tracking/          # Python SDK parity tracking (PR replay tracker)
```

**Data Flow**:
1. `Query()`/`Client` -> `Transport` interface -> `subprocess.Transport` -> Claude CLI
2. CLI stdout -> `parser.Parser` -> `shared.Message` types -> User code
3. Control protocol: `control.Protocol` <-> CLI (hooks, permissions, MCP)

**Documentation**: See ARCHITECTURE.md and CONTRIBUTING.md for comprehensive details on design patterns, interfaces, data flow, and contribution guidelines.

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: conventions -->
## Code Conventions

- **Idiomatic Go**: Use `gofmt` formatting, standard naming conventions
- **Interface-driven**: All message types implement `Message`, all content blocks implement `ContentBlock`
- **Error handling**: Use `fmt.Errorf` with `%w` verb for wrapping, include contextual information
- **Context-first**: All blocking functions accept `context.Context` as first parameter
- **JSON handling**: Custom `UnmarshalJSON` for union types, discriminate on `"type"` field
- **Cyclomatic complexity**: Keep functions under complexity 15 (measured by gocyclo); higher acceptable for table-driven tests, examples, orchestration code
- **Naming patterns**: Interfaces describe behavior, implementations use concrete names, options use `WithXxx()`, errors use `XxxError` suffix
- **No unnecessary exports**: Keep identifiers unexported unless needed by external consumers

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: patterns -->
## Detected Patterns

- **Transport interface**: Central abstraction for CLI communication; use `MockTransport` for tests
- **Process cleanup**: SIGTERM -> wait 5 seconds -> SIGKILL pattern
- **Buffer protection**: 1MB limit to prevent memory exhaustion
- **Environment variables**: Set `CLAUDE_CODE_ENTRYPOINT` to identify SDK to CLI
- **Table-driven tests**: Use for complex scenarios with multiple test cases
- **Functional options**: `WithXxx()` pattern for configuration
- **Benchmark tests**: Use `var sink any` to prevent dead code elimination, always call `b.ReportAllocs()` and `b.ResetTimer()`
- **tool_use_result metadata**: `UserMessage.ToolUseResult` carries rich edit info (filePath, structuredPatch, diffs); check with `HasToolUseResult()` before accessing via `GetToolUseResult()`
- **Init error routing**: `subprocess.routeInitError()` detects error `ResultMessage` arriving before transport is connected and calls `protocol.HandleControlInitErr()` to unblock `SendControlRequest()` via `initErrChan`
- **ThinkingConfig union type**: `ThinkingConfig` is an interface with three implementations: `ThinkingConfigAdaptive` (model decides budget), `ThinkingConfigEnabled{BudgetTokens: N}` (explicit budget), `ThinkingConfigDisabled`; use `WithThinking()`, `WithThinkingAdaptive()`, `WithThinkingBudget(N)`, `WithThinkingDisabled()` helpers
- **Forward-compat raw types**: `RawMessage` and `RawContentBlock` in `internal/shared` wrap unknown message/content-block types returned by the parser; `BlockType()` reads from `RawBlockType` field (not `BlockType_` - underscore suffix rejected by linter)
- **Hook event constants**: 10 total - PreToolUse, PostToolUse, PostToolUseFailure, UserPromptSubmit, Stop, SubagentStop, PreCompact, Notification, SubagentStart, PermissionRequest; new events added in Phase 1 #2/#4 (Python PRs #535/#545)
- **Hook agent fields**: PreToolUseHookInput, PostToolUseHookInput now include ToolUseID/AgentID/AgentType; SubagentStopHookInput includes AgentID/AgentTranscriptPath/AgentType; getBoolPtr/getSlice added as helper functions in hooks.go
- **GetMcpStatus flow**: `Client.GetMcpStatus()` -> `subprocess.Transport.GetMcpStatus()` -> `control.Protocol.GetMcpStatus()` -> `GetMcpStatusRequest{SubtypeGetMcpStatus}` -> CLI; response marshal/unmarshal to `*McpStatusResponse{McpServers []McpServerStatus}`; connection status values: connected/failed/needs-auth/pending/disabled
- **Agents-in-initialize**: Agents sent via `InitializeRequest.Agents` (not `--agents` CLI flag); `WithOptions()` ProtocolOption passes `shared.Options` to Protocol; Initialize() builds agents map with description/prompt/tools/model fields per agent
- **McpToolInfo vs McpToolDefinition**: `McpToolInfo` (control/types.go) reports tools from connected servers in status responses; `shared.McpToolDefinition` defines tools for SDK MCP servers - these are distinct types for distinct purposes

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: git-insights -->
## Git Insights

- Conventional commit messages: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`
- Issue references in commits: `(Issue #N)` or `(#N)`, use `Closes #N` in PR body
- PR-based workflow with CI checks
- Recent focus: Phase 1 parity complete via Go PR #117 - all 8 items ported (GetMcpStatus, PostToolUseFailure hook, AssistantMessage error fix, hook events, McpToolAnnotations, agents-in-initialize, ThinkingConfig, RawMessage/RawContentBlock); next up: Phase 2 items #9-#20 (stop_reason on ResultMessage, MCP reconnect/toggle, typed task messages, list_sessions, per-turn usage on AssistantMessage, skills/memory/mcpServers on AgentDefinition, CLAUDE_CODE_ENTRYPOINT set-if-absent, etc.)
- Benchmark organization: Table-driven benchmarks across all core modules (options, parser, shared, control, cli)
- Makefile integration: All code quality checks (fmt, vet, lint, cyclo) unified under `make check`
- Python SDK parity tracking: `docs/tracking/README.md` tracks all Python SDK PRs to port; organized into 4 chronological phases (Phase 1: Jan 26-Feb 20, Phase 2: Mar 3-Mar 16, Phase 3: Mar 20-Mar 30, Phase 4: Mar 31-Apr 8); Phase 1 complete via Go PR #117 (Python PRs #516,#535,#506,#545,#551,#468,#565,#598); last ported features: Phase 1 items #1-#8 (Go PR #117), errors field on ResultMessage (Go PR #114, Python PR #749)

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: best-practices -->
## Best Practices

- **TDD approach**: Write failing tests first, implement to make them pass
- **Test file organization**: Test functions first, then mocks, then helpers
- **Helper functions**: Always call `t.Helper()` in test utilities
- **Thread safety**: All mocks must be thread-safe with proper mutex usage
- **Self-contained tests**: Each test file has its own helpers to avoid dependencies
- **Benchmark organization**: Use table-driven benchmarks with realistic scenarios, measure allocations with `b.ReportAllocs()`
- **t.Fatal() + return**: Always follow `t.Fatal()` with `return` in subtests to prevent staticcheck SA5011 nil pointer dereference warnings (staticcheck does not track that t.Fatal() stops execution)

<!-- END AUTO-MANAGED -->

<!-- MANUAL -->
## Custom Notes

Add project-specific notes here. This section is never auto-modified.

<!-- END MANUAL -->
