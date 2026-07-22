# sanssushi/skills

A collection of [agent skills](https://agentskills.io/) for AI coding agents, installable via the open [`skills` CLI](https://github.com/vercel-labs/skills).

## Available Skills

### scala-weaver-test

Guidance for writing and modifying Scala 3 tests with [Weaver](https://typelevel.org/weaver-test/) (`weaver-cats`, `weaver-discipline`, `weaver-scalacheck`). Covers suite selection (`FunSuite`, `SimpleIOSuite`, `IOSuite`), effects, expectations, shared resources, and discipline laws.

**Use when:**
- Writing or modifying Scala 3 tests with Weaver
- Choosing a Weaver suite type by effect/resource needs
- Modeling setup/teardown as `Resource` acquire/release
- Working with Weaver expectations, `Discipline`, or `ScalaCheck`

### scala3-syntax

Reference for writing valid **modern Scala 3** code under a strict compiler profile (Scala 3.8.x, `-source:future`, `-preview`, `-language:strictEquality`, `-Yexplicit-nulls`). Covers enums/ADTs, `given`/`using`/context bounds (SIP-64 new syntax), union/match/opaque/`into` types, named tuples (SIP-58), strict `CanEqual` equality, explicit nulls, extension methods, `export`, fewer braces (SIP-44), clause interleaving (SIP-47), `inline`/`compiletime` metaprogramming, and the warning → fix table for the strict flag profile.

**Use when:**
- Writing or reviewing any new Scala 3 source file
- Defining a typeclass instance or adding a typeclass constraint
- Modelling a closed set of cases (prefer `enum` over `sealed trait`)
- Hitting an equality / `==` compile error under strict equality
- Dealing with `null` / NPE / Java interop under explicit nulls
- Triaging `-Wunused` / `-Wvalue-discard` / `-Wnonunit-statement` warnings

> Covers the **Scala 3 language itself**, not 3rd-party libraries.

### sbt2-cli

Practical reference for the **sbt 2.x command line** (runner, interactive shell, task invocation). Covers quoted command sequences (`sbt "clean ; compile ; test"`), incremental/cached `test` vs `testFull`, `testOnly` semantics, sbtn thin client + background server lifecycle, client-side `run`/`console`, mandatory slash syntax, sbt query, unified `target/out` layout, machine-wide caching, and CI recipes.

**Use when:**
- Running, scripting, or debugging `sbt` commands on an sbt 2.x build
- `sbt clean compile test` fails with `Expected whitespace character`
- `sbt compile` returns instantly and looks like it did nothing (thin client + disk cache)
- Tests don't run or don't re-run (`test` is now incremental; `clean` doesn't reset it)
- Writing CI pipelines for sbt 2.x (quoting, `shutdown`, env persistence, report paths)

> Covers the **CLI only** — `build.sbt` DSL and plugin migration defer to the official guides.

## Installation

Install all skills:

```bash
npx skills add sanssushi/skills
```

Install a specific skill:

```bash
npx skills add sanssushi/skills --skill scala-weaver-test
npx skills add sanssushi/skills --skill scala3-syntax
npx skills add sanssushi/skills --skill sbt2-cli
```

Install globally (available across all projects):

```bash
npx skills add sanssushi/skills --skill scala3-syntax -g
```

Install to a specific agent (e.g. OpenCode):

```bash
npx skills add sanssushi/skills --skill scala3-syntax -a opencode
```

Non-interactive (CI/CD friendly):

```bash
npx skills add sanssushi/skills --skill scala3-syntax -g -a opencode -y
```

### Installation Scope

| Scope | Flag | OpenCode Path | Use Case |
| --- | --- | --- | --- |
| Project | (default) | `.agents/skills/` | Committed with your project, shared with team |
| Global | `-g` | `~/.config/opencode/skills/` | Available across all projects |

The CLI auto-detects installed coding agents. To target OpenCode explicitly, pass `-a opencode`. See the [`skills` CLI docs](https://github.com/vercel-labs/skills) for the full list of supported agents and options.

## Usage

Skills are loaded on-demand. The agent reads the skill `name` and `description` at startup and loads the full `SKILL.md` into context only when it decides the skill is relevant to the current task.

Example triggers:

- `scala-weaver-test`:
  - "Write a Weaver test for this Scala service"
  - "Add a shared resource suite for my repository"
  - "Convert this ScalaTest assertion to Weaver"
- `scala3-syntax`:
  - "Write a `given Ord[Int]` instance"
  - "Why is my `==` comparison failing under strict equality?"
  - "Convert this Scala 2 ADT to Scala 3 enum syntax"
  - "How do I write a context bound with the new SIP-64 syntax?"
- `sbt2-cli`:
  - "Why does `sbt clean compile test` fail with 'Expected whitespace character'?"
  - "`sbt test` didn't run my tests after migrating to sbt 2"
  - "Force one test suite to run on sbt 2.x"
  - "Update our CI pipeline for sbt 2.x"

## Skill Structure

Each skill is a directory under `skills/` containing a `SKILL.md` file with YAML frontmatter (`name` + `description`, plus optional `author`/`tags`/`license`) followed by Markdown instructions. A human-facing `README.md` may sit alongside `SKILL.md` for browsing the repo; it is not loaded by the agent. Skills may also ship a `references/` subdirectory of topical `.md` files that the agent pulls into context on demand.

```
skills/
  scala-weaver-test/
    SKILL.md
    README.md
  scala3-syntax/
    SKILL.md
    README.md
    references/
      adts-and-deriving.md
      compiler-flags.md
      ...
  sbt2-cli/
    SKILL.md
    README.md
    references/
      commands.md
      client-server.md
      caching-and-ci.md
```

See the [Agent Skills specification](https://agentskills.io/) for the full format.

## License

[MIT](./LICENSE)
