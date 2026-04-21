# API Observe Modes

When `verify_toolkit = api`, the subagent verifies acceptance criteria by writing integration tests AND making live HTTP requests against the running service.

## Toolkit classification keywords

Task title or acceptance criteria containing any of these → `verify_toolkit = api`:

```
endpoint, returns JSON, HTTP, status code, request body, response,
REST, GraphQL, POST, GET, PUT, DELETE, PATCH, 201, 400, 401, 403, 404, 500,
API, header, content-type, payload, schema, webhook
```

## Target URL determination

- If `project_structure = docker`: use the Traefik URL from Phase 3 (e.g., `http://ai-share.localhost`)
- If `project_structure = standalone`: start the server locally using the start command (from Makefile `dev` target, package.json `start`/`dev` script, or plan instructions). Use `http://localhost:<port>` where port is from server startup output, project config, or acceptance criteria URL.

## Per-iteration observe pass

### 1. Write integration test (first iteration only)

- Detect the project's test framework from codebase signals:
  - `Cargo.toml` with `[dev-dependencies]` → `cargo test`
  - `go.mod` → `go test ./...`
  - `package.json` with jest/vitest/mocha → `npm test`
  - `pyproject.toml` or `pytest.ini` → `pytest`
- Write a test file that exercises the endpoint per the acceptance criteria.
- The test file should:
  - Start or connect to the running server
  - Make the HTTP request described in criteria
  - Assert on status code, response body fields, headers as specified
- Test file persists as a project artifact (path: `tests/api/` or project-conventional test dir).
- Run the test. If it fails, the implementation is wrong — fix code and re-run.

### 2. Live hit (every iteration after code changes)

After code changes rebuild/restart, make the actual HTTP request against the live service:

```bash
curl -s -w "\n%{http_code}" -X <METHOD> <URL> \
  -H "Content-Type: application/json" \
  -d '<body>' 
```

Capture:
- HTTP status code
- Response headers (relevant ones per criteria)
- Response body (full)
- Parse body as JSON if Content-Type indicates JSON

Compare each field against the acceptance criteria.

### 3. Evidence

Save to scratch dir as `task-N-api-evidence.json`:

```json
{
  "request": {
    "method": "POST",
    "url": "http://ai-share.localhost/api/users",
    "headers": {"Content-Type": "application/json"},
    "body": {"name": "test", "email": "test@example.com"}
  },
  "response": {
    "status": 201,
    "headers": {"Content-Type": "application/json"},
    "body": {"id": "uuid-here", "name": "test", "email": "test@example.com"}
  },
  "test_result": "pass (3/3 assertions)",
  "test_file": "tests/api/test_users.rs"
}
```

## Pass criteria

ALL of these must be true:
- Integration tests pass
- Live hit response matches every assertion in the acceptance criteria (status, body fields, headers)
- No server errors in logs (check `mx logs` for docker, or server stderr for standalone)

## Failure diagnosis

- **Live hit fails, tests pass:** wiring issue — route not registered, middleware blocking, CORS. Check server logs.
- **Tests fail, live hit also fails:** implementation bug — the code is wrong.
- **Connection refused:** server not running or port wrong. Check startup.
- **Timeout:** server hung. Check logs for deadlock or blocking operation.

## observe_mode

API tasks do NOT use static/interactive sub-classification. The observe pass is always the same: test + live hit.
