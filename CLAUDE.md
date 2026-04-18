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
deadcode -test=true \             # Find unreachable internal functions
  -filter='github.com/severity1/claude-agent-sdk-go/internal/...' \
  ./examples/... ./internal/...

# Makefile targets (recommended)
make check                        # Run all checks (fmt, vet, lint, cyclo, deadcode, fuzz-test)
make cyclo                        # Show complex functions (threshold: 15)
make cyclo-check                  # Fail if complexity exceeds threshold (CI)
make deadcode                     # Show unreachable internal functions
make deadcode-check               # Fail if unreachable internal functions exist (CI)
make fuzz-test                    # Verify fuzz corpus (fast, CI mode)
make fmt-check                    # Verify code formatting
make security                     # Run security vulnerability checks
make sdk-test                     # Test SDK as consumer would use it
make release-check                # Pre-release validation
make ci                           # Run full CI pipeline locally (includes fuzz + deadcode)
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
- **Dead code**: Run `make deadcode` (golang.org/x/tools/cmd/deadcode) before commits. Scope is `internal/*` rooted at `./examples/...` (examples are the SDK's only `main` entrypoints); the public API surface is excluded since it's reachable by external consumers, not us. Remove unreachable internal helpers instead of suppressing; if intentionally reserved (rare), add an example or test that exercises the symbol
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
- **ThinkingConfig union type**: `ThinkingConfig` is an interface with three implementations: `ThinkingConfigAdaptive` (model decides budget), `ThinkingConfigEnabled{BudgetTokens: N}` (explicit budget), `ThinkingConfigDisabled`; use `WithThinking()`, `WithThinkingAdaptive()`, `WithThinkingBudget(N)`, `WithThinkingDisabled()` helpers; all three variants implement `MarshalJSON` emitting a Python-SDK-compatible `"type"` discriminator (private constants `thinkingConfigTypeAdaptive/Enabled/Disabled`); `ThinkingConfigEnabled.MarshalJSON` also emits `budget_tokens`
- **Forward-compat raw types**: `RawMessage` and `RawContentBlock` in `internal/shared` wrap unknown message/content-block types returned by the parser; `BlockType()` reads from `RawBlockType` field (not `BlockType_` - underscore suffix rejected by linter)
- **Hook event constants**: 10 total - PreToolUse, PostToolUse, PostToolUseFailure, UserPromptSubmit, Stop, SubagentStop, PreCompact, Notification, SubagentStart, PermissionRequest; new events added in Phase 1 #2/#4 (Python PRs #535/#545)
- **Hook agent fields**: PreToolUseHookInput, PostToolUseHookInput now include ToolUseID/AgentID/AgentType; SubagentStopHookInput includes AgentID/AgentTranscriptPath/AgentType; helper functions in hooks.go: getBoolPtr, getSlice, getMap (returns empty map not nil when key absent - matches Python SDK dict behavior, safe for callbacks that mutate the returned map)
- **Hook output fields**: PreToolUseHookSpecificOutput has AdditionalContext + PermissionDecision + UpdatedInput; PostToolUseHookSpecificOutput has AdditionalContext + UpdatedMCPToolOutput (rewrites MCP tool output sent to Claude; `any` field with `omitempty` only drops nil - zero-value scalars like `""`, `0`, `false` are still serialized, so leave nil to omit); PermissionRequestHookSpecificOutput has Decision map[string]any (permission decision payload, replaces AdditionalContext)
- **HookJSONOutput response**: sendHookResponse serializes all HookJSONOutput fields (Continue, SuppressOutput, StopReason, Decision, SystemMessage, Reason, HookSpecificOutput) as omitempty; PermissionRequestHookInput includes PermissionSuggestions []string parsed via getSlice
- **Unknown control request subtypes**: protocol sends error response via sendErrorResponse() instead of silently ignoring - preserves forward compat (no crash) while completing the protocol roundtrip with a meaningful failure
- **AgentDefinition.Model validation**: Options.Validate() checks agent model values (must be sonnet/opus/haiku/inherit/empty) to catch typos before they reach the CLI
- **MCP config file FD**: temp file is closed immediately after write/sync; CLI reads by path; keeping handle open leaked FD for subprocess lifetime and blocked file open on Windows
- **RewindFiles request**: `RewindFilesRequest{SubtypeRewindFiles, UserMessageID}` rewinds file state to a specific user message UUID; UserMessage.UUID is the source for UserMessageID
- **RawControlMessage routing**: parser returns `&shared.RawControlMessage{MessageType, Data}` for both control_request and control_response message types; these bypass ParseMessage and go to the control protocol handler directly
- **Sandbox options**: `SandboxSettings` configures bash command isolation (Enabled, AutoAllowBashIfSandboxed, ExcludedCommands, Network *SandboxNetworkConfig, IgnoreViolations); use `WithSandbox()`, `WithSandboxEnabled()`, `WithSandboxExcludedCommands()`, `WithSandboxNetwork()`
- **OutputFormat type**: `OutputFormat{Type: "json_schema", Schema: map[string]any}` specifies structured JSON output format for the Messages API; set via `Options.OutputFormat`; use `OutputFormatTypeJSONSchema` constant (defined in `internal/shared/options.go`, re-exported in `types.go`)
- **AgentModel constants**: sonnet/opus/haiku/inherit (inherit = use parent model); set on `AgentDefinition.Model`; validated in Options.Validate()
- **PermissionResult no-callback default**: `handleCanUseToolRequest` returns `NewPermissionResultDeny("no permission callback registered")` when no CanUseToolCallback is set - secure-deny default
- **GetMcpStatus flow**: `Client.GetMcpStatus()` -> `subprocess.Transport.GetMcpStatus()` -> `control.Protocol.GetMcpStatus()` -> `GetMcpStatusRequest{SubtypeGetMcpStatus}` -> CLI; response marshal/unmarshal to `*McpStatusResponse{McpServers []McpServerStatus}`; connection status values: connected/failed/needs-auth/pending/disabled
- **McpServerStatus.Config field**: `map[string]any` preserving wire-format server configuration (URL for HTTP/SSE servers, type/name for SDK servers); mirrors Python SDK McpServerStatusConfig union (Python PR #516); typed union deferred to a later parity phase
- **Agents-in-initialize**: Agents sent via `InitializeRequest.Agents` (not `--agents` CLI flag); `WithOptions()` ProtocolOption passes `shared.Options` to Protocol; Initialize() builds agents map with description/prompt/tools/model fields per agent
- **McpToolInfo vs McpToolDefinition**: `McpToolInfo` (control/types.go) reports tools from connected servers in status responses; `shared.McpToolDefinition` defines tools for SDK MCP servers - these are distinct types for distinct purposes
- **UserMessage optional fields**: `UUID *string` and `ParentToolUseID *string` are optional top-level fields on `shared.UserMessage` (Issue #24); both nil when absent, pointer values when present
- **AssistantMessage.Error parsing**: `Error *string` field on `shared.AssistantMessage` is parsed from the top-level JSON object (not from nested `message` data); values: `AssistantMessageErrorRateLimit` ("rate_limit"), `AssistantMessageErrorAuthFailed` ("authentication_failed"), `AssistantMessageErrorUnknown` ("unknown") for unrecognized values; helpers: `HasError()`, `IsRateLimited()`; parser uses typed constants, never raw strings
- **Mock transport functional options**: `clientMockTransport` in client_test.go uses `WithClientXxx()` functional options (e.g. `WithClientMcpStatus`, `WithClientMcpStatusError`) to configure per-test behavior; all mock state access protected by mutex

<!-- END AUTO-MANAGED -->

<!-- AUTO-MANAGED: git-insights -->
## Git Insights

- Conventional commit messages: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`
- Issue references in commits: `(Issue #N)` or `(#N)`, use `Closes #N` in PR body
- PR-based workflow with CI checks
- Recent focus: Phase 1 parity complete via Go PR #117 - all 8 items ported (GetMcpStatus, PostToolUseFailure hook, AssistantMessage error fix, hook events, McpToolAnnotations, agents-in-initialize, ThinkingConfig, RawMessage/RawContentBlock); Phase 1 code review fixes landed (dead code removal: WithTransport Option, HookRegistration, generateHookRegistrations; bug fixes: MCP config FD leak, permission callback type mismatch now errors loudly, unknown control subtypes send error response, AgentDefinition.Model validation in Options.Validate()); Phase 1 review follow-up (feature/phase1-parser-fixes): M1 fix - ThinkingConfig variants now emit Python-SDK "type" discriminator via MarshalJSON; M2 fix - UpdatedMCPToolOutput zero-scalar foot-gun documented; minor: AssistantMessageErrorUnknown constant in parser, OutputFormatTypeJSONSchema re-exported in types.go, SA1019 nolint on MaxThinkingTokens legacy paths; next up: Phase 2 items #9-#20
- Benchmark organization: Table-driven benchmarks across all core modules (options, parser, shared, control, cli)
- Makefile integration: All code quality checks (fmt, vet, lint, cyclo, deadcode, fuzz-test) unified under `make check`; `make ci` runs full pipeline including fuzz corpus verification and deadcode enforcement
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
