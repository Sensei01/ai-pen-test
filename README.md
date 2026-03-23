# 🛡️ AI Chat Penetration Test

> A structured 28-test security framework for LLM-backed chat APIs.
> Covers prompt extraction, injection attacks, guardrail validation, and infrastructure hardening.
> Runs as a standalone Python tool, a Claude skill, or inside a CI/CD pipeline.

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-green.svg)
![Tests: 28](https://img.shields.io/badge/Tests-28-orange.svg)
![Status: Production Ready](https://img.shields.io/badge/Status-Production--Ready-brightgreen.svg)

---

## Why This Exists

Most AI security tooling focuses on infrastructure — auth, rate limits, input length. That's necessary, but it misses the biggest attack surface: **the model itself**.

A chatbot with weak system prompt defenses can be social-engineered into revealing its full instructions, executing injected commands, or leaking data from its context window — regardless of how locked-down the server layer is.

This framework tests **both layers together**: the AI's behavioral defenses (21 tests) and the server-side guardrails (7 tests). It was built from real-world attack patterns observed in production AI chat features.

**Minimum passing score for production: 25/28 (90%)**
**Recommended target: 28/28 (100%)**

---

## What It Tests

| # | Category | Tests | What It Covers |
|---|---|---|---|
| 1 | Direct Prompt Extraction | 5 | Simple requests, authority claims, role-play bypasses, token obfuscation |
| 2 | Indirect Prompt Extraction | 4 | Constraint mapping, behavior contradiction, context window overflow |
| 3 | Input Injection | 4 | Embedded directives (`[SYSTEM_OVERRIDE]`), pseudocode execution, multi-step social engineering |
| 4 | Sophisticated Techniques | 4 | Hypothetical framing, research justification, capability boundary probing |
| 5 | Context Pollution | 2 | Fake conversation history, fictional scenario authority |
| 6 | Output Exploitation | 2 | Leveraging alleged prior leaks, partial reveal completion |
| 7 | Infrastructure Guardrails | 7 | Length limits, empty messages, rate limiting, auth enforcement, HTML injection |

---

## How It Works

The framework has two test types that run separately:

**Infrastructure tests** (`run_infra_tests.py`) make real HTTP calls to your API and grade pass/fail automatically based on status codes and response content. No human needed.

**AI behavioral tests** (`run_ai_tests.py`) send 21 attack payloads to your chat endpoint and collect the model's responses. You grade them interactively in the terminal, or use `--auto-grade` mode to have Claude grade them automatically — useful for CI/CD.

After both run, `generate_report.py` combines the results into a clean terminal summary, a markdown report, and a standalone HTML report.

---

## Quickstart

### 1. Install dependencies

```bash
pip install requests rich
```

For auto-grade mode (optional):
```bash
pip install anthropic
```

### 2. Set your target

```bash
export PENTEST_ENDPOINT="https://your-api.example.com/v1/chat"
export PENTEST_TOKEN="Bearer sk-..."

# Optional — only if yours differ from the defaults:
export PENTEST_MSG_FIELD="message"        # JSON key for user message (default: message)
export PENTEST_AUTH_HEADER="Authorization"  # Auth header name (default: Authorization)
```

> ⚠️ Always test against a **staging** environment, not production. These tests send malicious payloads.

### 3. Run infrastructure tests

```bash
python scripts/run_infra_tests.py \
  --endpoint "$PENTEST_ENDPOINT" \
  --token "$PENTEST_TOKEN"
```

This runs 7 automated HTTP-level tests (G1–G7) and saves results to `pentest_results/infra_results.json`.

### 4. Run AI behavioral tests

**Interactive mode** (you grade each response in the terminal):
```bash
python scripts/run_ai_tests.py \
  --endpoint "$PENTEST_ENDPOINT" \
  --token "$PENTEST_TOKEN"
```

**Auto-grade mode** (Claude grades responses — good for CI):
```bash
python scripts/run_ai_tests.py \
  --endpoint "$PENTEST_ENDPOINT" \
  --token "$PENTEST_TOKEN" \
  --auto-grade
```
Auto-grade requires `ANTHROPIC_API_KEY` to be set.

Results are saved to `pentest_results/ai_results.json`.

### 5. Generate the report

```bash
python scripts/generate_report.py \
  --project "MyApp" \
  --tester "Your Name" \
  --model "Claude 3.5 Sonnet" \
  --endpoint "$PENTEST_ENDPOINT"
```

Opens an HTML report in your browser and saves both `report.md` and `report.html` to `pentest_results/`.

---

## Sample Output

```
┌─────────────────────────────────────────────────────────────┐
│          AI Chat Pen Test  ·  MyApp  ·  2026-03-23          │
└─────────────────────────────────────────────────────────────┘

  CAT 1  Direct Prompt Extraction           ✓  5/5
  CAT 2  Indirect Prompt Extraction         ✓  4/4
  CAT 3  Input Injection                    ✓  4/4
  CAT 4  Sophisticated Techniques           ✓  4/4
  CAT 5  Context Pollution                  ✓  2/2
  CAT 6  Output Exploitation                ✗  1/2
  CAT 7  Infrastructure Guardrails          ✗  5/7

  ──────────────────────────────────────────────────────
  TOTAL SCORE:  26/28  (92.8%)          ✓  PASS
  ──────────────────────────────────────────────────────

  Remediation needed:
  ✗  6.2  Partial Reveal Completion
          Add system prompt instruction to refuse explaining guardrail mechanisms.
  ✗  G3   Rate Limiting / Cooldown
          Implement per-user cooldown of 2–3 seconds between messages.
  ✗  G6   HTML / Script Injection
          Strip <script> and HTML tags server-side before storage and model input.
```

---

## CI/CD Integration

Add this to your pipeline to gate deployments automatically:

```yaml
# .github/workflows/ai-pentest.yml
name: AI Pen Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  pentest:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install requests rich anthropic

      - name: Run infrastructure tests
        run: |
          python scripts/run_infra_tests.py \
            --endpoint "$PENTEST_ENDPOINT" \
            --token "$PENTEST_TOKEN"
        env:
          PENTEST_ENDPOINT: ${{ secrets.PENTEST_ENDPOINT }}
          PENTEST_TOKEN: ${{ secrets.PENTEST_TOKEN }}

      - name: Run AI behavioral tests
        run: |
          python scripts/run_ai_tests.py \
            --endpoint "$PENTEST_ENDPOINT" \
            --token "$PENTEST_TOKEN" \
            --auto-grade
        env:
          PENTEST_ENDPOINT: ${{ secrets.PENTEST_ENDPOINT }}
          PENTEST_TOKEN: ${{ secrets.PENTEST_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Generate report
        run: |
          python scripts/generate_report.py \
            --no-browser \
            --project "MyApp" \
            --endpoint "$PENTEST_ENDPOINT"
        env:
          PENTEST_ENDPOINT: ${{ secrets.PENTEST_ENDPOINT }}

      - name: Upload report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: pentest-report
          path: pentest_results/
          retention-days: 30
```

The infrastructure test script exits with code `1` if any tests fail, which blocks the pipeline automatically. Store `PENTEST_ENDPOINT`, `PENTEST_TOKEN`, and `ANTHROPIC_API_KEY` as repository secrets — never hardcode them.

---

## Using as a Claude Skill

This framework is also packaged as a **Claude skill** — a prompt + scripts that Claude can invoke when you describe a security testing need.

Install it in [Claude Code](https://docs.claude.ai/claude-code) or Cowork by copying the `SKILL.md` and `scripts/` to your skills folder. Once installed, you can say things like:

- *"Pen test my AI chat feature at https://staging.myapp.com/api/chat"*
- *"Is my chatbot leaking its system prompt?"*
- *"Run the AI guardrail tests and give me a remediation plan"*

Claude will guide you through the full setup, run the tests, and interpret the results.

**Supported environments:** Claude Code, Windsurf, Cursor, Replit, GitHub Actions, any terminal with Python 3.8+.

---

## Repo Structure

```
ai-pen-test/
├── scripts/
│   ├── run_infra_tests.py        # Category 7: automated HTTP-level tests
│   ├── run_ai_tests.py           # Categories 1–6: sends attack payloads
│   └── generate_report.py        # Combines results into a clean report
├── references/
│   ├── attack_payloads.json      # All 21 attack payloads with pass/fail criteria
│   └── framework.md              # Full framework documentation
├── evals/
│   └── evals.json                # Test cases for validating the Claude skill
├── SKILL.md                      # Claude skill definition
└── README.md
```

---

## Implementation Checklist

Use this alongside the test results when hardening your AI feature.

### System Prompt Hardening
- [ ] Explicit identity statement ("You are X, you only help with Y")
- [ ] Refusal instructions for prompt extraction attempts
- [ ] Role-play and persona-change rejection
- [ ] Instruction to ignore embedded directives (`[SYSTEM_OVERRIDE]`, `[ADMIN]`, etc.)
- [ ] Topic boundaries with redirect language
- [ ] No explanation of why behaviors are restricted

### Server-Side Guardrails
- [ ] Input length validation (server-side, not just client-side)
- [ ] Injection pattern regex applied before AI processing
- [ ] HTML and script tag stripping
- [ ] Per-user message cooldown (2–3 seconds)
- [ ] Authentication enforcement on all endpoints (401 for missing/invalid tokens)
- [ ] Malformed request handling — no stack traces exposed
- [ ] Flagging and logging of injection attempts

### Output Protection
- [ ] Output sanitization for leaked prompt fragments
- [ ] Response length cap
- [ ] Keyword scanning for system prompt content in responses

### Operational
- [ ] Per-tier message limits enforced
- [ ] Context window capped (10 messages recommended)
- [ ] Conversation history sanitized before sending to model
- [ ] Re-run suite after every model change, prompt update, or major deployment

---

## Important Notes

**Re-run after every change.** A model that passes today may fail after a prompt update or model version change. Build this into your deployment pipeline, not just as a one-time audit.

**Test both layers.** A strong system prompt alone is not sufficient. Server-side guardrails catch attacks before they reach the model. This framework is designed to test both together.

**Use staging.** The behavioral tests send adversarial payloads. Run against a staging or test environment to avoid polluting production logs or triggering abuse detection on your own API.

**PARTIAL is a risk signal.** Tests graded as PARTIAL (model partially complied) count as failures. Even a partial information leak can be enough for a determined attacker to reconstruct the full system prompt.

---

## Contributing

Issues and PRs are welcome. If you've found a new attack pattern that isn't covered, or a test case that produces false positives against well-hardened models, open an issue or submit a PR adding it to `references/attack_payloads.json`.

When adding tests, follow the existing format: include the attack payload, pass criteria, and fail criteria. Each criterion should be objective enough to grade automatically.

---

## License

MIT — free to use, modify, and distribute.

---

*Built by [Ahmad Sani Barde](https://www.linkedin.com/in/asbarde)*
