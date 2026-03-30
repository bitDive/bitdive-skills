# BitDive Skills

Public BitDive workflows packaged in two formats:

- `skills/` for portable agent skills installed with `npx skills add`
- `claude/agents/` for Claude Code subagents

The set is focused on BitDive integration, runtime trace analysis, and
trace-based regression workflows.

## Repository Layout

```text
skills/<skill-name>/SKILL.md
skills/<skill-name>/agents/openai.yaml
claude/agents/<agent-name>.md
```

## Included Workflows

| Workflow | Purpose |
|---|---|
| [`add-bitdive-spring`](skills/add-bitdive-spring/SKILL.md) | Add BitDive producer and optional replay support to a Spring Boot service |
| [`bitdive-overview`](skills/bitdive-overview/SKILL.md) | Choose the right BitDive MCP tool for discovery, inspection, and updates |
| [`bitdive-trace-comparison`](skills/bitdive-trace-comparison/SKILL.md) | Compare traces and locate the exact point of behavior drift |
| [`bitdive-dev-workflow`](skills/bitdive-dev-workflow/SKILL.md) | Run a phased BitDive development workflow with human checkpoints |
| [`bitdive-test-management`](skills/bitdive-test-management/SKILL.md) | Create, wire, repair, refresh, and rebuild replay groups |
| [`bitdive-docker-networking`](skills/bitdive-docker-networking/SKILL.md) | Connect Dockerized services to cloud or self-hosted BitDive |

## Installation

`skills` CLI treats `--all` as "install all skills to all agents". To install
the full BitDive set only for selected agents, use `--skill '*'` with explicit
`-a` flags.

### Interactive install

Let the installer ask which agent you want to use:

```bash
npx skills add bitDive/bitdive-skills --skill '*'
```

Global interactive install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g
```

### Codex

Install the full portable skill set into Codex:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a codex
```

Global install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g -a codex
```

### Claude Code

Install the same portable skill set into Claude Code:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a claude-code
```

Global install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g -a claude-code
```

### Codex + Claude Code

Install the full portable set into both agents with one command:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a codex -a claude-code
```

Global install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g -a codex -a claude-code
```

### Claude Code Native Subagents

Claude-native subagents also live in `claude/agents/`.

Clone the repository, enter it, then copy the subagents you want into a Claude
Code project:

```bash
git clone https://github.com/bitDive/bitdive-skills.git
cd bitdive-skills
mkdir -p /path/to/project/.claude/agents
cp claude/agents/<agent-name>.md /path/to/project/.claude/agents/
```

Optional user-wide install:

```bash
mkdir -p ~/.claude/agents
cp claude/agents/<agent-name>.md ~/.claude/agents/
```

Use this path only if you specifically want native Claude subagents. For
one-command install across agents, use the `skills/` format above.

## Which Format To Use

- Use the `skills/` format if you want `npx skills add bitDive/bitdive-skills` for Codex, Claude Code, or both.
- Use `claude/agents/` only if you specifically want native Claude Code subagents in `.claude/agents/`.
- The workflow names are intentionally aligned across both formats.

## Recommended Environment

These workflows are most useful when:

- BitDive MCP is configured and reachable
- the target service is already instrumented or about to be instrumented
- the repository has a real module-scoped test command you can rerun quickly

## Notes

- `agents/openai.yaml` is included for each skill so the set is ready for UI-facing use.
- Claude support is stored as visible source files in `claude/agents/`, not hidden repo-local config.
- The workflows are text-first and repository-agnostic. Project-specific names and private paths were removed.
- If you want to publish under another license, replace the root `LICENSE` file before pushing the repository.
