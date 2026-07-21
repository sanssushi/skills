# sbt 2.x runner / sbtn / server (validated against sbt 2.0.3)

Why `sbt compile` returns instantly, why env vars "stick" across invocations,
and how to control the background server.

## Architecture: three pieces

1. **sbt runner** — the `sbt` shell script (`sbt.bat` on Windows) you install.
   Version-agnostic; can drive any sbt 1.x or 2.x build.
2. **sbt server** — the actual build tool. Version pinned per project via
   `project/build.properties` (e.g. `sbt.version=2.0.3`), so every machine
   builds with identical semantics.
3. **sbtn** — a GraalVM-native thin client. When the runner detects an sbt 2.x
   build, it hands commands to sbtn instead of booting a JVM per invocation.

Observed first invocation in a project directory:

```
$ sbt compile
[info] entering thin client - BEEP WHIRR
[info] starting sbt server in the background
[info] use 'sbt shutdown' to shutdown the server
[info] welcome to sbt 2.0.3 (Ubuntu Java 25.0.3)
```

(On pre-release docs you may see "entering *experimental* thin client"; the
2.0.3 message no longer says "experimental".)

Every later invocation reuses the running server — no JVM boot, warm build
state. Combined with the disk cache, `sbt compile` finishing in
`elapsed time: 0 s` with no `[info] compiling ...` lines is the **normal**
steady state, not a sign nothing happened. Verify with:

```bash
find target/out -name '*.class'   # classfiles are there
```

## Server lifecycle

- **Start:** on demand, in the background, on the first `sbt` command in the
  project. Survives after your command finishes.
- **Stop:** `sbt shutdown` (observed output: `[info] disconnected`). Use it to
  release memory, to pick up new env vars, or between CI steps.
- **Restart:** automatic — the next `sbt` command starts a fresh server.
- **One server per build directory.** Commands from a different project get
  their own server.

## Environment variables persist across invocations

Because later invocations reuse the first server, they inherit **its**
environment — not yours. `JAVA_OPTS`/`SBT_OPTS`/`-D` flags given to a later
`sbt` call do not retroactively apply to the running server.

Consequences:

- **CI:** either set all env at the job level (identical for every `sbt`
  invocation), or end each step with `shutdown`:
  `sbt "clean ; compile ; test ; shutdown"`.
- **Locally:** changed `SBT_OPTS` and nothing seems different?
  `sbt shutdown`, then re-run.
- This applies to anything that invokes `sbt` internally too (e.g. GitHub
  Actions like `sbt-dependency-submission`).

## Client-side `run` and `console`

In sbt 2.x the server hands `run` and `console` back to sbtn, which forks a
**fresh JVM** for your program/REPL:

```
$ sbt run
[info] running (fork) rootproj.hello
hello from root
```

Benefits over sbt 1.x in-server execution: the server is not blocked, several
runs/consoles can coexist, and a crashed app can't take down the build server.
`Test / run`, `Test / console`, `consoleQuick` work the same way.

## Requirements and useful flags

- **JDK 17+** is required to run sbt 2.x (it can still *build* projects
  targeting older JVMs). Validated on JDK 25.
- `sbt --script-version` — prints the runner script version.
- `sbt --server` — run as a server instead of thin-client mode.
- `-Dsbt.client=true` — force client mode.
- `-Dsbt.ci=true` — CI mode: suppresses supershell and color (auto-enabled
  when `BUILD_NUMBER` is set).
- JVM tuning: `SBT_OPTS`/`JAVA_OPTS` env vars, `.jvmopts` / `.sbtopts` files,
  `-J-Xmx2G`-style passthrough, `-Dkey=val` system properties. Remember the
  persistence rule above — these apply at **server** start.
- Boot directory (downloaded sbt/Scala artifacts) defaults to
  `$HOME/.sbt/boot`; override with `-Dsbt.boot.directory=...`. Environments may
  also pin `sbt.global.base` explicitly (check `SBT_OPTS` if paths surprise
  you) — see [caching-and-ci.md](caching-and-ci.md) for directory layout.

## Interactive shell

Bare `sbt` still drops you into the interactive sbt shell — now hosted on the
server via the thin client. Inside it, all of [commands.md](commands.md)
applies: slash syntax, `;`-chaining, `test`/`testFull`, queries, auto-reload.
`exit` leaves the shell but leaves the server running; `shutdown` kills both.
