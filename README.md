# BitDive Skills

[![Skills](https://img.shields.io/badge/Skills-Claude%20Code-111111)](#repository-layout)
[![MCP](https://img.shields.io/badge/MCP-BitDive%20Workflows-1F6FEB)](#what-this-repository-is-for)
[![Install](https://img.shields.io/badge/Install-npx%20skills%20add-0A7EA4)](#installation)
[![Format](https://img.shields.io/badge/Format-SKILL.md-6DB33F)](#repository-layout)

Public BitDive workflows packaged for agent use.

This repository contains reusable BitDive skills for runtime trace analysis, MCP-driven investigation, service instrumentation, regression management, and developer workflows. Each skill is a single `SKILL.md` written to the open Agent Skills standard, so one source installs into Claude Code (and any other agent that supports the standard) instead of rewriting prompts and runbooks by hand.

## Demo

[![Watch the BitDive demo](https://img.youtube.com/vi/WqtLXcODz8I/maxresdefault.jpg)](https://www.youtube.com/watch?v=WqtLXcODz8I)

Watch the BitDive product demo on YouTube:

- https://www.youtube.com/watch?v=WqtLXcODz8I

## What This Repository Is For

Use this repository when you want agent-ready BitDive workflows for tasks such as:

- adding BitDive to a Spring Boot service
- understanding which MCP tool to call and when
- comparing traces to find behavioral drift
- reviewing a PR/branch with before/after runtime evidence
- hunting real runtime issues and rejecting false positives
- building trace coverage across a service's endpoints
- running phased BitDive development workflows
- repairing or refreshing replay-based regression groups
- fixing MCP, service, or Docker connectivity to BitDive

These workflows are meant to make BitDive usage repeatable, faster, and less prompt-dependent.

## Repository Layout

One skill, one folder, one `SKILL.md`. There is a single source of truth per skill — no duplicated per-agent copies.

| Path | Purpose |
| --- | --- |
| `skills/<skill-name>/SKILL.md` | The skill: YAML frontmatter (`name`, `description`) plus the workflow instructions |
| `skills/<skill-name>/agents/openai.yaml` | Optional UI metadata (display name, prompt) for installers that surface it |

The agent decides when and how to apply each skill from its `description` — and is free to use several together or run one in isolation as it sees fit.

## Included Workflows

| Workflow | Purpose |
| --- | --- |
| [`add-bitdive-spring`](skills/add-bitdive-spring/SKILL.md) | Add BitDive producer and optional replay support to a Spring Boot service |
| [`bitdive-overview`](skills/bitdive-overview/SKILL.md) | Choose the right BitDive MCP tool for discovery, inspection, comparison, and updates |
| [`bitdive-trace-comparison`](skills/bitdive-trace-comparison/SKILL.md) | Compare traces and locate the exact point of behavior drift |
| [`bitdive-pr-review`](skills/bitdive-pr-review/SKILL.md) | Review a PR/branch with before/after runtime evidence and a merge verdict |
| [`bitdive-issue-finding`](skills/bitdive-issue-finding/SKILL.md) | Hunt real runtime issues and separate verified bugs from false positives |
| [`bitdive-endpoint-coverage`](skills/bitdive-endpoint-coverage/SKILL.md) | Trigger endpoints safely to build trace coverage and document gaps |
| [`bitdive-dev-workflow`](skills/bitdive-dev-workflow/SKILL.md) | Run a phased BitDive development workflow with human checkpoints |
| [`bitdive-test-management`](skills/bitdive-test-management/SKILL.md) | Create, wire, repair, refresh, and rebuild replay groups |
| [`bitdive-connectivity-setup`](skills/bitdive-connectivity-setup/SKILL.md) | Fix MCP/service/Docker connectivity to the intended BitDive backend |
| [`bitdive-docker-networking`](skills/bitdive-docker-networking/SKILL.md) | Connect Dockerized services to cloud or self-hosted BitDive |

## Installation

### Claude Code

Install the full skill set into Claude Code with the Agent Skills CLI:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a claude-code
```

Global (user-wide) install:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -g -a claude-code
```

This places each skill under `.claude/skills/<skill-name>/` (or `~/.claude/skills/` for a global install). Claude discovers them automatically and applies a skill when its `description` matches the task.

### Manual install

If you prefer not to use the CLI, copy the skill folders directly:

```bash
git clone https://github.com/bitDive/bitdive-skills.git
mkdir -p /path/to/project/.claude/skills
cp -R bitdive-skills/skills/* /path/to/project/.claude/skills/
```

### Other agents

Because each skill follows the open Agent Skills standard, the same source installs into other supported agents by changing the target, for example:

```bash
npx skills add bitDive/bitdive-skills --skill '*' -a cursor
npx skills add bitDive/bitdive-skills --skill '*' -a codex
```

## Recommended Environment

These workflows are most useful when:

- BitDive MCP is configured and reachable
- the target service is already instrumented or is about to be instrumented
- the repository has a real module-scoped test command that can be rerun quickly
- the team wants repeatable runtime-backed workflows rather than one-off prompts

## Design Notes

This repository is intentionally text-first and workflow-oriented.

- one `SKILL.md` per skill is the single source of truth — no per-agent duplicates to keep in sync
- the skills are meant to be portable across repositories
- private project names and local-only paths were removed from the public set
- MCP-centered investigation and replay management are the main focus
- each workflow is intended to reduce prompt ambiguity for common BitDive tasks

## Notes

- `agents/openai.yaml` is optional metadata for installers that show a UI; it is not required for the skill to work
- if you want to publish under another license, replace the root `LICENSE` file before pushing the repository
