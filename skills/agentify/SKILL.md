---
name: agentify
description: "Create Claude Code subagent files for a project by analyzing the codebase and turning useful roles/workflows into .claude/agents/*.md files."
when_to_use: "Use when the user wants to create Claude Code agent files, subagents, project agents, or a reusable agent setup for a repo. Examples: '/agentify', 'make agents for this project', 'create subagents for this repo', 'agentify this codebase', 'build Claude agents for this project'."
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - LS
  - AskUserQuestion
  - Bash(pwd:*)
  - Bash(git rev-parse:*)
  - Bash(mkdir:*)
  - Bash(test:*)
  - Bash(ls:*)
  - Bash(find:*)
user-invocable: true
disable-model-invocation: true
argument-hint: "[project path or agent brief]"
arguments:
  - brief
---

# Agentify

Create Claude Code subagent files for the current project or for the project named in `$brief`.

## Goal

Produce a focused set of Claude Code subagent Markdown files that are useful for the project, valid for Claude Code, and saved in the right scope:

- Project agents by default: `.claude/agents/<agent-name>.md`
- Personal agents only when the user explicitly asks: `~/.claude/agents/<agent-name>.md`

Claude Code subagents are Markdown files with YAML frontmatter. Only `name` and `description` are required. Common optional fields are `tools`, `disallowedTools`, `model`, `permissionMode`, `maxTurns`, `skills`, `mcpServers`, `hooks`, `memory`, `background`, `effort`, `isolation`, `color`, and `initialPrompt`.

## Inputs

- `$brief`: Optional project path, agent brief, or constraints. If omitted, use the current working directory and infer the agent set from the project.

## Hard Rules

- Do not create `AGENTS.md`, `CLAUDE.md`, commands, hooks, or settings unless the user explicitly asks for those files. This skill creates Claude Code subagent files.
- Default to project scope. Project-specific agents belong in `.claude/agents/` so they can be committed and shared.
- Do not overwrite an existing agent file without a human checkpoint.
- Let the project decide the count. Minimum 1, no upper bound. Each agent must point to evidence from the scan (a real workflow, file, command, or convention); drop any that cannot. Zero is a valid outcome if the project does not justify any agents. Do not pad to hit a number.
- Use project-specific prompts. Generic agents are only acceptable when the project truly lacks distinguishing architecture, tooling, or workflows.
- Restrict tools by role. Read-only reviewers should not get `Write` or `Edit`. Implementers can get write tools only when their job needs them.
- A subagent body is its system prompt. It does not inherit this skill, the full main Claude Code system prompt, or the conversation. Put all necessary role instructions in the file.
- If writing files directly on disk, remind the user that existing Claude Code sessions may need a restart for newly edited subagents to load.

## Subagent File Format

Use this structure for each file. Mirror the canonical write order Claude Code itself uses (`name`, `description`, `tools`, `model`, `effort`, `color`, `memory`):

```markdown
---
name: project-role
description: "Use this agent when ... <example>Context: ...\nuser: \"...\"\nassistant: \"...\"\n<commentary>...</commentary></example>"
tools: Read, Glob, Grep
model: inherit
color: blue
---

You are ...
```

Field rules (validated by Claude Code):

- `name` (required): regex `^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$`. Length 3-50. Must start AND end with an alphanumeric. Lowercase + hyphens is the convention; uppercase is permitted but avoid it. Filename must equal `<name>.md`.
- `description` (required): YAML double-quoted, escape newlines as `\n`. Should start with `Use this agent when ...`. Soft length window 10-5000 chars. Include one or more `<example>` blocks (see Step 4) so auto-delegation has concrete trigger scenarios.
- `tools` (optional): comma-separated tool names, e.g. `Read, Glob, Grep`. **Omit the field entirely** to inherit all tools (do not write `tools: *`). Read-only roles must not get `Write` or `Edit`.
- `model` (optional): one of `sonnet | opus | haiku | inherit`. Default to `inherit` unless a specific tier is justified. Use `haiku` for cheap classification, `opus` only when the role demands deep reasoning.
- `effort` (optional): an effort level string or a positive integer. Omit unless tuning.
- `color` (optional): one of `red | blue | green | yellow | purple | orange | pink | cyan`.
- `memory` (optional): one of `user | project | local`. Add only when the agent will benefit from persistent notes across runs (reviewers, architects, etc.).

Reader-only fields (parsed if present, but not part of the canonical write set; only include when explicitly needed):

- `disallowedTools`: comma-separated tool names to block.
- `permissionMode`: `own | approve | plan | chat`.
- `maxTurns`: positive integer.
- `isolation`: `worktree`.
- `background`: boolean.
- `initialPrompt`: string prepended to the first user turn.
- `skills`, `mcpServers`, `hooks`: include only when the user requests them or the project clearly needs them.

## Authoring Discipline (mirrors Claude Code's own AGENT_CREATION_SYSTEM_PROMPT)

When drafting each agent, follow the same rules Claude Code's built-in agent-creation wizard follows. These are not stylistic preferences; they are how the parent Claude decides whether to delegate.

1. **Identifier**: lowercase letters/numbers/hyphens, typically 2-4 words joined by hyphens. Avoid generic terms like `helper`, `assistant`, `manager`, `agent`. Make it concrete (`migration-checker`, not `db-helper`).
2. **Description starts with `Use this agent when ...`** and embeds one or more `<example>` blocks. The example shape is:
   ```
   <example>
   Context: <one-line situation>
   user: "<user message>"
   assistant: "I'm going to use the Task tool to launch the <name> agent to ..."
   <commentary>
   <one-line reason this agent fires here.>
   </commentary>
   </example>
   ```
   If the agent should fire proactively (without the user asking), say so in the description and add an example showing proactive invocation.
3. **System prompt is in second person**: "You are ...", "You will ...". Not "the agent should". The body IS the agent's full operating manual; it does not see this skill, the parent's CLAUDE.md hierarchy automatically (depending on `omitClaudeMd`), or the parent conversation.
4. **Code-review agents default to "recently written code"**, not the whole codebase. State this explicitly in the body unless the agent is intentionally a full-repo auditor.
5. **Memory pattern** (only when `memory:` is set): include a section telling the agent what to record. Template:
   > **Update your agent memory** as you discover [domain items]. This builds institutional knowledge across runs. Examples of what to record: [item 1], [item 2], [item 3].
6. **Avoid vague instructions.** Reference real paths, real commands, real conventions from the scan. "Run the tests" is dead weight; "Run `pnpm test --filter=api` from repo root" is useful.

## Steps

### 1. Resolve Scope and Project Root

Use `$brief` to detect whether the user provided a path, a desired agent set, or global/personal scope.

If a path was provided, use it. Otherwise:

1. Run `pwd`.
2. Try `git rev-parse --show-toplevel`.
3. If git root detection fails, use the current working directory.

Choose the target directory:

- If the user says personal, global, user-level, or reusable across all repos: `~/.claude/agents/`
- Otherwise: `<project-root>/.claude/agents/`

**Success criteria**: Project root and target agent directory are identified. Any ambiguity about scope has been resolved before writing.

### 2. Analyze the Project

Read enough of the project to design useful agents:

- Existing Claude files: `CLAUDE.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`, `.claude/commands/*.md`
- Repo overview: `README*`, package manifests, build files, test configs, CI configs, docs index files
- Source layout: top-level directories, important app packages, test directories, scripts, migration/config directories
- Existing conventions: framework, language, test commands, lint commands, deployment flow, review rules, security-sensitive areas

Avoid reading generated/vendor directories such as `node_modules`, `.git`, `dist`, `build`, `coverage`, `.next`, `target`, `.venv`, `vendor`, and binary assets.

**Success criteria**: You can state the project's main stack, major workflows, existing Claude configuration, and the boundaries that justify each proposed agent.

### 3. Propose the Agent Set

Propose a compact list before writing files. For each agent, include:

- `name`
- target path
- purpose
- when Claude should use it
- tool list
- model and color, if used
- what the agent returns
- why this agent belongs in this project

Use `AskUserQuestion` for confirmation. Offer concrete choices such as:

- Save proposed agents
- Revise names/scopes
- Create fewer agents

Let the user provide freeform edits. If `AskUserQuestion` is unavailable, ask the same checkpoint in plain text.

**Success criteria**: The user has confirmed the proposed files or provided edits that have been incorporated.

**Human checkpoint**: Required before writing new files if more than one agent will be created or any existing agent name conflicts.

### 4. Draft Each Agent

Each agent body should include:

1. Role and scope.
2. When to act and when to hand back to the main conversation.
3. Project-specific files, commands, directories, and conventions to inspect first.
4. Step-by-step working method.
5. Hard constraints, especially around edits, tests, secrets, migrations, deployments, and user-owned changes.
6. Expected output format.

Good project agent types include:

- Architecture explorer: read-only map of code paths, ownership, data flow, and risky areas.
- Code reviewer: read-only review with findings first and file/line references.
- Test/debug specialist: runs targeted tests, explains failures, and proposes or applies fixes depending on tools.
- Implementation worker: owns a bounded part of the codebase and writes changes under explicit scope.
- Frontend/UI specialist: handles UI implementation and visual verification when the project has a frontend.
- API/database specialist: handles contracts, migrations, schemas, queries, and integration risks.
- Release/CI specialist: checks workflows, build scripts, changelogs, and deployment readiness.

Avoid agents that duplicate Claude's normal behavior without project-specific value.

**Success criteria**: Every draft has valid frontmatter, a non-generic system prompt, scoped tools, and a clear output contract.

### 5. Write Files

Create the target directory with `mkdir -p`.

For each agent:

- Save to `<target-dir>/<name>.md`.
- If the file exists, read it and ask before overwriting.
- Match filename to `name`.
- Use ASCII unless the existing project convention requires otherwise.

**Success criteria**: All confirmed agent files exist at the target path and contain the approved content.

### 6. Validate

Read the written files back. These checks mirror Claude Code's own `validateAgent.ts` and `parseAgentFromMarkdown` and will reject the file if they fail:

Frontmatter:
- Starts and ends with `---`.
- `name` matches `^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$`, length 3-50.
- `name` equals filename without `.md` (case-sensitive).
- `name` is unique across the target agents directory.
- `description` is present, double-quoted, with newlines escaped as `\n`. Soft length window 10-5000 chars.
- `description` starts with `Use this agent when ...` and embeds at least one `<example>` block.
- `tools` (if present) is a comma-separated list of real Claude Code tool names. Read-only roles do not include `Write`, `Edit`, `NotebookEdit`, or write-capable Bash patterns.
- `model` (if present) is one of `sonnet | opus | haiku | inherit`.
- `color` (if present) is one of `red | blue | green | yellow | purple | orange | pink | cyan`.
- `memory` (if present) is one of `user | project | local`.
- `permissionMode` (if present) is one of `own | approve | plan | chat`.
- `isolation` (if present) is `worktree`.
- `maxTurns` (if present) is a positive integer.

Body:
- Written in second person ("You are ...", "You will ...").
- References real paths/commands/conventions from the scan; no generic filler.
- Has an explicit output contract (what the parent gets back).
- Memory section present iff `memory:` is set.

Filesystem:
- Existing files were not overwritten without confirmation.
- Directory is `<project-root>/.claude/agents/` (or `~/.claude/agents/` if user explicitly chose personal scope).

Tell the user they can list installed subagents with:

```bash
claude agents
```

or open the in-app manager with `/agents`.

**Success criteria**: Every check above passes for every written file. Any failure is fixed before reporting completion.

## Final Response

Report:

- The directory where files were saved.
- The agent filenames created or updated.
- Any existing files left untouched.
- That Claude Code may need a restart to load subagents written directly to disk.
- How to invoke one explicitly, e.g. `Use the <agent-name> subagent to ...`.
