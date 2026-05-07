# MetaAgents

> An open standard for composing and resolving dependencies between agents, skills, and MCPs.

## What is MetaAgents?

[AGENTS.md](https://agents.md) defines how to instruct a coding agent.  
[Agent Skills](https://agentskills.io) defines a portable skill format.  
**MetaAgents** defines how they relate to each other — dependency declarations, composition rules, and resolution semantics.

MetaAgents extends the existing community standards. If your agent or skill has no dependencies, it's already MetaAgents-compatible.

## Core Concepts

| Concept | File | Description |
|---------|------|-------------|
| **Agent** | `AGENTS.md` | An execution template — combines instructions with skills and MCPs |
| **Skill** | `SKILL.md` | A reusable capability package — methodology, knowledge, or workflow |
| **MCP** | `<name>.json` | A Model Context Protocol server configuration |

Agents depend on skills and MCPs. Skills can depend on other skills and MCPs. MCPs are always leaf nodes.

```
Agent A
├── Skill X
│   ├── Skill Y
│   └── MCP P
├── Skill Z
└── MCP Q
```

## File Formats

### AGENTS.md

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

#### Agent Frontmatter

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | 1-64 chars, lowercase kebab-case |
| `description` | Yes | string | 1-1024 chars |
| `version` | No | string | Semver. Defaults to `0.0.1` |
| `dependencies` | No | object | Skills and MCPs this agent requires |

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
prereqs:
  - message: "Requires a Semgrep API key. Set SEMGREP_KEY in your environment."
  - file: references/setup-guide.md
---

## Instructions

When reviewing code for security issues...
```

#### Skill Frontmatter

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | 1-64 chars, lowercase kebab-case |
| `description` | Yes | string | 1-1024 chars |
| `version` | No | string | Semver. Defaults to `0.0.1` |
| `dependencies` | No | object | Skills and MCPs this skill requires |
| `prereqs` | No | array | Prerequisites the LLM should check after installation |

### Prerequisites

The `prereqs` field defines conditions or setup steps that should be verified after a skill is installed. Each entry is one of:

- **`message`** — A plain-text instruction for the LLM to present or verify
- **`file`** — A relative path to a file the LLM should read for setup guidance

```yaml
prereqs:
  - message: "Run 'npm install -g playwright' before using this skill."
  - file: references/setup-guide.md
  - message: "Ensure OPENAI_API_KEY is set in your environment."
```

After installation, the LLM reads all prereqs and takes appropriate action (e.g., verifying env vars, running setup commands, informing the user).

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
│       └── AGENTS.md
├── skills/
│   └── <name>/
│       ├── SKILL.md
│       ├── scripts/       # optional
│       ├── references/    # optional
│       └── assets/        # optional
└── mcps/
    └── <name>.json
```

## Dependencies

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

1. **Topological order** — dependencies are resolved depth-first
2. **No cycles** — circular dependencies are invalid
3. **Missing dependencies** — resolution fails if a declared dependency is absent
4. **Name uniqueness** — no two entries may share the same name across the registry
5. **MCPs are leaves** — MCP entries cannot declare dependencies

## Backward Compatibility

MetaAgents is a strict superset:

- An `AGENTS.md` without frontmatter is a valid agents.md file
- A `SKILL.md` without `dependencies` is a valid Agent Skills file
- Tools that don't understand MetaAgents fields simply ignore them

## Scope

MetaAgents is format-level only. It does **not** define:

- How to fetch or publish packages (that's for package managers)
- How to provision or execute agents (that's for runtimes)
- How to interpret instructions (that's for LLMs)
- How to spawn MCP servers (that's for substrates)

## Reference Implementation

[@emploke/catalog](https://github.com/LangSensei/emploke) — a TypeScript library implementing MetaAgents resolution and local registry operations.

## License

[MIT](LICENSE)
