# BitDive Skills

[![Skills](https://img.shields.io/badge/Skills-Codex%20%7C%20Claude%20Code-111111)](#repository-layout)
[![MCP](https://img.shields.io/badge/MCP-BitDive%20Workflows-1F6FEB)](#what-this-repository-is-for)
[![Install](https://img.shields.io/badge/Install-npx%20skills%20add-0A7EA4)](#installation)
[![Format](https://img.shields.io/badge/Format-SKILL.md%20%7C%20Claude%20agents-6DB33F)](#repository-layout)

Public BitDive workflows packaged for agent use.

This repository contains reusable BitDive skills for runtime trace analysis, MCP-driven investigation, service instrumentation, regression management, and developer workflows. It is designed for teams who want to install BitDive guidance into agent environments such as Codex or Claude Code, instead of rewriting prompts and runbooks by hand.

## Demo

[![Watch the BitDive demo](https://img.youtube.com/vi/WqtLXcODz8I/maxresdefault.jpg)](https://www.youtube.com/watch?v=WqtLXcODz8I)

Watch the BitDive product demo on YouTube:

- https://www.youtube.com/watch?v=WqtLXcODz8I

## What This Repository Is For

Use this repository when you want agent-ready BitDive workflows for tasks such as:

- adding BitDive to a Spring Boot service
- understanding which MCP tool to call and when
- comparing traces to find behavioral drift
- running phased BitDive development workflows
- repairing or refreshing replay-based regression groups
- connecting Dockerized services to BitDive infrastructure

These workflows are meant to make BitDive usage repeatable, faster, and less prompt-dependent.

## Repository Layout

The repository ships the same ideas in two formats.

| Path | Purpose |
| --- | --- |
| `skills/<skill-name>/SKILL.md` | Portable installable skills for agent ecosystems that support the `skills` format |
| `skills/<skill-name>/agents/openai.yaml` | Agent-facing metadata for the portable skill package |
| `claude/agents/<agent-name>.md` | Native Claude Code subagents |

## Included Workflows

| Workflow | Purpose |
| --- | --- |
| [`add-bitdive-spring`](skills/add-bitdive-spring/SKILL.md) | Add BitDive producer and optional replay support to a Spring Boot service |
| [`bitdive-overview`](skills/bitdive-overview/SKILL.md) | Choose the right BitDive MCP tool for discovery, inspection, and updates |
| [`bitdive-trace-comparison`](skills/bitdive-trace-comparison/SKILL.md) | Compare traces and locate the exact point of behavior drift |
| [`bitdive-dev-workflow`](skills/bitdive-dev-workflow/SKILL.md) | Run a phased BitDive development workflow with human checkpoints |
| [`bitdive-test-management`](skills/bitdive-test-management/SKILL.md) | Create, wire, repair, refresh, and rebuild replay groups |
| [`bitdive-docker-networking`](skills/bitdive-docker-networking/SKILL.md) | Connect Dockerized services to cloud or self-hosted BitDive |

## Which Format To Use

Use the format that matches your agent environment:

- use `skills/` if you want one-command install through `npx skills add`
- use `claude/agents/` if you specifically want native Claude Code agents in `.claude/agents/`
- workflow names are intentionally aligned across both formats

## Installation

### Interactive install

Install the BitDive portable skills and choose the target agent interactively:

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

Install the full portable set into both agents:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a codex -a claude-code
```

Global install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g -a codex -a claude-code
```

## Claude Code Native Subagents

If you specifically want native Claude Code subagents rather than the portable `skills/` format:

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

## Recommended Environment

These workflows are most useful when:

- BitDive MCP is configured and reachable
- the target service is already instrumented or is about to be instrumented
- the repository has a real module-scoped test command that can be rerun quickly
- the team wants repeatable runtime-backed workflows rather than one-off prompts

## Design Notes

This repository is intentionally text-first and workflow-oriented.

- the skills are meant to be portable across repositories
- private project names and local-only paths were removed from the public set
- MCP-centered investigation and replay management are the main focus
- each workflow is intended to reduce prompt ambiguity for common BitDive tasks

## Notes

- `agents/openai.yaml` is included for each portable skill so the set is ready for UI-facing use
- Claude support is stored as visible source files in `claude/agents/`
- if you want to publish under another license, replace the root `LICENSE` file before pushing the repository
