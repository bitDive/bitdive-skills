# BitDive Skills

Public Codex skills and Claude Code subagents for BitDive integration,
trace analysis, and replay-test workflows.

## What This Repository Contains

This repository packages the same BitDive workflows in two GitHub-friendly
formats:

```text
skills/<skill-name>/SKILL.md
skills/<skill-name>/agents/openai.yaml
.claude/agents/<agent-name>.md
```

The skills are grouped around four common jobs:

- onboarding BitDive into Spring Boot services
- comparing runtime traces and debugging behavior drift
- managing replay baselines and test groups
- connecting services to the right BitDive environment

## Included Skills

| Skill | Purpose |
|---|---|
| `add-bitdive-spring` | Add BitDive producer and optional replay support to a Spring Boot service |
| `bitdive-overview` | Choose the right BitDive MCP tool for discovery, inspection, and updates |
| `bitdive-trace-comparison` | Compare traces and locate the exact point of behavior drift |
| `bitdive-dev-workflow` | Run a phased BitDive development workflow with human checkpoints |
| `bitdive-test-management` | Create, wire, repair, refresh, and rebuild replay groups |
| `bitdive-docker-networking` | Connect Dockerized services to cloud or self-hosted BitDive |

## Installation

### Codex / OpenAI Skills

#### User-wide install

Copy one or more skill directories into `~/.codex/skills/`:

```bash
mkdir -p ~/.codex/skills
cp -R skills/<skill-name> ~/.codex/skills/
```

#### Repo-local install

Copy one or more skill directories into a repository-local `.agents/skills/`
folder:

```bash
mkdir -p .agents/skills
cp -R skills/<skill-name> .agents/skills/
```

Restart Codex after installing new skills so they are picked up cleanly.

### Claude Code Subagents

This repository also includes Claude Code project subagents in `.claude/agents/`.

#### Project install

Copy one or more agent files into your project's `.claude/agents/` directory:

```bash
mkdir -p /path/to/project/.claude/agents
cp .claude/agents/<agent-name>.md /path/to/project/.claude/agents/
```

#### User-wide install

Copy one or more agent files into `~/.claude/agents/`:

```bash
mkdir -p ~/.claude/agents
cp .claude/agents/<agent-name>.md ~/.claude/agents/
```

Claude Code uses subagents as Markdown files with YAML frontmatter. In this
repository, Claude support is implemented as subagents rather than slash
commands because that is the closest analogue to task-specific skills.

## Recommended Environment

Most of these skills are most useful when:

- BitDive MCP is configured and reachable
- the target service is already instrumented or about to be instrumented
- the repository has a real module-scoped test command you can rerun quickly

These skills are still useful without a full BitDive setup, but they are
designed for runtime-first, BitDive-enabled workflows.

## Notes

- `agents/openai.yaml` is included for each skill so the set is ready for UI-facing use.
- `.claude/agents/` contains the parallel Claude Code format for the same workflows.
- The skills are intentionally text-first and repository-agnostic. Project-specific names and private paths were removed.
- If you want to publish under another license, replace the root `LICENSE` file before pushing the repository.
