# Multi-Agent Development Team Plan

## Overview

A 3-tier hierarchy of 10 specialized agents designed for end-to-end software development tasks — from requirements to validated, production-ready code.

---

## Architecture: 3-Tier Hierarchy

```
User Request
     │
     ▼
┌─────────────────────┐
│    ORCHESTRATOR     │  ← Tier 1: Brain
└─────────────────────┘
         │
    ┌────┴─────┬──────────────┐
    ▼          ▼              ▼
 Planning   Engineering   Validation   ← Tier 2: Team Leads
   Lead        Lead          Lead
    │          │              │
  PM, UX   BE, FE Dev      QA, Sec     ← Tier 3: Workers
```

---

## Folder Structure

```
agent-system/
│
├── main.py                     # Entry point
├── orchestrator.py             # Core brain
│
├── runtime/
│   ├── engine.py               # Runs agents
│   ├── router.py               # Routes tasks to teams
│   ├── memory.py               # Loads/saves agent memory
│   └── permissions.py          # Domain enforcement
│
├── config/
│   ├── system.yaml             # Full multi-team config
│   └── models.yaml             # Model routing (cost vs capability)
│
├── prompts/
│   ├── orchestrator.txt
│   ├── planning/
│   │   ├── lead.txt
│   │   ├── pm.txt
│   │   └── ux.txt
│   ├── engineering/
│   │   ├── lead.txt
│   │   ├── backend.txt
│   │   └── frontend.txt
│   └── validation/
│       ├── lead.txt
│       ├── qa.txt
│       └── security.txt
│
├── agents/
│   ├── base_agent.py
│   ├── orchestrator_agent.py
│   ├── lead_agent.py
│   └── worker_agent.py
│
├── tools/
│   ├── file_tools.py           # Read/write files
│   ├── code_tools.py           # Run code/tests
│   └── search_tools.py         # Repo search
│
├── expertise/                  # Persistent agent memory (Markdown)
│   ├── orchestrator.md
│   ├── planning/
│   ├── engineering/
│   └── validation/
│
├── sessions/                   # Chat logs (JSONL)
│   └── session_*.jsonl
│
├── logs/
│   └── runtime.log
│
└── tests/
    └── test_agents.py
```

---

## The 10 Agents

### Tier 1 — Orchestrator

| Agent | Role |
|-------|------|
| **Orchestrator** | Interprets user request, decides which teams to activate, delegates to leads, combines final output |

**Rules:**
- Never does the work itself — only delegates
- Enforces execution order: `planning → engineering → validation`
- Skips unnecessary teams for simple tasks

---

### Tier 2 — Team Leads

| Agent | Role |
|-------|------|
| **Planning Lead** | Breaks task into steps, delegates to PM and UX |
| **Engineering Lead** | Converts plan to technical tasks, delegates to BE/FE |
| **Validation Lead** | Reviews engineering output, delegates to QA and Security |

**Rules:**
- Leads never write code or do worker-level tasks
- Each lead owns the quality of their team's output
- Leads report structured output back to Orchestrator

---

### Tier 3 — Workers

| Agent | Team | Role |
|-------|------|------|
| **Product Manager** | Planning | Defines requirements, acceptance criteria, user intent |
| **UX Researcher** | Planning | Identifies user flows, friction points, usability risks |
| **Backend Developer** | Engineering | APIs, database design, business logic, error handling |
| **Frontend Developer** | Engineering | UI components, responsive design, UX implementation |
| **QA Engineer** | Validation | Test cases, edge cases, bug identification |
| **Security Reviewer** | Validation | Vulnerabilities, input validation, auth, data leaks |

---

## Execution Flow

```
1. User sends task to Orchestrator
2. Orchestrator activates Planning team
   └── Planning Lead → PM + UX → returns structured plan
3. Orchestrator activates Engineering team
   └── Engineering Lead → Backend + Frontend → returns code
4. Orchestrator activates Validation team
   └── Validation Lead → QA + Security → returns review
5. Orchestrator synthesizes all outputs → delivers final result
```

---

## Memory Strategy

Each agent has two memory layers:

| Layer | Location | Purpose |
|-------|----------|---------|
| **Expertise** | `expertise/<team>/<agent>.md` | Long-term knowledge, learned patterns |
| **Session** | `sessions/session_*.jsonl` | Short-term shared context within a task |

---

## Model Routing (`config/models.yaml`)

| Agent Type | Suggested Model | Reason |
|------------|----------------|--------|
| Orchestrator | Claude Sonnet | Needs strong reasoning |
| Team Leads | Claude Sonnet | Delegation logic |
| Workers (code) | Claude Sonnet | Code quality matters |
| Workers (docs/UX) | Claude Haiku | Cost-efficient for text tasks |

---

## Tools Per Agent

| Tool | Used By | Purpose |
|------|---------|---------|
| `file_tools.py` | BE Dev, FE Dev | Read/write source files |
| `code_tools.py` | BE Dev, QA | Execute code, run tests |
| `search_tools.py` | All | Search codebase or docs |

---

## What Makes This Production-Grade

- **Modular** — each agent is independently replaceable
- **Config-driven** — no hardcoding of roles or models
- **Memory-enabled** — agents accumulate expertise over time
- **Observable** — full session logs + runtime logs
- **Safe** — domain permissions prevent agents from overstepping
- **Cost-aware** — cheap models for simple tasks, powerful models where needed

---

## Next Steps

1. Implement `base_agent.py` with shared interface
2. Write all 10 system prompts in `/prompts/`
3. Build `runtime/router.py` delegation logic
4. Wire up `expertise/` memory read/write
5. Test with a real coding assignment end-to-end
