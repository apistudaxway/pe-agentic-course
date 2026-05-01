# Module 2 — Agentic AI Fundamentals: How Agents Reason and Act

## What You Will Build

**Part A (Exercise — no API key needed):** Use Claude.ai in the browser to experience the before/after of structured prompting. See how an unstructured prompt gets you prose; a structured prompt with a JSON schema gets you a parseable, predictable response.

**Part B (Demo — Python):** The same structured prompt moved into Python code. `triage_agent.py` wraps exactly what you did in Claude.ai as a five-step agentic loop that can run in GitHub Actions.

---

## Files

| File | Purpose |
|------|---------|
| `triage_agent.py` | **Exercise file** — implement `SYSTEM_PROMPT` and `run_agent()` |
| `agent.py` | Alternative exercise entry point (same pattern, simpler structure) |
| `sample_log.txt` | Sample CI failure log (agent input) |
| `agent-config.yml` | Model and output schema |
| `solutions/solution.py` | **Reference implementation** — read this only after your own attempt |

---

## Part A — Browser Exercise (Claude.ai)

No setup needed. Go to [claude.ai](https://claude.ai).

**Round 1 — no system prompt:**

Paste this request with no system prompt:
> *"Our Node.js build has been flaky for 3 days. Memory usage spikes every 2–3 builds. Help me fix this."*

Observe: Claude responds in prose, length varies, structure varies, nothing is reliably parseable.

**Round 2 — with a structured system prompt:**

Write a system prompt that specifies:
- Claude's role (platform engineering assistant)
- Required JSON output keys: `diagnosis`, `confidence` (HIGH/MEDIUM/LOW), `recommended_action`, `escalate` (boolean)
- Rule: confidence is HIGH only when root cause is confirmed in logs; MEDIUM when inferring state

Re-run the same request. Compare the two outputs.

**What to observe:** Same model, same question, completely different output. The prompt is the program.

---

## Part B — Python Agent (triage_agent.py)

```bash
# See expected output without an API key:
python module2/triage_agent.py --mock
```

**Expected output (mock mode):**
```json
{
  "diagnosis": "The deployment failed due to a missing environment variable PAYMENT_API_KEY in the production environment.",
  "confidence": "HIGH",
  "recommended_action": "Add PAYMENT_API_KEY to GitHub Actions secrets and reference it in the workflow env block.",
  "escalate": false
}
```

```bash
# Live call against Claude:
ANTHROPIC_API_KEY=sk-... python module2/triage_agent.py
```

The script implements the five-step agentic loop:
1. `step1_write_prompt()` — build the system prompt and user message
2. `step2_call_api()` — call `ask()` and return the dict
3. `step3_parse_json()` — validate required keys are present
4. `step4_execute_action()` — print `recommended_action`; if `escalate=true`, print escalation notice
5. `step5_verify_result()` — return True if output meets success criteria

**Key Takeaway:** The five steps make testing trivial — you can unit-test `step3_parse_json()` independently of the API call, and mock `step2_call_api()` without touching the prompt logic.

---

## Exercise

Open `triage_agent.py`. There are two things to implement:

1. **`SYSTEM_PROMPT`** — write a system prompt that tells Claude its role and specifies the JSON output schema (same schema you used in Part A). Use the `MOCK_RESPONSE` at the top of the file as a guide to the expected output shape.

2. **`run_agent()`** — implement the `ask()` call that sends `SYSTEM_PROMPT` and the log content to Claude and returns the result dict.

Run `--mock` first to see the expected output, then implement:

```bash
python module2/triage_agent.py --mock     # shows expected output shape
ANTHROPIC_API_KEY=sk-... python module2/triage_agent.py   # your live implementation
```

If you get stuck, check `solutions/solution.py` for the reference implementation.

---

## GitHub Actions

**Workflow file:** `.github/workflows/module2-first-agent.yml`

| Property | Value |
|----------|-------|
| Workflow name | `Module 2 — First AI Agent` |
| Trigger | Push to `module2/**` or `shared/**`, or manual via Actions tab |
| Script run | `python module2/agent.py` |
| Output artifact | `module2-output` → `output/output_module2.json` |

The workflow runs automatically when you push any change inside `module2/` or `shared/`. You can also trigger it manually: Actions tab → "Module 2 — First AI Agent" → Run workflow.

This is the first module where you can see your agent running in a real CI environment. The output artifact lets you compare what Claude produced on GitHub's runners against your local run — same prompt, same input, same structured JSON.

**Prerequisite:** Add your API key as a repository secret named `ANTHROPIC_API_KEY` (Settings → Secrets and variables → Actions → New repository secret).

---

## Success Criteria

- `triage_agent.py --mock` runs cleanly and prints valid JSON
- Live run returns all four keys: `diagnosis`, `confidence`, `recommended_action`, `escalate`
- `confidence` is `HIGH` for the `sample_log.txt` OOM scenario (the log is unambiguous)
- `escalate` is `false` — agent has a concrete fix, no human needed
- `output/output_module2.json` is written
- GitHub Actions workflow completes and `module2-output` artifact is attached to the run
