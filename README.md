# MetaAgents

> An open standard for composing, distributing, and resolving dependencies between agents, skills, and MCPs.

## What is MetaAgents?

[AGENTS.md](https://agents.md) defines how to instruct a coding agent.  
[Agent Skills](https://agentskills.io) defines a portable skill format.  
**MetaAgents** defines how agents, skills, and MCPs relate to each other — dependency declarations, composition rules, and resolution semantics.

MetaAgents is **not** a new file format. It extends the existing community standards with:

- A `dependencies` frontmatter field for declaring what an agent or skill needs
- A directory layout convention for local registries
- Resolution semantics for building the full dependency graph

If your agent or skill has no dependencies, it's already MetaAgents-compatible. Zero changes needed.

## Core Concepts

| Concept | File | Description |
|---------|------|-------------|
| **Agent** | `AGENT.md` | An execution template — combines instructions with skills and MCPs |
| **Skill** | `SKILL.md` | A reusable capability package — methodology, knowledge, or workflow |
| **MCP** | `<name>.json` | A Model Context Protocol server configuration |

Agents depend on skills and MCPs. Skills can depend on other skills and MCPs. MCPs are always leaf nodes (no dependencies).

```
Agent A
├── Skill X
│   ├── Skill Y
│   └── MCP P
├── Skill Z
└── MCP Q
```

## File Formats

### AGENT.md

A superset of [AGENTS.md](https://agents.md). Standard AGENTS.md files work as-is.

```markdown
---
name: code-reviewer
description: Reviews pull requests with security and style checks.
version: 1.0.0
dependencies:
  skills:
    - security-audit
    - style-guide
  mcps:
    - github
---

## Instructions

Review the PR for security vulnerabilities and style violations...
```

### SKILL.md

A superset of [Agent Skills](https://agentskills.io/specification). Standard Agent Skills work as-is.

```markdown
---
name: security-audit
description: Identifies common security vulnerabilities in code.
version: 1.2.0
dependencies:
  skills:
    - cve-database
  mcps:
    - semgrep
---

## Instructions

When reviewing code for security issues...
```

### MCP JSON

Standard [Model Context Protocol](https://modelcontextprotocol.io) server configuration. MetaAgents does not modify or interpret the contents.

```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["@modelcontextprotocol/server-github"]
}
```

## Directory Layout

```
<registry>/
├── agents/
│   └── <name>/
│       └── AGENT.md
├── skills/
│   └── <name>/
│       ├── SKILL.md
│       ├── scripts/       # optional
│       ├── references/    # optional
│       └── assets/        # optional
└── mcps/
    └── <name>.json
```

## Frontmatter Specification

### Required Fields

| Field | Type | Constraints |
|-------|------|-------------|
| `name` | string | 1-64 chars, lowercase kebab-case (`[a-z0-9-]`), no leading/trailing/consecutive hyphens, must match parent directory name |
| `description` | string | 1-1024 chars, describes what it does and when to use it |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | Semantic version. Defaults to `"0.0.1"` if omitted |
| `dependencies` | object | Declares required skills and MCPs (see below) |
| `license` | string | License identifier |
| `compatibility` | string | Environment requirements (max 500 chars) |
| `metadata` | object | Arbitrary key-value pairs, opaque to MetaAgents |

### Dependencies

```yaml
dependencies:
  skills:
    - skill-name-a
    - skill-name-b
  mcps:
    - mcp-name-a
```

Both `skills` and `mcps` are optional arrays of names referencing other entries in the same registry.

## Resolution Rules

1. **Topological order** — dependencies are resolved depth-first; every node appears after all its transitive dependencies
2. **No cycles** — circular dependencies are invalid and must be rejected
3. **Missing dependencies** — if a declared dependency is not present in the registry, resolution fails
4. **Name uniqueness** — no two entries (across agents, skills, and MCPs) may share the same name
5. **MCPs are leaves** — MCP entries cannot declare dependencies

## Backward Compatibility

MetaAgents is designed as a strict superset:

- An `AGENT.md` without frontmatter is a valid AGENTS.md file
- A `SKILL.md` without `dependencies` is a valid Agent Skills file
- Tools that don't understand `dependencies` simply ignore it

## Ecosystem

MetaAgents is format-level only. It does **not** define:

- How to fetch or publish packages (that's for package managers)
- How to provision or execute agents (that's for runtimes)
- How to interpret skill instructions (that's for LLMs)
- How to spawn MCP servers (that's for substrates)

Implementations are free to build any of these on top.

## Reference Implementation

[@emploke/catalog](https://github.com/LangSensei/emploke) — a TypeScript library implementing MetaAgents resolution, dependency graph management, and local registry operations.

## License

[CC-BY-4.0](LICENSE)
