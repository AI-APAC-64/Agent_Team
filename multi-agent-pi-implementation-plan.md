# Multi-Agent Team — Implementation Plan (Built on Pi Coding Agent)

## What Pi Actually Is (Key Insight)

Pi is NOT a multi-agent framework. It is a **single-agent coding harness** that is intentionally minimal:

> "Pi ships with powerful defaults but skips features like sub-agents and plan mode.  
> Instead, you can ask pi to build what you want."

This means your multi-agent system is NOT built inside pi — it is built **on top of pi** using two integration points:

1. **RPC Mode** — spawn and control multiple pi instances as subprocesses
2. **SDK Mode** — embed `createAgentSession()` directly in your orchestrator code

Your Orchestrator becomes the conductor. Each specialized agent = one pi instance.

---

## Architecture: How It Maps to Pi

```
Your Orchestrator (TypeScript)
         │
         │  spawns via RPC / SDK
         ├──────────────────────────────────┐
         ▼                                  ▼
   pi instance                        pi instance
  (Planning Lead)                  (Engineering Lead)
         │                                  │
    spawns workers                    spawns workers
         │                                  │
    ┌────┴────┐                    ┌────────┼────────┐
    ▼         ▼                    ▼        ▼        ▼
  pi (PM)  pi (UX)           pi (BE)  pi (FE)  pi (Arch)
                                       │
                              pi (QA)  pi (Sec)
```

Each pi instance gets:
- Its own **system prompt** (the agent's identity)
- Its own **session** (isolated memory / JSONL log)
- Its own **tools** (controlled via `--tools` flag)
- Its own **skills** (loaded from `.pi/skills/`)

---

## Pi's Key Features You Will Use

| Pi Feature | How You Use It |
|------------|----------------|
| **RPC Mode** (`--mode rpc`) | Orchestrator controls agents via JSON over stdio |
| **SDK Mode** (`createAgentSession`) | Embed agents directly in TypeScript orchestrator |
| **Skills** (`.pi/skills/SKILL.md`) | Each agent's domain expertise loaded automatically |
| **`--system` flag / `.pi/SYSTEM.md`** | Override system prompt per agent instance |
| **`--tools` / `--no-tools`** | Control which tools each agent can use |
| **`--cwd`** | Scope each agent to a specific directory |
| **Session JSONL** (`log.jsonl`) | Full trace per agent — feeds your Langfuse tracing |
| **Compaction** (`/compact`) | Context window management — pi handles it for you |
| **`~/.pi/agent/AGENTS.md`** | Global context injected into every agent session |

---

## Folder Structure

```
digital-worker-hq/
│
├── orchestrator/
│   ├── main.ts                    # Entry point — receives user task
│   ├── router.ts                  # Decides which teams to activate
│   ├── dispatcher.ts              # Spawns pi RPC instances
│   ├── synthesizer.ts             # Combines team outputs into final result
│   └── state.ts                   # Task state machine (pending → done → failed)
│
├── agents/
│   ├── planning/
│   │   ├── lead/
│   │   │   ├── SYSTEM.md          # Planning Lead system prompt
│   │   │   └── .pi/skills/        # Lead-specific skills
│   │   ├── pm/
│   │   │   ├── SYSTEM.md
│   │   │   └── .pi/skills/
│   │   └── ux/
│   │       ├── SYSTEM.md
│   │       └── .pi/skills/
│   │
│   ├── engineering/
│   │   ├── lead/
│   │   │   └── SYSTEM.md
│   │   ├── architect/             # NEW — closes gap from original plan
│   │   │   ├── SYSTEM.md
│   │   │   └── .pi/skills/
│   │   ├── backend/
│   │   │   ├── SYSTEM.md
│   │   │   └── .pi/skills/
│   │   └── frontend/
│   │       ├── SYSTEM.md
│   │       └── .pi/skills/
│   │
│   └── validation/
│       ├── lead/
│       │   └── SYSTEM.md
│       ├── qa/
│       │   ├── SYSTEM.md
│       │   └── .pi/skills/
│       └── security/
│           ├── SYSTEM.md
│           └── .pi/skills/
│
├── skills/                        # Shared skills (all agents inherit)
│   ├── error-handling/
│   │   └── SKILL.md
│   ├── tdd/
│   │   └── SKILL.md
│   └── code-review/
│       └── SKILL.md
│
├── config/
│   ├── agent-registry.yaml        # Which agents exist, their tools, model
│   ├── tool-permissions.yaml      # Which agent can use which tools
│   └── model-routing.yaml         # Sonnet vs Haiku per agent type
│
├── protocols/
│   ├── message-schema.ts          # Typed message format between agents
│   ├── task-schema.ts             # Task object with status + history
│   └── handoff-schema.ts          # Structured output format per team
│
├── observability/
│   ├── langfuse-bridge.ts         # Maps pi JSONL sessions → Langfuse traces
│   └── session-collector.ts       # Aggregates all agent sessions per task
│
├── sessions/                      # Pi session files, one folder per run
│   └── task-{id}/
│       ├── planning-lead.jsonl
│       ├── pm.jsonl
│       ├── backend.jsonl
│       └── ...
│
└── AGENTS.md                      # Global context for ALL pi instances
```

---

## Communication Protocol

This is the most critical gap from the original plan. Here is the contract.

### Task Object (passed agent to agent)

```typescript
interface Task {
  id: string;
  status: "pending" | "in_progress" | "awaiting_review" | "done" | "failed";
  user_request: string;
  history: AgentOutput[];       // Accumulates as teams work
  context: {
    codebase_path: string;
    tech_stack: string[];
    constraints: string[];
  };
  checkpoints: Checkpoint[];    // Human approval gates
}
```

### Agent Output (returned by each agent)

```typescript
interface AgentOutput {
  agent: string;                // "planning-lead", "backend-dev", etc.
  team: "planning" | "engineering" | "validation";
  status: "success" | "needs_revision" | "blocked" | "failed";
  content: string;              // The agent's actual output
  files_modified: string[];     // Tracked from pi session log
  issues: Issue[];              // Bugs, risks, security flags
  confidence: number;           // 0–1, agent self-assessment
  session_log: string;          // Path to .jsonl for Langfuse
}
```

### RPC Message Format (orchestrator ↔ pi instance)

```jsonl
// Send task to a pi agent
{"id": "task-001", "type": "prompt", "message": "<task as structured text>"}

// Pi streams back events
{"type": "message_update", "assistantMessageEvent": {"type": "text_delta", "delta": "..."}}
{"type": "tool_call", "tool": "bash", "input": {"command": "npm test"}}
{"type": "agent_end", "response": "...final output..."}
```

---

## Execution Flow (With State + Human Checkpoints)

```
1. User submits task
   └── Orchestrator creates Task object (status: pending)

2. PLANNING PHASE (status: in_progress)
   ├── Spawn: planning-lead pi instance
   │    └── Lead spawns: pm + ux instances in parallel
   │    └── Lead synthesizes → returns PlanOutput
   └── CHECKPOINT ← Show plan to user → approve / revise before engineering starts

3. ENGINEERING PHASE (status: in_progress)
   ├── Spawn: architect pi instance → returns API contracts + data models
   ├── Spawn: backend-dev pi instance (uses architect output as context)
   ├── Spawn: frontend-dev pi instance (uses architect output as context)
   └── Engineering Lead synthesizes → returns CodeOutput

4. VALIDATION PHASE (status: awaiting_review)
   ├── Spawn: qa pi instance → returns test results + issue list
   ├── Spawn: security pi instance → returns vulnerability report
   └── Validation Lead synthesizes → returns ValidationOutput

5. DECISION GATE
   ├── If issues found → loop back to Engineering (with issues as context)
   ├── If approved → status: done → return final result to user
   └── If critical failure → status: failed → escalate to user
```

---

## Pi Skills Per Agent

Skills are Markdown files that pi auto-loads. Each agent gets skills matching its role.

### Backend Developer — `.pi/skills/`

```
backend-dev/
├── error-handling/SKILL.md     # Always wrap with try/catch, typed errors
├── api-design/SKILL.md         # REST conventions, status codes
├── database/SKILL.md           # Query patterns, indexing
└── security-basics/SKILL.md    # Input validation, no secrets in code
```

### QA Engineer — `.pi/skills/`

```
qa-agent/
├── test-writing/SKILL.md       # Unit, integration, e2e structure
├── edge-cases/SKILL.md         # Boundary conditions, null checks
└── test-coverage/SKILL.md      # What must be covered before done
```

### Security Reviewer — `.pi/skills/`

```
security-agent/
├── owasp-top10/SKILL.md        # Common vulnerabilities checklist
├── auth-review/SKILL.md        # JWT, session, OAuth review
└── data-exposure/SKILL.md      # PII, logging, response sanitization
```

---

## Tool Permissions Per Agent

Controlled via `--tools` and `--no-tools` when spawning pi instances.

| Agent | Allowed Tools | Blocked Tools |
|-------|--------------|---------------|
| Planning Lead | read | write, bash |
| PM | read | write, bash |
| UX | read | write, bash |
| Architect | read, write | bash |
| Backend Dev | read, write, edit, bash | — |
| Frontend Dev | read, write, edit, bash | — |
| QA | read, bash | write, edit |
| Security | read | write, edit, bash |
| All Leads | read | write, bash |

---

## Observability: Langfuse Bridge

Pi writes a complete `log.jsonl` per session. Bridge it to Langfuse:

```
pi session log (JSONL)
        │
        ▼
langfuse-bridge.ts
  ├── Creates Langfuse trace per task
  ├── Creates span per agent (from session start/end timestamps)
  ├── Attaches tool_calls as observations
  ├── Records token usage from pi cost tracking
  └── Tags with: agent name, team, task ID, model used
```

This gives you a full end-to-end trace across all agents in one Langfuse task view — directly compatible with your existing OTel + Langfuse stack.

---

## Model Routing

```yaml
# config/model-routing.yaml

orchestrator:   claude-sonnet-4-5   # complex reasoning
leads:          claude-sonnet-4-5   # delegation logic
architect:      claude-sonnet-4-5   # system design quality matters
backend:        claude-sonnet-4-5   # code quality critical
frontend:       claude-sonnet-4-5   # code quality critical
qa:             claude-haiku-4-5    # structured output, cost-efficient
security:       claude-sonnet-4-5   # cannot afford misses here
pm:             claude-haiku-4-5    # text-heavy, cost-efficient
ux:             claude-haiku-4-5    # text-heavy, cost-efficient
```

---

## Error Handling & Fallback Strategy

| Failure Type | Strategy |
|-------------|----------|
| Agent returns empty output | Retry once with same context |
| Agent confidence < 0.5 | Flag to Lead, request revision |
| Agent fails 2x | Escalate to human checkpoint |
| Context window exceeded | Pi handles via compaction automatically |
| Validation finds critical bugs | Re-route to Engineering with issue list |
| Total task failure | Save full session state, surface to user with partial output |

---

## Definition of Done

Validation Lead approves output when ALL of:

- [ ] QA: 0 critical bugs, all test cases pass
- [ ] Security: 0 high/critical vulnerabilities
- [ ] Backend: error handling present, no raw exceptions exposed
- [ ] Frontend: no broken UI states, responsive
- [ ] Architect: API contracts honoured by implementation
- [ ] Code: no TODO/FIXME left in delivered files

---

## Implementation Phases

### Phase 1 — Foundation (Week 1)
- Install pi: `npm install -g @mariozechner/pi-coding-agent`
- Write `SYSTEM.md` for all 10 agents
- Write core skills: `error-handling`, `tdd`, `api-design`
- Build `dispatcher.ts` — spawn and communicate with pi via RPC
- Test single agent round-trip end to end

### Phase 2 — Team Structure (Week 2)
- Build `state.ts` task state machine
- Implement `protocols/message-schema.ts`
- Wire Planning team: Lead → PM + UX → synthesize
- Add human checkpoint after planning phase
- Test with a real simple coding assignment

### Phase 3 — Full Pipeline (Week 3)
- Wire Engineering team: Lead → Architect → BE + FE
- Wire Validation team: Lead → QA + Security
- Implement error handling + retry logic
- Add validation → engineering feedback loop

### Phase 4 — Observability (Week 4)
- Build `langfuse-bridge.ts`
- Map pi sessions → Langfuse spans
- Add cost tracking per agent per task
- Add task-level dashboards in SigNoz

### Phase 5 — Hardening (Week 5)
- Run 10 real coding assignments end to end
- Tune system prompts based on output quality
- Tune model routing based on cost vs quality data
- Write `AGENTS.md` global context from learnings

---

## Key Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Integration mode | RPC + SDK mixed | RPC for isolation, SDK for tight TypeScript integration |
| Language | TypeScript | Matches pi's native SDK |
| Sub-agent communication | Structured JSONL over RPC | Pi's native protocol — no new dependency |
| Memory | Pi session JSONL + expertise `.md` files | Pi handles compaction natively |
| Context window management | Delegated to Pi | Pi's compaction is battle-tested |
| Observability | Pi sessions → Langfuse bridge | Reuses your existing OTel + Langfuse stack |
| Human-in-the-loop | After planning phase | Prevents wasted engineering work on wrong plan |
