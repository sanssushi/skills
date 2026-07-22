# sbt 2.x caching, output layout, and CI

## All tasks are cached by default

sbt 2.x treats task execution like a pure function of its inputs and caches
results:

- **Machine-wide disk cache** — shared by all builds on the machine, on by
  default. No opt-in, no settings.
- **Optional remote cache** — Bazel-compatible gRPC protocol; off by default,
  needs setup (see the official caching docs).

`compile` and `test` were rewritten to be cacheable tasks. Consequences you
will observe:

```
$ sbt "clean ; compile"
[success] elapsed time: 0 s        # no "compiling" lines at all
```

`clean` deleted `target/`, yet `compile` finished instantly without compiling:
the previous result was restored from the machine-wide disk cache. Classfiles
reappear in `target/out/...` as if compilation happened. **This is a cache
hit, not a broken build.**

Change one source file and only its subproject recompiles onsite:

```
[info] compiling 1 Scala source to .../target/out/jvm/scala-3.8.4/foo/classes ...
```

The official docs also show an extended success line with cache statistics,
e.g. `[success] elapsed time: 3 s, cache 49%, 25 disk cache hits, 26 onsite
tasks`. In sbt 2.0.3's default output the plain `elapsed time: N s` form is
typical — don't rely on the stats line being present; rely on elapsed time,
absent `compiling` logs, and `find target/out -name '*.class'`.

## The test cache is hermetic and survives `clean`

Test success is recorded in the action cache keyed by **content hashes**
(SHA-256 Merkle tree of the suite's transitive classfiles), not timestamps:

```
$ sbt "clean ; test"
[info] No tests to run for foo / Test / testQuick
[info] Passed: Total 0, Failed 0, Errors 0, Passed 0
```

Everything still skipped — the machine-wide cache knows these exact bits
already passed. Practical rules:

- **To force re-running green tests: `testFull`.** Not `clean`.
- **To force one suite right now: `testOnly <pattern>`** — it bypasses the
  incremental cache and runs the selected suites unconditionally. Note
  `test <filter>` is NOT a force-run — it still respects the cache, and
  `testFull` accepts no filters (`testFull *Foo*` is a parse error).
- Failed suites re-run automatically on the next `test`.
- The machine-wide disk cache lives under the resolved sbt global base
  (observed as `<sbt.global.base>/cache`); caching that directory between CI
  runs preserves the benefit.

## Directory layout

### Unified `target/` (per build)

```
target/out/jvm/scala-3.8.4/<subproject>/classes           # main classfiles
target/out/jvm/scala-3.8.4/<subproject>/test-classes      # test classfiles
target/out/jvm/scala-3.8.4/<subproject>/test-reports/TEST-*.xml
```

- One `target/` at the build root; subproject/platform/Scala-version encoded
  in the path. There is **no** `foo/target/` anymore.
- JUnit XML reports for CI artifact upload:
  `target/out/**/test-reports/*.xml`.

### Global directories

- **Global base dir** (machine-wide settings/plugins): resolved in order —
  `-Dsbt.global.base`, `$SBT_CONFIG_HOME`, `%LOCALAPPDATA%\sbt` (Windows),
  `$XDG_CONFIG_HOME/sbt`, `$HOME/.config/sbt` — then a subdirectory named `2`.
  So typically `~/.config/sbt/2` (was `~/.sbt/1.0` in sbt 1.x). It is created
  **lazily** — its absence on a fresh machine is normal. Some environments
  override it explicitly (e.g. via `SBT_OPTS=-Dsbt.global.base=...`); check
  `sbt "eval System.getProperty(\"sbt.global.base\")"` when in doubt.
- **Boot dir** (downloaded sbt/Scala artifacts): `$HOME/.sbt/boot` by default,
  `-Dsbt.boot.directory` to override.
- **Coursier cache** for dependencies: standard Coursier resolution
  (`~/.cache/coursier` on Linux).

## CI recipes

```bash
# One step, one quoted sequence — fail fast, then free the server
sbt "clean ; compile ; test ; shutdown"

# Force the full, sbt 1.x-style test run
sbt "testFull"

# Non-interactive output
sbt -Dsbt.ci=true "compile ; testFull"
```

Guidelines:

- **Never** write `sbt clean compile test` (unquoted) — it exits 1 with
  `Expected whitespace character`. This is a common first breakage after migration.
- Prefer `testFull` in CI unless you deliberately want incremental/cached
  testing with its (large) speedup and shared-state semantics.
- Multiple `sbt` steps in one job reuse the first step's server **and its
  environment**. Set env at job level, or `shutdown` between steps.
- Update artifact paths to `target/out/**/test-reports/*.xml`.
- Ensure the CI image has **JDK 17+** for running sbt itself.

## Remote cache (out of scope for setup, good to recognize)

Builds can be configured with a Bazel-compatible remote cache; then cache hits
may come from CI-produced artifacts on other machines. If a colleague's build
"compiles nothing" on first checkout, that is the feature working. Setup is a
`build.sbt`/ops concern — see <https://www.scala-sbt.org/2.x/docs/en/concepts/caching.html>
and the remote-cache setup reference linked from there.
