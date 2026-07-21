# sbt2-cli

Practical reference for the **sbt 2.x command line** — the `sbt` runner, the
interactive shell, and task invocation syntax. Covers the CLI only; `build.sbt` DSL and
plugin migration are out of scope (official guides linked instead).

## Main changes vs sbt 1.x

| sbt 1.x habit | sbt 2.x |
|---|---|
| `sbt clean compile test` | `sbt "clean ; compile ; test"` — quoted, `;`-separated; bare form errors with `Expected whitespace character` |
| `sbt test` runs everything | `test` is incremental + cached (old `testQuick`); use `testFull` for the old behavior |
| `testQuick` | renamed to `test` (logs may still show the internal id) |
| `clean ; test` forces re-run | no — hermetic, machine-wide test-success cache survives `clean`; use `testFull` (or `testOnly <pattern>` to force one suite) |
| `test:compile` | `Test/compile` — colon syntax removed |
| instant `sbt compile` = broken? | no — sbtn thin client + warm server + disk cache restore |
| per-project `target/` | unified `target/out/jvm/scala-<ver>/<subproject>/` |
| `target/test-reports/*.xml` | `target/out/**/test-reports/*.xml` |
| `~/.sbt/1.0` global config | `$XDG_CONFIG_HOME/sbt/2` (aka `~/.config/sbt/2`) |
| manual `reload` after editing build | auto-reload by default |
| `IntegrationTest` / `it:test` | removed — use a separate subproject |
| JDK 8/11 | JDK 17+ to run sbt |

New capabilities: sbt query (`foo.../test`, `.../test`,
`...@scalaBinaryVersion=3/test`), `test` suite filters with `*` wildcards,
`testOnly` that fails on unmatched patterns, machine-wide disk cache with
optional Bazel-compatible remote cache, client-side `run`/`console`.

## Use when

- Running, scripting, or debugging `sbt` commands on an sbt 2.x build
- A command fails with `Expected whitespace character`, `Expected '/'`, or
  `No such setting/task` after migrating from 1.x
- Tests don't run, don't re-run, or "nothing gets compiled"
- Writing CI pipelines for sbt 2.x (quoting, `shutdown`, env persistence,
  report paths, `-Dsbt.ci=true`)
- Selecting subprojects/tests: queries, filters, `testOnly`

## Files

```
sbt2-cli/
  SKILL.md                      # mistakes table + orientation (loaded by the agent)
  references/
    commands.md                 # quoting, slash syntax, test family, query, shell behavior
    client-server.md            # sbtn thin client, server lifecycle, env persistence, run/console
    caching-and-ci.md           # task caching, target/out layout, dirs, CI recipes
```

## Sources

- [The Book of sbt (2.x)](https://www.scala-sbt.org/2.x/docs/en/)
- [sbt 2.0 changes](https://www.scala-sbt.org/2.x/docs/en/changes/sbt-2.0-change-summary.html)
- [Migrating from sbt 1.x](https://www.scala-sbt.org/2.x/docs/en/changes/migrating-from-sbt-1.x.html)
- [sbt 2 announcement](https://www.scala-lang.org/blog/2026/06/29/sbt2.html)
