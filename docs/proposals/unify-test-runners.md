# Unify test runners under `amc test`

> Status: proposed 2026-05-21. Step 1 (flags) shipped on this PR.
> Steps 2-5 are follow-ups, each its own PR.

## Why

The repo ships **2,372 lines of bash** in five `tests/run_*.sh` scripts
that all reinvent the same wheel — discovery, compile, run, parse
`[PASS]`/`[FAIL]`/`[SKIP]`, tally:

| Script | Lines | Scope |
|---|--:|---|
| `tests/run_tests.sh`          | 1383 | core compiler regressions, multifile, LSP probes, migrate prompt checks |
| `tests/run_stdlib_tests.sh`   | 655  | stdlib (Path, DateTime, Json, …) + PackageRegistry |
| `tests/run_amc_new_tests.sh`  | 145  | `amc new --template <…>` smoke tests |
| `tests/run_fmt_tests.sh`      | 132  | `amc fmt` idempotency + semantic round-trip |
| `tests/run_all_tests.sh`      |  57  | top-level orchestrator |

Five different invocations, five different output formats, none of
them runs on Windows-native (they all assume POSIX `find` / `gcc`
quoting). The compiler already ships `amc test` that does the same
job for `*_test.am` files — bringing the shell scripts under that
roof gets us one entry point, one runner, one CI surface.

## Goal

`amc test ./tests/` (or just `amc test` from the repo root) drives
**every check that today's `tests/run_*.sh` drive**. Then the
`.sh` files get deleted.

## Step 1 — Add `--filter` / `--ci` / `--list` to `amc test` (shipped)

The bash runners filter by name (`run_tests.sh some-grep-pattern`).
`amc test` had no such surface. This PR adds:

- `--filter <pat>` — substring match on test path; only matching
  tests are run.
- `--ci` — terse output: skip per-file headers, skip `[PASS]` /
  `[SKIP]` lines, keep `[FAIL]` + the final tally. Matches what
  the bash runners write to stdout in `--ci` mode.
- `--list` — discover + print the matching test paths; don't
  compile, don't run. Useful for test-explorer integrations
  (`amc test --list --filter Phase` → fast feedback in editors).

With those in place, `amc test ./tests/samples/ --filter closure`
matches the ergonomics shell developers already expect.

## Step 2 — Convert `run_amc_new_tests.sh` (~145 lines, smallest)

The smallest runner. Tests run `amc new <name> --template <kind>`
in fresh tmp dirs and assert the resulting layout. Convert to
`tests/integration/amc_new_test.am`:

```amalgame
class Program {
    public static void Main() {
        Test.scaffoldExe("exe-smoke")
        Test.scaffoldLib("lib-smoke")
        Test.scaffoldTest("test-smoke")
        Test.scaffoldService("service-smoke")
        // …
    }
}

class Test {
    public static void scaffoldExe(string name) {
        let tmp = "/tmp/amc-new-" + name
        Process.Run("rm -rf " + tmp)
        let amc = Program.ResolveSelfPath()
        let cmd = amc + " new " + name + " --template exe"
        let r = Process.RunCapture(cmd)
        if (r.Exit != 0) {
            Console.WriteLine("[FAIL] scaffold exe " + name)
            return
        }
        let mainAm = tmp + "/src/main.am"
        if (File.Exists(mainAm)) {
            Console.WriteLine("[PASS] scaffold exe " + name)
        } else {
            Console.WriteLine("[FAIL] missing main.am for " + name)
        }
    }
    // …
}
```

Validates the conversion pattern. Estimated ~200 lines of AM (slightly
longer than the bash because AM doesn't have shell's terse glob
operators, but reads strictly).

## Step 3 — Convert `run_fmt_tests.sh` (~132 lines)

Same shape as Step 2: shell out to `amc fmt`, diff the output. The
multifile loop converts cleanly to a `for path in 0..n` over the
list of `.am` files.

## Step 4 — Convert the LSP / migrate sections of `run_tests.sh`

`run_tests.sh` has three logical sections that need different
strategies:

| Section | Lines | Migration |
|---|--:|---|
| AM sample regressions | ~700 | already `*_test.am`-shaped; just need `expected_stdout` assertion field in the test sample itself |
| LSP probes            | ~150 | port to AM: open a `Process` to `amc lsp`, write LSP frames, parse responses |
| Multifile compile     | ~100 | needs `amc test --multifile <list.txt>` flag — small extension |
| Migrate prompt checks | ~60  | similar to LSP probes — exec amc, grep stdout |

The compile-then-grep-expected-stdout sections need a new `amc test`
feature: **per-test expected-stdout assertions**. Two paths:

1. **Comment-driven**: a `// AMC_EXPECT: <substr>` header in the test
   file. `amc test` parses it and matches against the binary's
   stdout. Backward compatible (today's tests just don't have the
   header).
2. **In-test assertions**: the test prints `[PASS] x = 42` itself
   when it sees the right output. No runner change needed but
   moves the assertion into AM code.

Path 2 is cleaner — keep `amc test` minimal, put assertions in
the test code. Path 1 needed only when the existing bash runner
groups many `run_test` invocations against the SAME test file
(see `closures_capture.am` with 4 `run_test` calls in
`run_tests.sh`). Solution: split those into 4 separate `*_test.am`
files, each printing its own `[PASS]`/`[FAIL]`.

## Step 5 — Convert `run_stdlib_tests.sh` (~655 lines) + retire bash

Largest, but mostly mechanical — the stdlib tests are already AM
files that print `[PASS]`/`[FAIL]`; the bash just orchestrates the
runs. Once Steps 2-4 land, the bash here just goes away.

## What `amc test` still needs after Step 5

Out of scope for this proposal, tracked for later:

- **Parallel execution** — `--jobs N` for multi-core fan-out. Today
  `amc test` is strictly serial.
- **`--multifile <list>`** — explicit multi-file compile (the bash
  `run_multifile_test` helper). Could also be solved by a
  `// AMC_MULTIFILE: path1, path2` header.
- **Per-test env (`AMC_FLAGS=...`)** — for tests that need special
  amc flags. Header pragma `// AMC_FLAGS: --foo` would do it.
- **Streaming output mode** — for CI, dump each test's line in
  real time (today the runner waits per-test then prints all at
  once). Subtle: needs `Process.Spawn` API (planned, not shipped).

## Why not all-at-once?

Bash → AM port for 2,400 lines is genuinely 1-2 weeks. Splitting
into 5 PRs:

- Lands the smallest concrete user benefit immediately (`--filter` etc.)
- Each PR is its own atomic verification (the bash and the AM
  versions run side-by-side during transition)
- Reduces the blast radius if one conversion regresses something
  subtle in the test suite
- Keeps the release-cadence small (one digit each per PR)

## Out of scope

- Replacing the platform-specific Windows MSYS2 path in
  `tests/run_tests.sh` with a true Windows-native test runner.
  The bash works fine under MSYS2 today.
- Splitting `amc test`'s output into a junit.xml or TAP format.
  Adds value when wiring into a test-explorer UI; do when needed.
