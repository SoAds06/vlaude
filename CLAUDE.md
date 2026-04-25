# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A local-first, markdown-only multi-agent system. No code, no database. Each agent is a folder of markdown files that define its mission, skills, memory, and schedule. Claude Code is the runtime.

```
Human → Orchestrator → Agents → Outputs
                           ↓
                        Journal (shared memory — all agents read/write here)
```

## Creating a New Agent

```bash
cp -r agents/standard-agent agents/[your-agent-name]
```

Then follow `NEW_AGENT_BOOTSTRAP.md` (Steps 2–9). Key steps:
1. Fill `AGENT.md` — mission (one sentence), KPIs (2–4 measurable), non-goals
2. Add one `.md` per skill in `skills/` using `agents/standard-agent/skills/_SKILL_TEMPLATE.md`
3. Fill `HEARTBEAT.md` — schedule, decision tree for which skill runs each cycle
4. Fill `RULES.md` — CAN/CANNOT boundaries, handoff conditions
5. Register the agent in `AGENT_REGISTRY.md`
6. Verify with `AGENT_CREATION_CHECKLIST.md`

See `examples/podcast-agent/` for a fully filled-out reference.

## Running an Agent (Manual Trigger)

There is no build step or test runner. To run an agent manually, open Claude Code and instruct it to execute the agent's heartbeat cycle:

```
"Run agents/[name]/HEARTBEAT.md — execute the daily cycle for today."
```

The agent will:
1. Read `journal/` for recent signals, `knowledge/` for reference, own `MEMORY.md` for learnings
2. Assess the current pipeline and pick the right skill
3. Execute the skill, write output to `agents/[name]/outputs/YYYY-MM-DD_description.md`
4. Log a journal entry to `journal/entries/YYYY-MM-DD_HHMM.md`

To schedule recurring runs, use `/schedule` in Claude Code to wire the heartbeat to a cron.

## Data Flow Rules

| Layer | Path | Who writes | Who reads |
|-------|------|-----------|-----------|
| Static reference | `knowledge/` | Human only | All agents |
| Shared memory | `journal/entries/` | All agents | All agents |
| Agent memory | `agents/[name]/MEMORY.md` | That agent only | That agent only |
| Outputs | `agents/[name]/outputs/` | That agent | Human |
| Orchestrator | `orchestrator/PRIORITIES.md` | Human | Orchestrator |

Agents **never** write to `knowledge/` — they propose changes via the journal or by flagging to the human.

## Orchestrator

`orchestrator/IDENTITY.md` defines the orchestrator's role: route tasks to the right agent, flag overlaps, suggest new agents. It is always-on (not scheduled). When a task doesn't fit any existing agent, the orchestrator should suggest creating one.

## Key Conventions

- Agent folders: `agents/[lowercase-hyphen]/`
- Output files: `YYYY-MM-DD_agent-name_description.md`
- Journal entries: `YYYY-MM-DD_HHMM.md` in `journal/entries/`
- Never overwrite an output file — create a new dated one
- MEMORY.md is the only file an agent updates in-place
- MEMORY.md starts empty — memory is earned from real data, never pre-filled
- Each skill must map to at least one KPI in `AGENT.md`; delete skills that don't

## Templates

- `templates/JOURNAL_ENTRY.md` — format for journal entries
- `templates/WEEKLY_REVIEW.md` — format for weekly review logs
- `templates/TASK_INTAKE.md` — format for human task handoffs
