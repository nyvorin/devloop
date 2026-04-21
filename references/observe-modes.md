# Observe Modes

The orchestrator classifies each plan task into one of two observe modes by keyword match on the task title. The subagent uses this hint to decide how thoroughly to probe.

## Static mode

**Triggered when:** the task is about layout, styling, content rendering, or component visual structure.

**Subagent does, every iteration:**

1. `browser_navigate` to the target URL (or the sub-route the task scopes to).
2. `browser_snapshot` — DOM accessibility tree.
3. `browser_console_messages` — capture errors and warnings.
4. `browser_network_requests` — capture failed (4xx/5xx) requests.
5. Compare snapshot against the task's acceptance criteria.

**Pass criteria:** every checklist item is observable in the snapshot **AND** zero console errors **AND** zero failed network requests.

## Interactive mode

**Triggered when:** the task title (lowercased) contains any of these keywords:

```
form, submit, login, click, navigate, flow, auth,
input, select, checkbox, button, upload, search
```

**Subagent does, every iteration:**

1. Static-mode pass first (above).
2. Then exercise the user flow described in the task acceptance criteria:
   - `browser_fill_form` for inputs
   - `browser_click` for actions
   - `browser_wait_for` for state changes
3. Capture state after each interaction (snapshot + console + network).
4. Verify the expected post-action state matches criteria.

**Pass criteria:** static criteria **AND** post-flow state matches expectation **AND** no errors thrown during interaction.

## Mode classification rule (orchestrator)

Lowercase the task title, check against the interactive keyword list above. Match → `interactive`. No match → `static`. The orchestrator passes `observe_mode` in the subagent prompt.

When in doubt, default to `static`. The subagent is allowed to escalate to interactive within an iteration if criteria explicitly mention user actions, but it must log the escalation in its return summary.
