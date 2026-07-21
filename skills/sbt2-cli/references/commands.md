# sbt 2.x commands (validated against sbt 2.0.3)

Command-level behavior of the `sbt` CLI. Examples are real outputs from a
multi-project build (`root` aggregating `foo` and `bar`).

## Invoking commands from the system shell

### A sequence of commands must be one quoted, `;`-separated string

```bash
# sbt 1.x style — FAILS on 2.x
$ sbt clean compile test
[error] Expected whitespace character
[error] Expected '/'
[error] clean compile test
[error]       ^
# exit code 1

# sbt 2.x — correct
$ sbt "clean ; compile ; test"
```

Rules of thumb:

- **One command per argument is NOT a thing anymore.** Quote the whole
  sequence as a single shell argument, commands separated by `;`.
- Inside the **interactive sbt shell**, `A ; B` chains commands (`;` runs B if
  A succeeded). The quoting requirement is about the *system shell* invocation.
- Batch files still work: `sbt "< commands.txt"` executes one command per line
  (`#` comments and empty lines ignored). Validated on 2.0.3.
- A single bare command does not need quotes as long as it contains no spaces:
  `sbt compile`, `sbt testFull`, `sbt shutdown` are all fine.
- Quote anything containing `*` or `...` anyway — your system shell would
  otherwise glob them (`sbt test *Foo*` unquoted expands to files!).

### Slash syntax is mandatory (colon syntax removed)

```bash
$ sbt "test:compile"
[error] Expected ID character
[error] Not a valid command: test (similar: testOnly, set, last)
...
# exit code 1

$ sbt "Test/compile"   # works
```

The sbt 0.13-style `config:task` syntax is removed from the shell (and from
`build.sbt`). Always use `<project-id> / Config / task`, with any axis omitted
if unneeded: `Test/compile`, `foo/Test/compile`, `compile`.

### Scoped keys no longer delegate on the shell

```bash
$ sbt "Compile/update"
[error] No such setting/task
# exit code 1
```

In sbt 1.x a non-existent scoped key fell back through the delegation chain
(`Compile/update` → `update`). sbt 2.x (PR #8539) disables delegation for
**scoped** keys typed on the shell — it fails instead. The **subproject axis**
still delegates (so `Global`/`ThisBuild`-style resolution works), and a later
fix (#8751) relaxed the rule for settings. Practical impact is small, but if an
old script breaks with `No such setting/task` on a scoped key, this is why.

## The test family

| Task | sbt 2.x meaning | sbt 1.x equivalent |
|---|---|---|
| `test` | incremental + cached, accepts suite filters | `testQuick` (renamed, PR #7686) |
| `testFull` | always runs everything, uncached | old `test` |
| `testOnly` | selection; at root it is a **command** that fails if nothing matches | `testOnly` (but silent success on typos) |

### `test` is incremental and cached

`test` runs only suites that:

1. failed in the previous run, **or**
2. never ran, **or**
3. had a transitive dependency recompiled (possibly in another subproject).

Everything else is skipped, observed as:

```
[info] No tests to run for foo / Test / testQuick
[info] Passed: Total 0, Failed 0, Errors 0, Passed 0
[success] elapsed time: 0 s
```

Two important corollaries:

- **The skip state survives `clean`.** Success is recorded in a machine-wide
  action cache keyed by content hashes (hermetic), not timestamps.
  `sbt "clean ; test"` still skips green suites. To force a full re-run, use
  `testFull`.
- **`testFull` and `test` share the success state.** Running `testFull` marks
  suites green for subsequent `test` runs too.
- Log lines still say `testQuick` — that's the internal task id left over from
  the rename; it does not mean you invoked the wrong thing.

### `test` accepts suite filters directly

```bash
sbt "test foo.FooSuite foo.OtherSuite"   # whitespace-separated names
sbt "test *Foo*"                         # * wildcard (quote against shell glob!)
sbt "test org.example.MyTest -- -verbosity 1"   # args after -- go to the framework
```

Filters **compose** with incrementality: a filtered `test` still skips suites
whose success is cached. Only an actually-changed suite runs:

```
$ sbt "test *Bar*"        # after editing bar's test source
[info] compiling 1 Scala source to .../bar/test-classes ...
Test run bar.BarSuite started
[info] Passed: Total 2, Failed 0, Errors 0, Passed 2
```

### `testOnly` fails when nothing matches

At the root of an aggregated build, `testOnly` is now a command (PR #8607) that
verifies the pattern matched something across all subprojects:

```bash
$ sbt "testOnly NonExistent"
[error] No tests match the patterns: NonExistent
[error] Available tests:
[error]   - bar.BarSuite
[error]   - foo.FooSuite
# exit code 1
```

This makes typos in CI fail loudly instead of passing silently. `testOnly`
also supports `...` as a wildcard pattern.

Unlike filtered `test`, **`testOnly` is a force-run**: it executes the matched
suites even when their success is cached (validated: a green suite re-ran via
`testOnly foo.FooSuite`). This is the way to run one specific suite *now*
without touching code. `testFull`, by contrast, takes no filters —
`sbt "testFull *Foo*"` is a parse error.

### Other test facts

- `test` returns `TestResult` (was `Unit`) — matters only for build authors.
- JUnit XML reports land in `target/out/**/test-reports/*.xml`
  (see [caching-and-ci.md](caching-and-ci.md)).
- sbt 2.x enforces eviction errors in the `Test` configuration: version
  conflicts that were warnings in `Test` under 1.x now fail `update`.
- `IntegrationTest` is **removed** — `sbt "it:test"` no longer exists. Migrate
  to a separate subproject with normal tests.

## sbt query (subproject axis)

The slash syntax gained a query axis: `[query /] [Config /] [task /] key`.

```bash
sbt "foo.../test"                      # all subprojects whose id starts with foo
sbt ".../test"                         # the whole aggregate
sbt "...@scalaBinaryVersion=3/test"    # all subprojects with scalaBinaryVersion 3
```

Validated: `foo.../testFull` compiled and ran only `foo`'s suites; the
`@scalaBinaryVersion=3` form ran every subproject (all on Scala 3).

Notes:

- The wildcard is `...`, deliberately **not** `*`: `*` would be glob-expanded
  by your system shell against real files (e.g. `*/test` → `src/test`).
  Quote queries in scripts regardless.
- Queries also narrow **compilation**: only the matched subprojects (and their
  dependencies) compile.
- For project-matrix builds, `@scalaBinaryVersion=` is the version axis
  (e.g. `...` across all matrix projects of one binary version).

## Shell session behavior

- **Auto-reload is default.** When `build.sbt` (or `project/`) changes, the
  next command reloads automatically:
  `[info] build source files have changed ... Reloading sbt...`. No more
  manual `reload` prompt. `reload` still exists for forcing it.
- **`shutdown` vs `exit`.** `shutdown` stops the background sbt server (use it
  to reset env vars or free the machine). `exit`/`quit`/Ctrl+D only ends the
  client session; the server keeps running. See
  [client-server.md](client-server.md).
- **Watch mode** `~ <command>` still exists for re-running on file change.
- `clean` still deletes the unified `target/` directory — but remember it does
  **not** clear the machine-wide task/test caches.
- `dependencyTree` was unified into a single input task.
