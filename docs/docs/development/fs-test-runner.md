# Filesystem Test Runner Design

## Goal

Add one `cargo xtask` entry point for running filesystem functional tests and
performance tests from the workspace root.

Target user commands:

```bash
cargo xtask fs list
cargo xtask fs test rsext4
cargo xtask fs test rsext4 --functional
cargo xtask fs test rsext4 --perf
cargo xtask fs test all
```

The runner should make filesystem validation discoverable, repeatable, and easy
to extend to more filesystems without adding ad-hoc shell scripts.

## Non-Goals

- Do not replace existing `cargo xtask test`, `cargo xtask clippy`, ArceOS QEMU
  tests, or Axvisor tests.
- Do not require every filesystem to have performance tests immediately.
- Do not introduce a new benchmark framework in the first patch.
- Do not run physical board tests through this command.
- Do not change filesystem crate public APIs as part of adding the runner.

## CLI Shape

Add a top-level `fs` subcommand to `axbuild`, which is already invoked by
`cargo xtask`.

```text
cargo xtask fs list
cargo xtask fs test <FS>
```

`<FS>` accepts either a filesystem suite name, such as `rsext4`, or `all`.

### `fs list`

Lists all configured filesystem suites and shows whether each suite has
functional and performance commands.

Example output:

```text
filesystem suites from scripts/test/fs_suites.toml
- rsext4: functional, perf
- axfs: functional
```

### `fs test`

Runs tests for one suite or all suites.

```text
cargo xtask fs test rsext4
cargo xtask fs test rsext4 --functional
cargo xtask fs test rsext4 --perf
cargo xtask fs test all
```

Default mode runs both functional and performance commands that are configured
for the selected suite.

Flags:

```text
--functional  Run functional commands only.
--perf        Run performance commands only.
--list        Show the selected commands without executing them.
```

Validation rules:

- `--functional` and `--perf` may both be omitted. This means run both kinds.
- `--functional --perf` is accepted and is equivalent to the default mode.
- Unknown suite names produce a clear error.
- Selecting `--perf` for a suite without performance commands should fail with
  a clear message rather than silently passing.

## Configuration File

Use a small TOML file so adding a suite does not require editing Rust code.

Path:

```text
scripts/test/fs_suites.toml
```

Initial contents:

```toml
[[suite]]
name = "rsext4"
package = "rsext4"

functional = [
  ["cargo", "test", "-p", "rsext4"],
]

perf = [
  ["cargo", "test", "-p", "rsext4", "--test", "perf", "--", "--ignored", "--nocapture"],
]
```

Future suite example:

```toml
[[suite]]
name = "axfs"
package = "ax-fs"

functional = [
  ["cargo", "test", "-p", "ax-fs"],
  [
    "cargo", "xtask", "arceos", "test", "qemu",
    "--arch", "x86_64",
    "--test-group", "rust",
    "--package", "arceos-fs-shell",
  ],
]
```

The command arrays are intentionally explicit. This avoids shell parsing,
quoting differences, and accidental dependence on user shell behavior.

## Code Layout

Add one module:

```text
scripts/axbuild/src/fs_test.rs
```

Update:

```text
scripts/axbuild/src/lib.rs
```

The new module owns CLI args, config parsing, suite selection, command planning,
and command execution.

Suggested structures:

```rust
#[derive(clap::Args)]
pub(crate) struct FsArgs {
    #[command(subcommand)]
    command: FsCommand,
}

#[derive(clap::Subcommand)]
pub(crate) enum FsCommand {
    List,
    Test(FsTestArgs),
}

#[derive(clap::Args)]
pub(crate) struct FsTestArgs {
    suite: String,
    #[arg(long)]
    functional: bool,
    #[arg(long)]
    perf: bool,
    #[arg(long)]
    list: bool,
}

#[derive(Debug, serde::Deserialize)]
struct FsSuitesConfig {
    suite: Vec<FsSuite>,
}

#[derive(Debug, serde::Deserialize)]
struct FsSuite {
    name: String,
    package: Option<String>,
    #[serde(default)]
    functional: Vec<Vec<String>>,
    #[serde(default)]
    perf: Vec<Vec<String>>,
}
```

In `scripts/axbuild/src/lib.rs`:

```rust
mod fs_test;

#[derive(Subcommand)]
enum Commands {
    // ...
    /// Run filesystem functional and performance test suites
    Fs {
        #[command(subcommand)]
        command: fs_test::Command,
    },
}
```

The dispatch branch calls:

```rust
Commands::Fs { command } => fs_test::execute(command),
```

## Execution Model

The runner should execute commands from the workspace root.

Use the existing process helper style from `scripts/axbuild/src/support/process.rs`
where possible. If a command is not `cargo`, add a generic `run_status` helper
that prints the command and inherits stdout/stderr.

Pseudo-flow:

```text
load config
validate duplicate suite names
resolve selected suites
resolve selected kinds: functional, perf, or both
build a flat command plan
if --list: print plan and return
run commands in order
collect failures
print summary
fail if any command failed
```

Command plan item:

```rust
struct PlannedCommand {
    suite: String,
    kind: TestKind,
    argv: Vec<String>,
}

enum TestKind {
    Functional,
    Perf,
}
```

Example runtime output:

```text
running filesystem tests for 1 suite(s), 2 command(s)
[1/2] rsext4 functional: cargo test -p rsext4
ok: rsext4 functional
[2/2] rsext4 perf: cargo test -p rsext4 --test perf -- --ignored --nocapture
ok: rsext4 perf
all filesystem tests passed
```

Failure output should include all failed commands:

```text
filesystem tests failed for 1 command(s):
- rsext4 perf: cargo test -p rsext4 --test perf -- --ignored --nocapture
```

## First rsext4 Performance Test

Add a minimal ignored performance test:

```text
components/rsext4/tests/perf.rs
```

Recommended first benchmark:

- create an in-memory mock block device
- `mkfs`
- `mount`
- create one file
- write a fixed-size buffer, for example 16 MiB
- read the same file back
- print write/read throughput with `std::time::Instant`

Use plain ignored tests first:

```rust
#[test]
#[ignore = "performance test"]
fn sequential_write_read_perf() {
    // ...
}
```

This keeps the first implementation dependency-free. A later patch can add
Criterion or structured JSON output if needed.

## Functional Test Coverage

For `rsext4`, the first functional command should be:

```bash
cargo test -p rsext4
```

This already covers the existing integration tests under:

```text
components/rsext4/tests/
```

No new functional tests are required for the first runner patch.

## Unit Tests for the Runner

Add unit tests inside `fs_test.rs` for logic that does not spawn processes.

Recommended tests:

- parse a valid `fs_suites.toml`
- reject duplicate suite names
- reject empty command arrays
- `fs list` rendering includes configured test kinds
- selecting `rsext4 --functional` plans only functional commands
- selecting `rsext4 --perf` plans only perf commands
- selecting `rsext4` plans both kinds
- selecting `all` plans suites in TOML order
- selecting an unknown suite returns an error
- selecting `--perf` for a suite without perf commands returns an error

Use a fake runner for command execution tests:

```rust
trait CommandRunner {
    fn run(&mut self, workspace_root: &Path, argv: &[String]) -> anyhow::Result<bool>;
}
```

The production implementation runs real processes. Tests can record invocations
and return configured success/failure values.

## Validation Plan

After implementing the runner:

```bash
cargo fmt
cargo xtask clippy --package axbuild
cargo xtask fs list
cargo xtask fs test rsext4 --functional
cargo xtask fs test rsext4 --perf
```

If the performance test is intentionally long, keep the initial data size small
enough for local CI-like validation, for example 16 MiB or less.

## Incremental Implementation Steps

1. Add `scripts/test/fs_suites.toml` with the `rsext4` functional command only.
2. Add `scripts/axbuild/src/fs_test.rs` with `list`, config parsing, command
   planning, and fake-runner unit tests.
3. Wire `cargo xtask fs list` and `cargo xtask fs test rsext4 --functional`.
4. Add `components/rsext4/tests/perf.rs`.
5. Add the `rsext4` perf command to `fs_suites.toml`.
6. Validate `cargo xtask fs test rsext4` and `cargo xtask fs test all`.

This sequence keeps each patch reviewable and gives a working command after the
third step.

