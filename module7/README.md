# Module 7 — Multi-Agent Coordination and Implementation Strategy

## What You Will Build

A multi-agent orchestration system with two specialist agents running in parallel, a conflict-detection step, and a synthesis agent that produces a unified action plan:

- **`orchestrator.py`** — coordinates two specialist agents (Gate Agent + Rollback Agent) running in parallel threads, detects conflicts between their outputs, and calls a synthesis step to resolve them.
- **`interpret.py`** — reads the orchestrator's JSON output and uses Claude to convert it into a prioritised task list and a Slack-ready escalation memo.

---

## Files

| File | Purpose |
|------|---------|
| `orchestrator.py` | **Exercise file** — implement the three TODO functions |
| `interpret.py` | **Secondary script** — converts orchestrator output to human-readable format |
| `agent.py` | Simplified single-agent entry point |
| `sample_data.json` | Platform incident event (P1 — payment service errors, auth OOMKill) |
| `agent-config.yml` | Model and output schema |
| `solutions/solution.py` | **Reference implementation** — read this only after your own attempt |

---

## Setup

```bash
# From the repo root
export ANTHROPIC_API_KEY=your_key_here
python module1/verify_setup.py
```

---

## Run

```bash
# Mock mode — no API key needed:
python module7/orchestrator.py --mock

# Specific conflict scenario:
python module7/orchestrator.py --mock --scenario full_conflict
python module7/orchestrator.py --mock --scenario partial_conflict
python module7/orchestrator.py --mock --scenario no_conflict

# Live run against Claude (parallel specialist agents):
ANTHROPIC_API_KEY=sk-... python module7/orchestrator.py --scenario full_conflict

# Interpret the orchestrator's output (run after orchestrator):
python module7/interpret.py --mock
ANTHROPIC_API_KEY=sk-... python module7/interpret.py
```

---

## Architecture

```
Platform Event
     │
     ▼
 Orchestrator
     │
     ├──────────────────┐
     ▼                  ▼
 Gate Agent       Rollback Agent
 (parallel)        (parallel)
     │                  │
     └──────────────────┘
             │
             ▼
      Conflict Check
             │
      ┌──────┴──────┐
      ▼             ▼
  No Conflict    Conflict
  → SYNTHESISE → ESCALATE
```

All messages flow through the orchestrator. Specialists never communicate with each other directly.

---

## Conflict Scenarios (from sample_data.json)

| Scenario | Gate Agent | Rollback Agent | Outcome |
|----------|-----------|----------------|---------|
| `no_conflict` | APPROVE | No rollback | SYNTHESISE — safe to deploy |
| `partial_conflict` | APPROVE_WITH_CONDITIONS | SCHEDULED rollback | SOFT_ESCALATE — inform on-call |
| `full_conflict` | APPROVE | IMMEDIATE rollback | SAFETY_FIRST_ESCALATE — halt all deploys |

---

## Expected Output (`full_conflict` scenario)

```json
{
  "gate_agent": {
    "decision": "APPROVE",
    "confidence": "HIGH",
    "rationale": "CI pipeline passed all gates before deploy. Snapshot taken 45 minutes ago.",
    "blocking_issues": [],
    "conditions": [],
    "risk_score": "LOW",
    "escalate": false
  },
  "rollback_agent": {
    "rollback_recommended": true,
    "severity": "IMMEDIATE",
    "trigger": "latency_p95_delta exceeded 18.4% and error_rate_pct > 10% post-deploy.",
    "rollback_target": "v1.8.2",
    "escalate": false
  },
  "conflict": {
    "detected": true,
    "type": "HARD_CONFLICT",
    "resolution": "SAFETY_FIRST_ESCALATE",
    "summary": "Gate Agent: APPROVE (stale pre-deploy snapshot). Rollback Agent: IMMEDIATE rollback (live post-deploy data). Hard conflict — Safety First: escalate and halt all deploys until human reviews."
  }
}
```

Full result saved to `output/orchestrator_module7.json`.

---

## Exercise

**Part A — Orchestrator:** Open `orchestrator.py`. Implement the three TODO functions:

1. **`run_gate_agent(context)`** — call `ask()` with `GATE_SYSTEM_PROMPT` to evaluate quality gates
2. **`run_rollback_agent(context)`** — call `ask()` with `ROLLBACK_SYSTEM_PROMPT` to assess rollback need
3. **`detect_conflict(gate_result, rollback_result)`** — compare both outputs and return a conflict dict (HARD_CONFLICT / SOFT_CONFLICT / no conflict)

The specialists are already wired to run in parallel via `ThreadPoolExecutor` — you do not need to change `main()`.

```bash
python module7/orchestrator.py --mock --scenario full_conflict    # shows expected output
ANTHROPIC_API_KEY=sk-... python module7/orchestrator.py --scenario full_conflict
```

**Part B — Interpret:** Run `interpret.py` after the orchestrator to see the output converted into a human-readable escalation memo.

If you get stuck, see `solutions/solution.py`.

---

## Key Takeaway

**Safety First** is the non-negotiable conflict resolution rule. When the Gate Agent says APPROVE (based on a pre-deploy snapshot) and the Rollback Agent says IMMEDIATE rollback (based on live post-deploy signals), the Rollback Agent wins — always. Deployment approval is reversible; a live incident is not. The conflict detector's job is not to find a compromise: it is to implement the Safety First rule and escalate. Notice that `full_conflict` produces `resolution: SAFETY_FIRST_ESCALATE` — because the orchestrator cannot verify live infrastructure state in real time, and escalating to a human is the only safe answer when agents contradict each other on a live system.

---

## GitHub Actions

**Workflow file:** `.github/workflows/module7-multi-agent.yml`

| Property | Value |
|----------|-------|
| Workflow name | `Module 7 — Multi-Agent` |
| Trigger | Push to `module7/**` or `shared/**`, or manual via Actions tab |
| Script run | `python module7/agent.py` |
| Output artifact | `module7-output` → `output/output_module7.json` |

The workflow runs `agent.py` — the simplified single-agent entry point. In CI, `orchestrator.py` is not run directly because it requires a `--scenario` flag to produce deterministic output. Run `orchestrator.py` locally with `--mock` or a specific `--scenario` to test the full multi-agent orchestration with parallel specialists and conflict detection.

**Prerequisite:** Add your API key as a repository secret named `ANTHROPIC_API_KEY` (Settings → Secrets and variables → Actions → New repository secret).

---

## Success Criteria

- `orchestrator.py --mock` runs cleanly and shows all three scenarios
- Both specialist agents run and return valid JSON
- Conflict detection correctly identifies overlapping targets
- Synthesis agent produces `unified_actions` and `final_decision`
- `interpret.py` produces a readable task list and escalation memo
- Full output saved to `output/orchestrator_module7.json`
- If `escalate=true`, an escalation notice is printed
- GitHub Actions workflow completes and `module7-output` artifact is attached to the run
- If stuck, see `solutions/solution.py`
