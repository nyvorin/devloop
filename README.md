# devloop

Build with eyes on — a Claude Code skill that executes implementation plans with live verification.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## What it does

Devloop takes a design spec and implementation plan (produced by Claude's brainstorming and planning skills) and executes the plan task by task — with a subagent that can **see the results** after each change. It dispatches one isolated subagent per task, each equipped with the right verification toolkit:

| Toolkit | Verifies | How |
|---|---|---|
| **Playwright** | Web UI | DOM snapshots, console errors, network failures, click/fill interactions |
| **XcodeBuildMCP** | iOS/macOS apps | Simulator screenshots, tap/swipe automation, build output |
| **API** | HTTP endpoints | Integration tests + live curl against running service |
| **CLI** | Command-line tools | Integration tests + live run + stdout/stderr/file-effect proof snapshots |

Verification toolkit is selected **per task** — a single plan can mix all four. The subagent iterates (build → observe → fix) until acceptance criteria pass, with stuck detection (3 identical failures) and a hard cap (10 iterations) to prevent runaway loops.

## Installation

### Option 1: Git clone (recommended)

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/nyvorin/devloop ~/.claude/skills/devloop

# Symlink the companion plan-writing skill
ln -s ~/.claude/skills/devloop/writing-devloop-plans ~/.claude/skills/writing-devloop-plans
```

### Option 2: Claude Code plugin

```bash
claude plugin install nyvorin/devloop
```

### Verify installation

Start a new Claude Code session and check that `devloop` and `writing-devloop-plans` appear in your available skills.

## Prerequisites

Devloop auto-detects your project structure and selects the right tools. Install only what you need:

| Project type | Detection | Requirements |
|---|---|---|
| **Docker/mx** (web apps, APIs) | `mx.toml` or `compose.yml` | [mx CLI](https://github.com/mech-crate/mx), Docker |
| **Xcode** (iOS/macOS) | `*.xcodeproj` / `*.xcworkspace` | Xcode, [XcodeBuildMCP](https://xcodebuildmcp.com) |
| **Standalone** (Rust, Go, Node, Python) | `Cargo.toml`, `go.mod`, `package.json`, etc. | Language toolchain only |

**For web UI verification:** Install the Playwright MCP plugin in Claude Code.

**For iOS verification:** Add XcodeBuildMCP:
```bash
claude mcp add -s user XcodeBuildMCP -- npx -y xcodebuildmcp@latest mcp
```

**For API/CLI verification:** No MCP needed — uses curl and bash.

## Quick start

The workflow is three steps:

### 1. Design (brainstorming)

```
You: Let's build a REST API with user registration and login
Claude: [runs brainstorming skill, asks questions, produces design spec]
```

### 2. Plan (writing-devloop-plans)

```
You: Write the devloop plan
Claude: [runs writing-devloop-plans skill, produces plan with per-task acceptance criteria]
```

### 3. Build (devloop)

```
You: /devloop
Claude: [stands up environment, dispatches subagents per task, verifies via Playwright/API/CLI, reports results]
```

Devloop reads the most recent spec and plan from `docs/superpowers/specs/` and `docs/superpowers/plans/`, brings up your dev environment, and works through each task. When a task passes, its checkboxes flip to `[x]` in the plan file. If you abort mid-run, re-running `/devloop` resumes from the first unchecked task.

## How it works

```
Phase 1: Detect project structure (docker / xcode / standalone)
Phase 2: Stand up environment (mx dev / xcodebuild / cargo build)
Phase 3: Discover target (Traefik URL / simulator / localhost:port)
Phase 4: For each plan task:
           → Classify verify toolkit (playwright / xcodebuildmcp / api / cli)
           → Dispatch subagent
           → Subagent: implement → verify → iterate until pass (or stuck)
           → Mark task complete in plan file
Phase 5: Final acceptance pass across all completed tasks
```

The orchestrator (SKILL.md) runs in your main session and stays lean — all iteration noise (screenshots, DOM dumps, curl output, fix attempts) lives in the subagent's isolated context. You see clean per-task pass/fail results.

## Verification toolkits

### Playwright (web UI)

Activated by keywords: `page`, `element`, `h1`, `click`, `navigate`, `DOM`, `renders`, `form`

The subagent navigates to your app's URL, takes DOM snapshots, checks console for errors, checks network for failures. In interactive mode, it clicks links, fills forms, and verifies post-action state.

### XcodeBuildMCP (iOS/macOS)

Activated by keywords: `simulator`, `tap`, `swipe`, `screen`, `iOS`, `view`, `UIKit`, `SwiftUI`

The subagent builds and runs in the simulator, takes screenshots, and uses UI automation (tap, swipe) for interactive verification.

### API (HTTP endpoints)

Activated by keywords: `endpoint`, `returns JSON`, `HTTP`, `status code`, `POST`, `GET`, `response`

The subagent writes integration tests using your project's test framework AND makes a live curl request against the running service. Both must pass. Evidence includes the full request/response pair.

### CLI (command-line tools)

Activated by keywords: `command`, `stdout`, `stderr`, `exit code`, `creates file`, `flag`, `subcommand`

The subagent writes integration tests AND runs the binary live, capturing stdout, stderr, exit code, and file effects as a proof snapshot.

### Explicit override

If auto-detection picks the wrong toolkit, add this to your task block:

```markdown
**Verify via:** api
```

## Project structures

| Structure | Detected by | Standup | Target |
|---|---|---|---|
| `docker` | `mx.toml`, compose files | `mx router up` + `mx dev` | Traefik URL (e.g., `http://myapp.localhost`) |
| `xcode` | `*.xcodeproj` / `*.xcworkspace` | `simulator/build` | Running simulator app |
| `standalone` | `Cargo.toml`, `go.mod`, `package.json`, `pyproject.toml` | `cargo build` / `go build` / `npm run build` | `localhost:port` (API) or binary path (CLI) |

## Stuck detection and limits

- **3 identical failures** → subagent returns `stuck`, orchestrator asks you: skip / retry / abort
- **10 iterations max** per task → hard cap, returns `fail`
- **Resumable** — plan file tracks progress via checkboxes. Abort and re-run anytime.

## Writing devloop-compatible plans

Use the included `writing-devloop-plans` skill instead of plain `writing-plans`. It adds a **per-task Acceptance Criteria** section that devloop's subagents use for verification:

```markdown
### Task 3: Add /api/users endpoint

**Acceptance Criteria (observable):**
- POST /api/users with valid JSON body returns 201
- Response body contains `id`, `email`, `created_at` fields
- POST with missing email returns 400 with error message
- No server errors in logs

**Verify via:** api

**Files:**
- Create: `src/handlers/users.rs`
...
```

If your plan doesn't have explicit acceptance criteria, devloop's subagent derives them at runtime from the spec + task description. Explicit criteria are more reliable.

## Contributing

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Open a PR

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
