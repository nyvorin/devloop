# XcodeBuildMCP Quick Reference

Tools devloop uses for iOS/macOS app observation, with the MCP tool mapping.

## Tool naming convention

XcodeBuildMCP tools follow the pattern `mcp__XcodeBuildMCP__<tool_name>`. The exact names may vary by version. At the start of each subagent dispatch, run `ToolSearch` with query `"XcodeBuildMCP"` to discover the current tool names. The categories below are stable.

## Core tools by category

### Build & Run

| Tool | Purpose | Web equivalent |
|---|---|---|
| `simulator/build` | Compile the project for simulator | N/A (mx handles builds) |
| `simulator/build-and-run` | Build + launch in iOS Simulator | `browser_navigate` to URL |
| `simulator/test` | Run test suite in simulator | Running test commands via Bash |

### Observation

| Tool | Purpose | Web equivalent |
|---|---|---|
| `simulator/screenshot` | Capture current simulator screen | `browser_take_screenshot` |
| (accessibility snapshot via screenshot + analysis) | Inspect UI element tree | `browser_snapshot` |

### UI Automation (interactive mode)

| Tool | Purpose | Web equivalent |
|---|---|---|
| `ui-automation/tap` | Tap a UI element | `browser_click` |
| `ui-automation/swipe` | Swipe gesture | N/A (scroll equivalent) |
| (type text via tap + keyboard) | Enter text in fields | `browser_fill_form` |

### Debugging

| Tool | Purpose | Web equivalent |
|---|---|---|
| `debugging/attach` | Attach LLDB debugger to running app | N/A |
| `debugging/breakpoint` | Set/manage breakpoints | N/A |
| (variable inspection via LLDB) | Inspect runtime state | `browser_console_messages` |

## Detection

- **Is this an Xcode project?** Check for `*.xcodeproj` or `*.xcworkspace` in the project root (or one level deep for monorepo layouts).
- **Is XcodeBuildMCP available?** Run `ToolSearch` for `"XcodeBuildMCP"` — if any tools are returned, the MCP is connected.

## Build verification

After `simulator/build-and-run`:
- Check for build errors in the tool's return output. Build failures include compiler errors with file paths and line numbers.
- If the build fails, the subagent should read the error, fix the code, and rebuild — same loop as the web path but with build errors instead of console errors.
- A successful build + launch means the app is running in the simulator and ready for screenshot/interaction.

## Crash detection

- If the app crashes after launch, the XcodeBuildMCP tools will report it. The subagent should use `debugging/attach` or check crash logs to diagnose.
- Treat crashes the same as console errors in the web path — they block the "clean" pass criteria.

## Project configuration

XcodeBuildMCP supports a `.xcodebuildmcp/config.yaml` in the project root for:
- Default scheme and project path
- Simulator device name
- Enabled workflows

If this file doesn't exist, XcodeBuildMCP uses sensible defaults (first scheme found, default simulator).
