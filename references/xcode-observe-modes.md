# Xcode Observe Modes

iOS/macOS equivalent of `observe-modes.md`. The orchestrator classifies each plan task into one of two observe modes by keyword match on the task title. The subagent uses this hint to decide how thoroughly to probe the running simulator app.

## Static mode

**Triggered when:** the task is about layout, styling, content rendering, view structure, or data display.

**Subagent does, every iteration:**

1. `simulator/build-and-run` — build the project and launch in the simulator. If already running and no code changed since last build, skip this step.
2. `simulator/screenshot` — capture the current screen.
3. Analyze the screenshot against the task's acceptance criteria — check for expected UI elements, layout, text content.
4. Check build output for warnings or errors.
5. If the app crashed, capture crash info via debugging tools.

**Pass criteria:** every checklist item is observable in the screenshot **AND** the build succeeded with no errors **AND** the app is not crashing.

## Interactive mode

**Triggered when:** the task title (lowercased) contains any of these keywords:

```
tap, swipe, button, form, input, submit, login, navigate,
flow, auth, gesture, scroll, press, toggle, picker, alert,
modal, sheet, push, transition, action
```

**Subagent does, every iteration:**

1. Static-mode pass first (above).
2. Then exercise the user flow described in the task acceptance criteria:
   - `ui-automation/tap` for button presses and element selection
   - `ui-automation/swipe` for scroll, dismiss, or navigation gestures
   - Text entry via tapping text fields and typing
3. `simulator/screenshot` after each interaction to capture state transitions.
4. Verify the expected post-action state matches criteria (e.g., "after tapping Login, the Home screen appears").

**Pass criteria:** static criteria **AND** post-flow state matches expectation **AND** no crashes during interaction.

## Mode classification rule (orchestrator)

Same logic as the web path: lowercase the task title, check against the interactive keyword list above. Match → `interactive`. No match → `static`. The orchestrator passes `observe_mode` in the subagent prompt.

When in doubt, default to `static`. The subagent is allowed to escalate to interactive within an iteration if criteria explicitly mention user actions, but it must log the escalation in its return summary.

## Key differences from web observe modes

| Aspect | Web (Playwright) | iOS (XcodeBuildMCP) |
|---|---|---|
| Build step | Not needed (hot-reload handles it) | Required — `simulator/build-and-run` before each observe pass if code changed |
| DOM snapshot | `browser_snapshot` (accessibility tree) | Screenshot analysis (no DOM equivalent) |
| Console errors | `browser_console_messages` | Build warnings/errors + crash logs |
| Network failures | `browser_network_requests` | Not directly observable; check for timeout/crash symptoms |
| Click/interact | `browser_click` / `browser_fill_form` | `ui-automation/tap` / `ui-automation/swipe` |
| Wait for state | `browser_wait_for` | Screenshot polling (take screenshot, check, retry) |
