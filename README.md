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

## Installation

Install all skills:

```bash
npx skills add sanssushi/skills
```

Install a specific skill:

```bash
npx skills add sanssushi/skills --skill scala-weaver-test
```

Install globally (available across all projects):

```bash
npx skills add sanssushi/skills --skill scala-weaver-test -g
```

Install to a specific agent (e.g. OpenCode):

```bash
npx skills add sanssushi/skills --skill scala-weaver-test -a opencode
```

Non-interactive (CI/CD friendly):

```bash
npx skills add sanssushi/skills --skill scala-weaver-test -g -a opencode -y
```

### Installation Scope

| Scope | Flag | OpenCode Path | Use Case |
| --- | --- | --- | --- |
| Project | (default) | `.agents/skills/` | Committed with your project, shared with team |
| Global | `-g` | `~/.config/opencode/skills/` | Available across all projects |

The CLI auto-detects installed coding agents. To target OpenCode explicitly, pass `-a opencode`. See the [`skills` CLI docs](https://github.com/vercel-labs/skills) for the full list of supported agents and options.

## Usage

Skills are loaded on-demand. The agent reads the skill `name` and `description` at startup and loads the full `SKILL.md` into context only when it decides the skill is relevant to the current task.

Example triggers for `scala-weaver-test`:
- "Write a Weaver test for this Scala service"
- "Add a shared resource suite for my repository"
- "Convert this ScalaTest assertion to Weaver"

## Skill Structure

Each skill is a directory under `skills/` containing a `SKILL.md` file with YAML frontmatter (`name` + `description`) followed by Markdown instructions.

```
skills/
  scala-weaver-test/
    SKILL.md
```

See the [Agent Skills specification](https://agentskills.io/) for the full format.

## License

[MIT](./LICENSE)
