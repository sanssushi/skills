---
name: sbt2-cli
author: sanssushi
description: >
  Practical reference for the sbt 2.x command line (runner, interactive shell,
  task invocation). Use whenever running, scripting, or debugging `sbt` commands on sbt 2.x builds.
  Covers the CLI only — for build.sbt DSL or plugin migration, defer to the linked official guides.
tags: ["scala", "sbt", "build"]
license: MIT
---

# sbt 2.x CLI Skill

Covers **driving sbt 2.x from the command line**: the `sbt` runner, the
interactive shell, and task invocation syntax.

**Frame of reference:** sbt 2.x as it behaves today. sbt 1.x habits —
`sbt clean compile test`, `test` running everything, `testQuick` for
incremental runs, `test:compile`, per-project `target/` dirs — no longer
hold.

## Mental model shifts

1. **Client/server.** The `sbt` runner launches a native thin client (sbtn)
   that sends your command to a long-lived background sbt server. `sbt compile`
   returning in ~0 s is normal: no JVM boot, warm server, plus caches.
2. **Everything is cached.** All tasks are cached by default to a machine-wide
   disk cache. `clean ; compile` can finish in 0 s with no `compiling` logs —
   the result was restored from cache. This is a hit, not a bug.
3. **`test` is incremental.** `test` only runs suites that failed, never ran,
   or whose transitive dependencies changed — keyed by content hash, so the
   skip state even survives `clean`. `testFull` is the old always-run-everything
   `test`.

## Common mistakes coming from sbt 1.x

| sbt 1.x habit | sbt 2.x behavior |
|---|---|
| `sbt clean compile test` | `sbt "clean ; compile ; test"` — one quoted, `;`-separated string. Bare form fails: `[error] Expected whitespace character`, exit 1 |
| `sbt test` (expect all tests to run) | `test` is incremental + cached (old `testQuick`). Use `sbt "testFull"` for the old always-run behavior |
| `sbt testQuick` | renamed: use `test`. Log lines still show the internal id, e.g. `No tests to run for foo / Test / testQuick` — this is expected |
| `sbt "testOnly com.example.FooSuite"` as the only filter mechanism | `test` itself takes filters now: `sbt "test *Foo*"` (whitespace-separated names, `*` wildcard). `testOnly` still exists: it **force-runs** matched suites bypassing the cache, and at root **fails the build if no test matches** anywhere |
| `sbt "clean ; test"` to force a re-run | does NOT force a re-run: test success is cached machine-wide by content hash and survives `clean`. Use `testFull` |
| `test:compile` (colon syntax) | rejected with a parse error. Slash syntax only: `Test/compile` |
| `sbt "Compile/update"` relying on scope delegation | scoped keys no longer delegate on the shell: `[error] No such setting/task`. (Subproject-axis delegation still works.) |
| "sbt compile returned instantly — it didn't compile!" | thin client + warm server + disk cache. Verify with `find target/out -name '*.class'` — classfiles are restored even after `clean` |
| Per-project `target/` dirs (`foo/target/...`) | unified layout: `target/out/jvm/scala-<ver>/<subproject>/` |
| Test reports at `target/test-reports/*.xml` | `target/out/**/test-reports/*.xml` (e.g. `target/out/jvm/scala-3.8.4/bar/test-reports/TEST-bar.BarSuite.xml`) |
| Global config/plugins at `~/.sbt/1.0/` | global base dir is now `<base>/2`, e.g. `$XDG_CONFIG_HOME/sbt/2` or `~/.config/sbt/2` |
| Editing `build.sbt`, then typing `reload` | auto-reload is on by default: next command logs `build source files have changed ... Reloading sbt...` |
| `sbt "it:test"` (IntegrationTest config) | `IntegrationTest` is removed. Make a separate subproject with normal tests |
| CI steps as separate `sbt` invocations with per-step env vars | later steps reuse the first invocation's server and its env. Pass env at job level, or end steps with `shutdown` |
| `sbt` requires JDK 8/11 | sbt 2.x requires **JDK 17+** |

## Quick reference

| Topic | Reference file |
|-------|----------------|
| Quoting/sequencing rules, slash syntax, delegation, test family, query, watch, `shutdown` vs `exit`, removed features | [references/commands.md](references/commands.md) |
| Runner/sbtn/server architecture, lifecycle, env-var persistence, client-side run/console, JVM flags | [references/client-server.md](references/client-server.md) |
| Task caching, reading cache behavior from output, `target/out` layout, test-report globs, global dirs, CI recipes | [references/caching-and-ci.md](references/caching-and-ci.md) |

## When to read which file

- **Any `sbt` invocation, one-off or scripted** → skim the mistakes table above;
  details in [commands.md](references/commands.md).
- **Tests not running / not re-running** → [commands.md](references/commands.md)
  (test semantics) and [caching-and-ci.md](references/caching-and-ci.md)
  (hermetic test cache).
- **A command "does nothing", returns instantly, or behaves stale** →
  [client-server.md](references/client-server.md) and
  [caching-and-ci.md](references/caching-and-ci.md).
- **Writing or fixing CI pipelines** →
  [caching-and-ci.md](references/caching-and-ci.md).
- **Multi-project / cross-build invocation** → sbt query in
  [commands.md](references/commands.md).

## Out of scope

This skill covers **CLI usage only**.

- **`build.sbt` DSL migration** (Scala 3 metabuild, bare settings as common
  settings, `%%` platform awareness, `Def.uncached`, `rootProject`, …):
  <https://www.scala-sbt.org/2.x/docs/en/changes/migrating-from-sbt-1.x.html>
- **sbt plugin authoring/cross-building** (`_sbt2_3` suffix, `PluginCompat`,
  sbt2-compat): same migration guide plus
  <https://www.scala-lang.org/blog/2026/03/02/sbt2-compat.html>
- **Full change list**:
  <https://www.scala-sbt.org/2.x/docs/en/changes/sbt-2.0-change-summary.html>

## References

- The Book of sbt (2.x docs root): <https://www.scala-sbt.org/2.x/docs/en/>
- sbt 2.0 change summary: <https://www.scala-sbt.org/2.x/docs/en/changes/sbt-2.0-change-summary.html>
- Migrating from sbt 1.x: <https://www.scala-sbt.org/2.x/docs/en/changes/migrating-from-sbt-1.x.html>
- `sbt` command reference: <https://www.scala-sbt.org/2.x/docs/en/reference/sbt.html>
- `sbt test` reference: <https://www.scala-sbt.org/2.x/docs/en/reference/sbt-test.html>
- sbt query: <https://www.scala-sbt.org/2.x/docs/en/concepts/sbt-query.html>
- Caching: <https://www.scala-sbt.org/2.x/docs/en/concepts/caching.html>
- sbt 2 announcement (scala-lang blog): <https://www.scala-lang.org/blog/2026/06/29/sbt2.html>
- Key PRs: rename testQuick→test [#7686](https://github.com/sbt/sbt/pull/7686),
  testOnly as command [#8607](https://github.com/sbt/sbt/pull/8607),
  no shell delegation [#8539](https://github.com/sbt/sbt/pull/8539)
