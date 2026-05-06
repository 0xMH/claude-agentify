# claude-agentify

Claude Code ships a built-in agent-creation wizard at `/agents`. It walks you through one agent at a time: type, description, prompt, model, color, tools, memory, location, confirm. The system prompt it uses to generate the agent for you is `AGENT_CREATION_SYSTEM_PROMPT` in the binary, and it has a lot of opinions about what a good subagent looks like (identifier rules, `Use this agent when ...` descriptions with `<example>` blocks, second-person prompts, "recently written code" defaults for reviewers, etc.).

That wizard is fine for one-off agents. But if you sit down in a fresh repo and want a useful set of subagents, you don't want to run the wizard six times and answer the same context questions. You want something that reads the repo, figures out what a good set of agents would be, and writes them all in one shot.

So I built `/agentify`. It scans the project (manifests, CI, CLAUDE.md, source layout, existing `.claude/` setup), proposes a set of agents tailored to what it finds, and writes them to `.claude/agents/*.md` after you confirm. The authoring discipline mirrors `AGENT_CREATION_SYSTEM_PROMPT` and the validation mirrors `validateAgent.ts` so the output is the same shape Claude Code's own wizard produces.

## What it does

Given a project directory (defaults to your current working directory):

1. Resolve the project root via `pwd` / `git rev-parse --show-toplevel`.
2. Scan the codebase: `CLAUDE.md`, `AGENTS.md`, `README*`, package manifests, CI configs, source layout, existing `.claude/agents/` and `.claude/skills/`.
3. Propose a set of subagents. Each one points to evidence from the scan (a real workflow, file, command, or convention). Minimum one. Zero is a valid outcome if the project doesn't justify any. No padding to hit a count.
4. Wait for your confirmation. No files written before you say yes.
5. Write each agent to `<project>/.claude/agents/<name>.md` with canonical frontmatter (`name`, `description`, `tools`, `model`, `effort`, `color`, `memory` in that order) and a second-person system prompt that references real paths and conventions from the scan.
6. Validate against the real rules: name regex `^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$` (length 3-50), enum-checked `model`/`color`/`memory`/`permissionMode`/`isolation`, description starting with `Use this agent when ...` with at least one `<example>` block.

## Install

Add the marketplace and install the plugin in Claude Code:

```
/plugin marketplace add 0xMH/claude-agentify
/plugin install agentify@0xMH/claude-agentify
```

Or test locally during development:

```
claude --plugin-dir ./path/to/claude-agentify
```

## Usage

From inside a project:

```
/agentify
```

Or pass a path or a brief:

```
/agentify ~/code/my-app
/agentify make me a migration checker and a release auditor
```

After it writes files to `.claude/agents/`, you can use the agents three ways:

1. **Auto-delegation**: the main Claude reads each agent's `description` and routes matching tasks to it. The `<example>` blocks in the description are what drives this, so they matter.
2. **Explicit**: `Use the <agent-name> subagent to ...`
3. **`/agents`**: list, view, edit, invoke from the built-in UI.

Newly written agents may need a session restart to load.

## How it differs from the bundled wizard

The built-in `/agents` wizard creates one agent per run and asks you for every dimension. `agentify`:

- Reads the project and proposes a tailored set, so you confirm once instead of running the wizard N times.
- Forces evidence per agent. If a proposed agent can't point to something in the scan, it gets dropped. No generic `code-reviewer` filler.
- Restricts tools by role automatically (read-only roles do not get `Write`/`Edit`).
- Mirrors the `AGENT_CREATION_SYSTEM_PROMPT` discipline (identifier rules, `Use this agent when ...` descriptions with `<example>` blocks, second-person prompts, code-review "recently written" default, memory section pattern when `memory:` is set).
- Writes the canonical field set (`name`, `description`, `tools`, `model`, `effort`, `color`, `memory`) in the same order Claude Code's own writer uses.

## Subagent file format reference

Agents written by agentify follow this structure:

```yaml
---
name: project-role
description: "Use this agent when ... <example>Context: ...\nuser: \"...\"\nassistant: \"...\"\n<commentary>...</commentary></example>"
tools: Read, Glob, Grep
model: inherit
color: blue
---

You are ...
```

Field rules (enforced by Claude Code's own validator):

- `name` (required): regex `^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$`, length 3-50, must equal filename without `.md`.
- `description` (required): YAML double-quoted, newlines escaped as `\n`. Should start with `Use this agent when ...` and embed `<example>` blocks.
- `tools` (optional): comma-separated. Omit the field entirely to inherit all tools.
- `model` (optional): `sonnet | opus | haiku | inherit`.
- `color` (optional): `red | blue | green | yellow | purple | orange | pink | cyan`.
- `memory` (optional): `user | project | local`.
- `effort` (optional): enum string or positive integer.

Reader-only fields parsed if present: `disallowedTools`, `permissionMode`, `maxTurns`, `isolation`, `background`, `initialPrompt`, `skills`, `mcpServers`, `hooks`.

## License

MIT
