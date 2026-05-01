# Module 6 — Operational Intelligence and Conversational Observability

## What You Will Build

A two-phase conversational ops agent that can answer natural-language questions about platform health:

1. **`observability_mock.py`** — a local HTTP server that simulates a real observability platform (think Datadog/Prometheus). Serves realistic platform health data across four endpoints.
2. **`conversational_agent.py`** — a two-phase agent: Phase 1 routes the question (incident / investigation / health check), Phase 2 fetches live data from the mock server and produces a structured diagnosis with a causal chain.

---

## Files

| File | Purpose |
|------|---------|
| `observability_mock.py` | **Local mock observability server** — start this first |
| `conversational_agent.py` | **Exercise file** — implement `phase1_route()` and `phase2_analyse()` |
| `agent.py` | Simplified REPL alternative |
| `sample_data.json` | Static snapshot of platform health data |
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

## Run (Two-Terminal Workflow)

**Terminal 1 — start the mock server:**

```bash
# Normal scenario (all services healthy):
python module6/observability_mock.py

# Active incident scenario (OOMKill in checkout-service):
python module6/observability_mock.py --scenario incident

# High-load scenario:
python module6/observability_mock.py --scenario high-load
```

**Terminal 2 — run the conversational agent:**

```bash
# Mock mode (no server or API key needed):
python module6/conversational_agent.py --query "What's wrong?" --mock

# Live call against the mock server:
ANTHROPIC_API_KEY=sk-... python module6/conversational_agent.py --query "We are getting paged. What is causing the latency spike?"

# Interactive REPL (keeps asking questions):
ANTHROPIC_API_KEY=sk-... python module6/agent.py
```

---

## Mock Server Endpoints

| Endpoint | Returns |
|----------|---------|
| `GET /health` | Per-service health status (UP / DEGRADED / DOWN) |
| `GET /metrics` | Error rates, latency, throughput, memory usage |
| `GET /anomalies` | Detected anomalies with severity and correlation chain |
| `GET /events` | Recent deployment and config-change events |

---

## Expected Output

```json
{
  "answer": "The checkout-service is experiencing elevated latency (p95 = 2.3s vs 0.4s baseline) triggered by a payment-api 40% error rate starting 8 minutes after the last deployment.",
  "causal_chain": [
    "checkout-service deployment completed at 14:24 UTC",
    "payment-api error rate began rising at 14:32 UTC (8-minute lag)",
    "checkout-service latency spiked as retries accumulated against failing payment-api"
  ],
  "confidence": "MEDIUM",
  "follow_up_questions": [
    "Was there a simultaneous change to payment-api at 14:32 UTC?",
    "What does the payment-api error response body contain?"
  ],
  "escalate": true
}
```

Full result saved to `output/output_module6.json`.

---

## Exercise

Open `conversational_agent.py`. There are two functions to implement:

**`phase1_route(query)`** — call `ask()` with `ROUTING_SYSTEM_PROMPT` to classify the query as `incident`, `investigation`, or `health_check`. Return `result.get("query_type", "health_check")`.

**`phase2_analyse(query, query_type, platform_data)`** — build a `user_msg` combining the query, type, and platform data snapshot, then call `ask()` with `ANALYSIS_SYSTEM_PROMPT` and return the result dict.

```bash
python module6/conversational_agent.py --query "What's wrong?" --mock     # shows expected output
ANTHROPIC_API_KEY=sk-... python module6/conversational_agent.py --query "We are getting paged. What is causing the latency spike?"
```

Both system prompts are already written — study them before implementing the `ask()` calls. If you get stuck, see `solutions/solution.py`.

---

## Troubleshooting

**`ConnectionRefusedError: [Errno 111]`** — The mock server isn't running. Start `observability_mock.py` in a separate terminal first.

**`ModuleNotFoundError: No module named 'observability_mock'`** — Run from the repo root: `python module6/conversational_agent.py`, not `cd module6 && python conversational_agent.py`.

---

## Key Takeaway

Phase 1 routing is cheap on purpose. Classifying the question first — before fetching any data — means you only pull the observability signals that are actually relevant to the question type. An incident query needs anomalies and events; a health check only needs `/health`. Without routing, the agent would over-fetch on every query and send irrelevant data to Claude, increasing token cost and degrading reasoning quality. The two-phase pattern scales: adding a new question type means adding a new route and a new data-fetch strategy — not rewriting the agent. This is the same pattern used in production AIOps systems.

---

## GitHub Actions

**Workflow file:** `.github/workflows/module6-conv-ops.yml`

| Property | Value |
|----------|-------|
| Workflow name | `Module 6 — Conversational Ops` |
| Trigger | Push to `module6/**` or `shared/**`, or manual via Actions tab |
| Script run | `python module6/agent.py` |
| Output artifact | `module6-output` → `output/output_module6.json` |

The workflow runs `agent.py` (the simplified REPL alternative) rather than `conversational_agent.py`. In CI there is no interactive terminal, so the REPL script runs a single preconfigured query against `sample_data.json` — no live mock server is needed.

When you run `conversational_agent.py` locally with the two-terminal setup, you are adding the live mock server to the loop. The GitHub Actions run shows you the same structured JSON output but using the static snapshot, which is reproducible and does not depend on a running server.

**Prerequisite:** Add your API key as a repository secret named `ANTHROPIC_API_KEY` (Settings → Secrets and variables → Actions → New repository secret).

---

## Success Criteria

- Mock server starts and responds to all four endpoints
- `conversational_agent.py --mock` runs cleanly without a server or API key
- Agent correctly routes the query (incident / investigation / health_check)
- Response contains `answer`, `causal_chain` (list), `confidence`, and `escalate`
- Full output saved to `output/output_module6.json`
- GitHub Actions workflow completes and `module6-output` artifact is attached to the run
- If stuck, see `solutions/solution.py`
