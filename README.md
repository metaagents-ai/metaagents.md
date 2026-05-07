# MetaAgents

> An open standard for composing and resolving dependencies between agents, skills, and MCPs.

## What is MetaAgents?

[AGENTS.md](https://agents.md) defines how to instruct a coding agent.  
[Agent Skills](https://agentskills.io) defines a portable skill format.  
**MetaAgents** defines how they relate to each other — dependency declarations, composition rules, and resolution semantics.

MetaAgents extends the existing community standards. If your agent or skill has no dependencies, it's already MetaAgents-compatible. MCP names follow the same scoped convention used by the [official MCP registry](https://registry.modelcontextprotocol.io), so the same identifiers used in the registry can appear directly in MetaAgents dependency declarations. (How those identifiers are resolved to a package is outside this spec — see [Out of Scope](#out-of-scope).)

## Core Concepts

| Concept | File | Description |
|---------|------|-------------|
| **Agent** | `AGENTS.md` | An execution template — combines instructions with skills and MCPs |
| **Skill** | `SKILL.md` | A reusable capability package — methodology, knowledge, or workflow |
| **MCP** | `<name>.json` | A [Model Context Protocol](https://modelcontextprotocol.io) server configuration. Public servers are catalogued in the [MCP Registry](https://registry.modelcontextprotocol.io). |

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
name: langsensei/code-reviewer
description: Reviews pull requests with security and style checks.
version: 1.0.0
dependencies:
  skills:
    - langsensei/security-audit
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
| `name` | Yes | string | Lowercase kebab-case identifier, optional `scope/name` format. Total length 1-64 chars including any scope prefix (see [Scoped Names](#scoped-names)) |
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
prereqs: "Requires a Semgrep API key. Set SEMGREP_KEY in your environment. Read references/setup-guide.md for details."
---

## Instructions

When reviewing code for security issues...
```

#### Skill Frontmatter

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | Lowercase kebab-case identifier, optional `scope/name` format. Total length 1-64 chars including any scope prefix (see [Scoped Names](#scoped-names)) |
| `description` | Yes | string | 1-1024 chars |
| `version` | No | string | Semver. Defaults to `0.0.1` |
| `dependencies` | No | object | Skills and MCPs this skill requires |
| `prereqs` | No | string | Prerequisites the LLM should check after installation |

### Prerequisites

The `prereqs` field is an optional string describing conditions or setup steps that should be verified after a skill is installed. The LLM reads it and takes appropriate action (e.g., verifying env vars, running setup commands, reading referenced files, informing the user).

```yaml
prereqs: "Run 'npm install -g playwright'. Ensure OPENAI_API_KEY is set. Read references/setup-guide.md for configuration details."
```

### MCP JSON

Standard [Model Context Protocol](https://modelcontextprotocol.io) server configuration. MetaAgents does not modify or interpret the contents.

The MCP name is derived from the filename and (for scoped names) the parent directory:

- `mcps/filesystem.json` → name is `filesystem`
- `mcps/io.github.modelcontextprotocol/filesystem.json` → name is `io.github.modelcontextprotocol/filesystem`

```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["@modelcontextprotocol/server-filesystem"]
}
```

#### MCP name validation

MCP names follow the validation rules of the [official MCP registry](https://registry.modelcontextprotocol.io). MetaAgents accepts any name the registry would accept, plus an unscoped form for local-only use.

| Form | Pattern | Length | Use case |
|------|---------|--------|----------|
| **Scoped** (registry-aligned) | `^[A-Za-z0-9.-]+/[A-Za-z0-9._-]+$` | 3-200 chars | Public MCPs (resolvable from the registry) |
| **Unscoped** (local) | `^[A-Za-z0-9._-]+$` | 1-64 chars | Internal / community packages where collision is not a concern |

**Why MCP rules differ from Agent/Skill rules.** MCPs are referenced from a public ownership-verified registry, which is the single source of truth for what scoped names exist. Per the [robustness principle](https://en.wikipedia.org/wiki/Robustness_principle) — *be liberal in what you accept* — MetaAgents accepts every name the registry accepts and does not impose additional syntactic restrictions on top (in particular: mixed case, underscores, and longer names are all permitted, matching the registry exactly). Agents and Skills, by contrast, follow the lowercase-kebab convention inherited from [agents.md](https://agents.md) and [Agent Skills](https://agentskills.io) and have their own stricter rules defined in their respective frontmatter tables.

## Scoped Names

Agent and skill names support an optional scope prefix using `scope/name` format:

- `security-audit` — unscoped (community/standard)
- `langsensei/xiaohongshu` — scoped to `langsensei`
- `openclaw/weather` — scoped to `openclaw`

Scope rules for agents and skills:
- The **name part** (after `/`) follows kebab-case rules (`[a-z0-9-]`). The **scope part** (before `/`) follows the same rules but additionally allows `.` to support reverse-DNS-style namespaces. A scope is treated as a single atomic identifier — `io.playwright` and `com.example.team` are each *one* scope string, not nested sub-segments. Dots are not permitted in the name part.
- A single `/` separates scope from name
- Scoped and unscoped names coexist in the same registry
- The full string (including scope) is the unique identifier — `security-audit` and `langsensei/security-audit` are two distinct entries, not aliases

For MCP names, see [MCP name validation](#mcp-name-validation) — MCPs follow the official registry's pattern, which is more permissive than the agent/skill rules above.

### Recommended scope conventions

For agents and skills, scopes typically reflect publisher identity (`langsensei/`, `openclaw/`).

For MCPs, the [official MCP registry](https://registry.modelcontextprotocol.io) uses **reverse-DNS scopes** that the publisher can prove ownership of. Two namespace patterns are common:

- **GitHub-verified** — `io.github.<gh-user-or-org>/<server>` (e.g. `io.github.modelcontextprotocol/filesystem`)
- **Domain-verified** — `<reverse-domain>/<server>` (e.g. `ai.aarna/atars-mcp`, `ac.inference.sh/mcp`)

MetaAgents recommends — but does not require — the same convention for MCPs published to public registries, so that names remain globally unique across publishers.

Unscoped MCP names remain valid for local-only use, internal registries, or community packages where collision is not a concern.

## Directory Layout

The registry directory is organized by type, with scoped names mapped to subdirectories:

```
<registry>/
├── agents/
│   ├── <name>/AGENTS.md              # unscoped
│   └── <scope>/<name>/AGENTS.md      # scoped
├── skills/
│   ├── <name>/SKILL.md               # unscoped
│   └── <scope>/<name>/               # scoped
│       ├── SKILL.md
│       ├── scripts/                   # optional
│       ├── references/                # optional
│       └── assets/                    # optional
└── mcps/
    ├── <name>.json                    # unscoped
    └── <scope>/<name>.json            # scoped
```

How scoped names are mapped to the runtime environment is outside the scope of this specification. Two implementer concerns worth calling out:

- **Flattening** for systems that require a single-level directory (e.g. encoding `scope/name` as `scope__name`).
- **Case-sensitive uniqueness** is part of the abstract spec, but on case-insensitive filesystems (Windows NTFS default, macOS APFS default) MCP names that differ only in case will collide on disk; implementations may need to normalize, hash, or otherwise disambiguate such names.

## Dependencies

```yaml
dependencies:
  skills:
    - langsensei/security-audit
    - style-guide
  mcps:
    - io.github.modelcontextprotocol/filesystem
    - github
```

Both `skills` and `mcps` are optional arrays of names referencing other entries in the same registry.

## Resolution Rules

1. **Topological order** — dependencies are resolved depth-first
2. **No cycles** — circular dependencies are invalid
3. **Missing dependencies** — resolution fails if a declared dependency is absent
4. **Name uniqueness** — the fully-qualified name (including scope if present) must be unique within each type (agents, skills, mcps)
5. **MCPs are leaves** — MCP entries cannot declare dependencies

## Backward Compatibility

MetaAgents is a strict superset:

- An `AGENTS.md` without frontmatter is a valid agents.md file
- A `SKILL.md` without `dependencies` is a valid Agent Skills file
- Tools that don't understand MetaAgents fields simply ignore them

## Out of Scope

MetaAgents is format-level only. It does **not** define:

- How to fetch or publish packages (that's for package managers)
- How to provision or execute agents (that's for runtimes)
- How to interpret instructions (that's for LLMs)
- How to spawn MCP servers (that's for substrates)

## License

[MIT](LICENSE)
