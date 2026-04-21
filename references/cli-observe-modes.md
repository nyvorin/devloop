# CLI Observe Modes

When `verify_toolkit = cli`, the subagent verifies acceptance criteria by writing integration tests AND running the binary live, capturing stdout/stderr/exit code and file effects as proof.

## Toolkit classification keywords

Task title or acceptance criteria containing any of these → `verify_toolkit = cli`:

```
command, stdout, stderr, exit code, outputs, creates file, flag, argument,
subcommand, binary, prints, writes to, reads from, pipe, CLI, terminal,
--help, --version, parse, input file, output file
```

## How to run the binary

- If `project_structure = docker`: use `mx run <service> -- <command>` or `docker exec <container> <command>`
- If `project_structure = standalone`: build locally first, then run the binary directly:
  - Rust: `cargo build && ./target/debug/<binary>`
  - Go: `go build -o ./<binary> && ./<binary>`
  - Node: `npm run build && node dist/<entrypoint>` (or `npx ts-node src/<entrypoint>`)
  - Python: `python <script>` (no build step)

## Per-iteration observe pass

### 1. Write integration test (first iteration only)

- Detect the project's test framework and CLI testing pattern:
  - Rust: `assert_cmd` crate (add to dev-dependencies if missing), use `Command::cargo_bin`
  - Go: `os/exec` in `_test.go` files
  - Node: `child_process` in jest/vitest test files
  - Python: `subprocess` in pytest test files
- Write a test that:
  - Invokes the binary as a subprocess with the exact args from criteria
  - Captures stdout, stderr, exit code
  - Asserts on expected output and file effects
- Test file persists as a project artifact.
- Run the test. If it fails, fix code and re-run.

### 2. Live run (every iteration after code changes)

After code changes and rebuild, run the exact command from the acceptance criteria:

```bash
<command> <args> 2>stderr.tmp; echo "EXIT:$?" 
```

Capture:
- `stdout` (full output)
- `stderr` (full output)
- `exit_code`
- File effects: check for files created/modified in expected locations. Capture contents (first 500 chars if large).

### 3. Proof snapshot

Save to scratch dir as `task-N-cli-proof.json`:

```json
{
  "command": "mytool export --format json input.csv",
  "exit_code": 0,
  "stdout": "Exported 42 records to output.json\n",
  "stderr": "",
  "files_created": ["output.json"],
  "file_samples": {
    "output.json": "[{\"id\":1,\"name\":\"Alice\"},{\"id\":2,\"name\":\"Bob\"}]"
  },
  "test_result": "pass (4/4 assertions)",
  "test_file": "tests/cli/test_export.rs"
}
```

## Pass criteria

ALL of these must be true:
- Integration tests pass
- Live run exit code matches expected (usually 0 for success, non-zero for expected errors)
- stdout content matches acceptance criteria assertions
- stderr has no unexpected errors (expected stderr like progress bars is fine if criteria say so)
- File effects match (files created/modified with expected content)

## Failure diagnosis

- **Live run fails, tests pass:** stale binary (not rebuilt after code change). Rebuild and retry.
- **Wrong output:** logic bug. Check the code path for the specific subcommand/flag.
- **Exit code non-zero unexpectedly:** unhandled error. Check stderr for the error message.
- **File not created:** check the output path — might be relative vs absolute, or permissions.
- **Binary not found:** build step didn't produce it at expected path. Check build output.

## observe_mode

CLI tasks do NOT use static/interactive sub-classification. The observe pass is always: test + live run + proof snapshot.
