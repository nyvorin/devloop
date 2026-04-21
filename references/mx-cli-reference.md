# mx CLI Quick Reference

Commands devloop uses, with the exact invocation and what to expect.

## Project lifecycle

| Command | Purpose | Expected output |
|---|---|---|
| `mx ps` | List running services | Table with service names + status |
| `mx dev` | Start in dev mode (hot-reload) | Streams startup logs; returns when all services report ready |
| `mx up` | Start in non-dev mode | Same shape as `mx dev`, no hot-reload |
| `mx down` | Stop and remove containers | "Services stopped" |
| `mx logs <service>` | Tail logs for a service | Streaming log output |
| `mx logs <service> 2>&1 \| tail -n 50` | Last 50 lines (mx has no `--tail` flag; pipe through `tail`) | Static dump |

## Router

| Command | Purpose | Expected output |
|---|---|---|
| `mx router status` | Is global Traefik running? | `running` / `stopped` / `not installed` |
| `mx router up` | Start global Traefik | "router started" |
| `mx router inspect` | Show router config + routes | YAML-ish dump including registered hosts |

## Detection

- **Is this an mx project?** Check for `mx.toml` in the project root (preferred). If absent, check for `compose.yml` containing the marker comment `# mx-managed` on any line.
- **Are services running?** `mx ps` exits 0 with a populated table. An empty table means nothing is running.
- **Did startup succeed?** After `mx dev` (or `mx up`), run `mx ps` — every service should show `Up` / `running`. If any show `Restarting` or `Exited`, dump `mx logs <service> 2>&1 | tail -n 50` and abort.

## Restart-loop detection (heuristic)

Poll `mx ps` every 2 seconds for 30 seconds. If any service transitions through `Restarting` ≥3 times, treat as a restart-loop, abort, and dump the last 50 log lines for that service.

## Preferred startup mode

Devloop prefers `mx dev` over `mx up` because hot-reload makes the iteration loop faster. Fall back to `mx up` only if `mx dev` is unavailable in the project (rare).
