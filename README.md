# Service-Desk AI Tools — CPU-Only, Local-First, Safety-Layered

Two production-style AI utilities I built for a service-desk workflow, engineered
to run **entirely on a CPU-only Linux box** (no GPU, no cloud, no API keys) in
**under 15 seconds per request** — with a measured **zero-dangerous-miss** safety
record.

> **Note on source:** these are work projects, so the source code is private.
> This repo documents the architecture, the engineering decisions, and the
> measured results.

---

## The two tools

| Tool | What it does |
|---|---|
| **Service Request Cleanup** | Turns a messy support note (*"cust says cant login been trying since morning very upset"*) into a clean, structured, schema-validated JSON record: issue type, impact, missing info, next step, ML tags. Never invents facts. |
| **Translation Quality Checker** | Judges whether an English→Spanish customer reply is accurate and safe to send. Returns a verdict — `Send` / `Review First` / `Do Not Send` — plus risky phrases, back-translation, and a suggested correction. |

Both are served from a single local web UI (Python stdlib `http.server` — zero
web-framework dependencies) and as CLIs, and deploy to a fresh Linux box with
**one command** (`python3 deploy.py`: venv, dependencies, config, model checks,
server — all automated, in Python).

---

## The hard problem

A message that goes **to a customer** can't be wrong. But the deployment target
was a **CPU-only box with a <15s latency budget** — which rules out every large
model. Small models are fast enough, but they demonstrably miss dangerous
translation errors (invented refund promises, changed amounts, reversed
meanings).

**The answer: don't trust any single layer. Stack three, and only let them get stricter.**

```
 English + Spanish
        │
        ▼
 ┌─────────────────┐   first judgement: accuracy, tone,
 │  1. Local LLM    │   verdict — as schema-validated JSON
 │  (granite3.3:2b) │   (~2GB model via Ollama, CPU-fast)
 └────────┬────────┘
          ▼
 ┌─────────────────┐   deterministic rules, pure Python, ~0ms:
 │  2. Safety gate  │   changed/added numbers & currencies,
 │  (rule-based)    │   unauthorised commitments (refunds,
 └────────┬────────┘   guarantees, legal admissions, deadlines),
          ▼            dropped negations that reverse meaning
 ┌─────────────────┐
 │  3. Semantic     │   open-source cross-lingual NLI model
 │  check (NLI)     │   (mDeBERTa-v3 XNLI, MIT, ~280M params);
 └────────┬────────┘   catches pure meaning-swaps with no
          ▼            trigger words — blocks only when BOTH
      Verdict          reading directions agree it contradicts
```

**Escalate-only invariant:** every layer may raise the verdict
(`Send → Review First → Do Not Send`) and may never lower it. A clean pass
through a later layer can't undo an earlier block. This single design rule is
what makes a small, fast model safe to use at all.

---

## Measured results

**Validation method:** a 152-case labelled evaluation set. To avoid grading my
own homework, 100 of the cases were written *blind* — the writers were given
only the labelling rubric, never the checker's rules — including an
adversarial batch written specifically to evade word-based checks.

| Metric (152-case set) | Result |
|---|---|
| Dangerous messages rated **Send** (the must-be-zero number) | **0** |
| Safe messages wrongly blocked (false alarms) | **0** |
| Dangerous messages flagged (`Review First` or blocked) | **100%** |
| Latency per request on CPU | ~8–13s (budget: 15s) |

The ablation study showed each layer earns its place: the model alone let **10**
dangerous messages through as `Send`; adding the semantic check cut that to 2;
adding the deterministic gate cut it to **0**. The full small-model stack also
scored higher than a **7× larger model** equipped with the same safety gate — at
roughly **1/8th the latency**.

The blind round-1 run also *found real bugs* (e.g. a regex that matched
`reemplazo`/"replacement" as a time-window commitment, and one-directional NLI
false positives on number-heavy sentences) — each fix was made at the level of
the *rule*, not the failing example, and pinned with regression tests.

---

## Engineering practices

- **Schema-first:** every model response is validated (Pydantic) before display;
  malformed output triggers a re-ask, then a clean failure — never junk output.
- **Provider-agnostic:** one adapter file talks to models; local (Ollama),
  OpenAI, Anthropic, or Google is a config switch, not a rewrite.
- **83 automated tests**, all offline (no network, no model downloads) —
  the ML layer is tested through mocks; its *accuracy* is measured by the eval
  harness, not asserted by unit tests.
- **Fail-open where safe, fail-closed where not:** if the optional NLI stack is
  missing or crashes, the tool degrades gracefully to model + gate; the
  deterministic gate itself has no failure mode (pure stdlib).
- **Deliberate scope honesty:** politely *implied* commitments ("the money will
  be back in your account") floor at `Review First` rather than a hard block —
  documented as the human-review boundary instead of over-fitting the rules to
  the test set.
- **Licence-clean:** every shipped component is MIT/Apache-2.0; strong
  candidates with non-commercial licences were evaluated and rejected.

## Stack

Python · Pydantic · Ollama (`granite3.3:2b`) · Hugging Face Transformers
(mDeBERTa-v3-XNLI, CPU) · stdlib `http.server` · pytest — deployed with a
single-command Python installer on Linux.
