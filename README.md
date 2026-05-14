# MetaAgents

> An open standard for composing and resolving dependencies between agents, skills, and MCPs.

## What is MetaAgents?

[AGENTS.md](https://agents.md) defines how to instruct a coding agent.
[Agent Skills](https://agentskills.io) defines a portable skill format.
**MetaAgents** defines how they relate to each other — frontmatter shape, naming, dependency declarations, MCP packaging rules, and resolution semantics — as a self-contained format spec.

## Core Concepts

| Concept | File | Description |
|---------|------|-------------|
| **Agent** | `AGENTS.md` | An execution template — combines instructions with skills and MCPs |
| **Skill** | `SKILL.md` | A reusable capability package — methodology, knowledge, or workflow |
| **MCP** | `<namespace>_<short>.json` | A [Model Context Protocol](https://modelcontextprotocol.io) server configuration. Public servers are catalogued in the [MCP Registry](https://registry.modelcontextprotocol.io). |

Agents depend on skills and MCPs. Skills can depend on other skills and MCPs. MCPs are always leaf nodes.

```
Agent A
├── Skill X
│   ├── Skill Y
│   └── MCP P
├── Skill Z
└── MCP Q
```

## Directory Layout

A catalog has a flat three-bucket layout:

```
<catalog-root>/
  agents/<name>/AGENTS.md         (+ any sibling files)
  skills/<name>/SKILL.md          (+ scripts/, templates/, references/, hooks/, etc.)
  mcps/<namespace>_<short>.json
```

Rules:

- The folder name MUST equal `frontmatter.name` (kebab-case, lowercase `[a-z0-9]+(-[a-z0-9]+)*`, no `/`).
- A catalog typically shares one `scope:` value across its entries (e.g. `acme`); the scope is per-catalog convention, not part of the schema. Different catalogs use different scopes.
- MCP filenames replace `/` in the FQN with `_` for cross-platform compatibility — the file whose `_meta.name` is `io.example/mcp` lives at `mcps/io.example_mcp.json`.

## Naming Rules

Frontmatter `name` and `scope` are **separate fields** (see Frontmatter sections below). The fully-qualified name is computed as `<scope>/<name>` when both are present.

| Field | Grammar | Length | Notes |
| --- | --- | --- | --- |
| `name` | `^[a-z0-9]+(-[a-z0-9]+)*$` | ≤ 64 chars | identifier within a scope; no `/` |
| `scope` | `^[a-z0-9]+(-[a-z0-9]+)*(\.[a-z0-9]+(-[a-z0-9]+)*)*$` | ≤ 64 chars | reverse-DNS allowed (e.g. `io.example`) |

Do not write `name: "<scope>/<name>"` — `scope` and `name` are parsed independently.

### MCP names

MCP `_meta.name` is the MCP spec FQN. Reverse-DNS namespaces are preferred (`io.example/mcp`); single-segment vendor names (`acme/cli`, `azure/mcp`) are also accepted. MCP names follow the validation rules of the [official MCP registry](https://registry.modelcontextprotocol.io) and are more permissive than the agent/skill `name` field — mixed case, underscores, and longer identifiers (up to 200 chars) are all permitted.

## Frontmatter — Skill (`skills/<name>/SKILL.md`)

```yaml
---
name: my-skill                                  # required, kebab-case, matches folder
scope: example-org                              # optional but typical for published skills
description: "What the skill does, one line."   # required, 1-1024 chars
version: 1.0.0                                  # required, 3-segment semver (bare or quoted)
prereqs: |                                      # optional, skill-only
  Requires: <one-line summary>. See `references/SETUP.md` for step-by-step setup.
dependencies:                                   # optional
  skills:
    - "https://github.com/<owner>/<repo>/tree/<ref>/skills/<other-skill>"
  mcps:
    - "https://github.com/<owner>/<repo>/tree/<ref>/mcps/<file>.json"
---
# Skill body (markdown, verbatim)
```

Required: `name`, `description`, `version`. Optional: `scope`, `prereqs`, `dependencies.skills`, `dependencies.mcps`.

Field rules:

- `description` is one short sentence.
- `version` is mandatory and must be 3-segment semver. Both bare (`1.0.0`) and quoted (`"1.0.0"`) forms are accepted.
- `prereqs` is a YAML literal-block string. Keep it short — link to a sibling `references/SETUP.md` for the long version.
- `dependencies.skills` and `dependencies.mcps` are **arrays of bare origin URI strings**. Object form (`{origin: ...}`) is parsed but discouraged.
- Dependencies may reference any public catalog — entries from other publishers are allowed by URL.

## Frontmatter — Agent (`agents/<name>/AGENTS.md`)

Same shape as Skill, with one difference: agents reject `prereqs`.

```yaml
---
name: my-agent
scope: example-org
description: "What the agent does, one line."
version: 1.0.0
dependencies:
  skills:
    - "https://github.com/<owner>/<repo>/tree/<ref>/skills/git-pr"
  mcps:
    - "https://github.com/<owner>/<repo>/tree/<ref>/mcps/io.playwright_mcp.json"
---
# Agent body (markdown — instructions, playbook, etc.)
```

Required: `name`, `description`, `version`. Optional: `scope`, `dependencies.skills`, `dependencies.mcps`.

`prereqs` is **rejected** on agents. If your agent needs setup steps, put them in the body as a `## Setup` section.

## Frontmatter / format — MCP (`mcps/<namespace>_<short>.json`)

```json
{
  "_meta": {
    "name": "<namespace>/<short>"
  },
  "type": "stdio",
  "command": "...",
  "args": ["..."]
}
```

Required `_meta.*`:

- `_meta.name` is the MCP FQN. Reverse-DNS namespaces are preferred; single-segment vendor names are accepted. See [MCP names](#mcp-names) above.

Filename rule: the on-disk filename is `<namespace>_<short>.json` (replace `/` in the FQN with `_`). For example, the MCP whose `_meta.name` is `io.example/mcp` lives at `mcps/io.example_mcp.json`.

Other top-level fields (`type`, `command`, `args`, `env`, …) follow the [MCP client-config convention](https://modelcontextprotocol.io). Other `_meta.*` keys (e.g. registry sub-objects) survive untouched on re-write.

Files MUST be pretty-printed with 2-space indent and a trailing newline.

### MCP cross-platform rules

The MCP spec at modelcontextprotocol.io has **no** shell-style variable expansion: `command` is an executable name, `args` is an array of literal strings, `env` is an explicit map. Wrapping commands in `bash -c "..."` to get `$HOME` / `$PATH` expansion is a tempting workaround on POSIX that **breaks Windows immediately** (no `bash` on PATH; no POSIX env var names). MCP specs in this format MUST be cross-platform.

Four rules:

1. **`command` is a bare executable name** — `npx`, `node`, `python`, `uvx`. Let the OS PATH resolve it (Windows ships `npx.cmd` shims for Node tooling; the same name works on every host). Do NOT hardcode `bash`, `/usr/bin/...`, or any other absolute interpreter.
2. **No shell wrappers** — `["bash", "-c", "..."]` and friends are forbidden. If you need command composition, write a small `node` script inside your MCP project and call it directly.
3. **`args` are literal strings** — no `$HOME`, no `${VAR}`, no `~/`. The MCP server receives every arg verbatim.
4. **For paths that can't be hardcoded, use placeholder substitution** (see below). Hosts substitute these at provision time, before the MCP child is spawned, so the path the server sees is already absolute and platform-correct.

### Placeholder substitution

Two well-known placeholders are supported in any string field of an MCP spec (`command`, any element of `args`, any value of `env`, plus nested strings inside any custom object you put in the spec):

| Placeholder | Resolves to | Use for |
| --- | --- | --- |
| `${workspaceDir}` | The absolute path of the active workspace | State scoped to a single project (per-workspace cookies, repo-local credentials, browser login state that should reset between projects) |
| `${sharedDir}` | A stable per-host directory chosen by the runtime | State that genuinely belongs to the user account, not any single project (a global API token cache, a shared CA bundle, model weights downloaded once per machine) |

Hosts substitute both before writing the resolved `.mcp.json` to disk. The substituted paths use forward slashes regardless of host OS, so the same JSON value bytes ship to Windows and POSIX. A typo in a placeholder (`${workspceDir}`) MUST be rejected at install time with a clear error — placeholders are not silently passed through.

Pick `${sharedDir}` over `${workspaceDir}` only when the state genuinely belongs to the user account rather than the project — e.g. a model download cache or a global API token jar.

#### Example

```json
{
  "_meta": {
    "name": "io.playwright/mcp"
  },
  "type": "stdio",
  "command": "npx",
  "args": [
    "-y",
    "@playwright/mcp@latest",
    "--headless",
    "--storage-state",
    "${workspaceDir}/.playwright/storage-state.json"
  ]
}
```

## Origin URI Grammar

Dependency origins (`dependencies.skills`, `dependencies.mcps`) are bare URI strings. Two schemes are accepted:

- `https://github.com/<owner>/<repo>/tree/<ref>[/path]` — recommended for shared catalog entries; supports any public GitHub repo
- `file:<absolute-path>` — local-only; never commit a `file:` origin

## CHANGELOG Conventions

Every published agent and skill ships a `CHANGELOG.md` next to its `AGENTS.md` / `SKILL.md`.

- Version headers use `## X.Y.Z (YYYY-MM-DD)` format. Example: `## 1.2.0 (2026-04-17)`.
- Bump guidance:
  - **patch (`X.Y.Z+1`)** — bug fixes, typos, minor edits that don't change behavior or the public surface
  - **minor (`X.Y+1.0`)** — new features, behavioral additions, new optional dependencies
  - **major (`X+1.0.0`)** — breaking changes (rename, dropping a public dependency, removing a tool, semantic change to a workflow that downstream consumers rely on)
- One version bump per change set per agent/skill — do not bump version multiple times within a single PR.
- Frontmatter `version` MUST match the latest entry in `CHANGELOG.md`.
- Sections that document past renames or breaking changes are **provenance** — do not delete them when later versions move on.

## Resolution Rules

1. **Topological order** — dependencies are resolved depth-first
2. **No cycles** — circular dependencies are invalid
3. **Missing dependencies** — resolution fails if a declared dependency origin is unreachable or returns 404
4. **Name uniqueness** — the fully-qualified name (including scope if present) MUST be unique within each type (agents, skills, mcps) at any single resolution layer
5. **MCPs are leaves** — MCP entries cannot declare dependencies

## Backward Compatibility

MetaAgents extends [agents.md](https://agents.md) and [Agent Skills](https://agentskills.io) with: separate `scope:` field, mandatory `version`, `dependencies` block, and the placeholder/cross-platform rules for MCPs. An `AGENTS.md` or `SKILL.md` that omits the MetaAgents-only fields still parses as a valid agents.md / Agent Skills file; tools that don't understand MetaAgents simply ignore the extra fields.

## Out of Scope

MetaAgents is format-level only. It does **not** define:

- How to fetch or publish packages (that's for package managers and runtimes)
- How to provision or execute agents (that's for runtimes)
- How to interpret instructions (that's for LLMs)
- How to spawn MCP servers (that's for substrates)
- Authoring conventions for agent body sections (different runtimes use different playbook structures)

## License

[MIT](LICENSE)
