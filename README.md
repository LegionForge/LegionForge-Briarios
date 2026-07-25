# LegionForge-Briarios

#### Status: R&D

> Named for Briareos (Βριάρεως) — one of the Hecatoncheires of Greek myth. 50 heads, 100 arms. An entity that holds and executes a hundred things simultaneously. In the Appleseed universe, the cyborg who protects and orchestrates.

💛 [Support this project](https://legionforge.org/donations) — LegionForge is open-source and independently maintained.

**Briarios is a security-native orchestration meta-layer for parallel agentic development workstreams.**

It does not replace OpenHands, LangGraph, or Anthropic Managed Agents. It sits on top of them and adds what they don't have: security-first issue triage, model-tier routing, multi-resource budget gating, and an independent verification pipeline.

---

## What Problem This Solves

Modern AI agent frameworks (CrewAI, AutoGen, LangGraph) give you the primitives to run parallel agents. What they don't give you:

- **Security-aware triage** — not all issues have the same blast radius; a CVE and a UI label cleanup shouldn't compete for the same agent tier
- **Model-tier routing** — routing Opus at a cosmetic refactor is waste; routing Haiku at a CVE is risk
- **Multi-resource budget gating** — token budgets exist; CPU/GPU/VRAM awareness does not
- **Independent verification** — no existing orchestrator enforces a separate pentest/eval agent reviewing every change before merge
- **Security-native guardrails** — scope enforcement, no-direct-to-main, mandatory risk assessment per change

---

## Architecture (Meta-Layer)

```
┌─────────────────────────────────────────────────────┐
│                  BRIARIOS META-LAYER                │
│                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Triage     │  │  Model-Tier  │  │  Budget   │ │
│  │  Scorer     │→ │  Router      │→ │  Gate     │ │
│  └─────────────┘  └──────────────┘  └───────────┘ │
│                                           │         │
│                          ┌────────────────┘         │
│                          ▼                          │
│  ┌─────────────────────────────────────────────┐   │
│  │           AGENT EXECUTION LAYER             │   │
│  │  OpenHands (coding) │ LangGraph (workflows) │   │
│  │  Anthropic Managed Agents (orchestration)   │   │
│  └─────────────────────────────────────────────┘   │
│                          │                          │
│                          ▼                          │
│  ┌─────────────────────────────────────────────┐   │
│  │         VERIFICATION PIPELINE               │   │
│  │  Test Agent → Review Agent → Eval Agent     │   │
│  │  → Pentest Agent (security tier only)       │   │
│  └─────────────────────────────────────────────┘   │
│                          │                          │
│                   Human Gate (required)             │
└─────────────────────────────────────────────────────┘
```

---

## Triage Scoring

Every issue is scored on four axes before assignment:

| Axis | Low (1) | Med (3) | High (5) |
|------|---------|---------|----------|
| Security risk | cosmetic | functional gap | exploitable CVE |
| Blast radius | isolated file | subsystem | cross-cutting |
| Complexity | unambiguous | some judgment | ADR needed |
| Blocking | standalone | blocks 1 | blocks 2+ |

**Tier assignment:**
- **S** (score 16–20): Security issues → Opus
- **A** (score 10–15): Complex bugs, ADRs → Sonnet
- **B** (score 5–9): Simple bugs → Sonnet/Haiku
- **C** (score 1–4): Quick wins, cosmetic, docs → Haiku / local model

---

## Budget Gate

Before spawning any parallel agent lane:

- Active agent count < `MAX_PARALLEL` (default: 3)
- Token spend today < `DAILY_TOKEN_BUDGET`
- CPU load < 80%
- GPU VRAM headroom > `MIN_VRAM_GB`
- GitHub API requests < rate limit threshold

All conditions must pass. Any failure queues the agent and retries on next budget tick.

---

## Verification Pipeline

```
Agent produces diff
  → Test Agent        (runs CI, reports pass/fail)
  → Review Agent      (reads diff, checks scope creep + security)
  → Eval Agent        (checks diff against issue Done-When criteria)
  → Pentest Agent*    (*security tier only — checks fix doesn't introduce new vulns)
  → Human gate        (no merge without explicit approval)
```

Each agent in the pipeline is independent — it sees only the artifact, not the prior agent's reasoning.

---

## What We Build vs. What We Consume

| Capability | Source |
|-----------|--------|
| Coding agent execution | OpenHands (open source, 65K+ stars) |
| Workflow graph orchestration | LangGraph (already in LegionForge) |
| Managed multi-agent API | Anthropic Managed Agents (public beta) |
| **Triage scorer** | **Briarios — build** |
| **Model-tier router** | **Briarios — build** |
| **Multi-resource budget gate** | **Briarios — build** |
| **Verification pipeline** | **Briarios — build** |
| **Security-native guardrails** | **Briarios — build** |

---

## License

MIT — see [LICENSE](LICENSE)

## Architectural Decisions

All four open questions from initial research are now resolved:

| ADR | Decision |
|-----|---------|
| [ADR-001](docs/decisions/ADR-001-bolt-on-vs-standalone.md) | Bolt-on meta-layer — consume OpenHands + LangGraph + Anthropic Managed Agents |
| [ADR-002](docs/decisions/ADR-002-agent-tier-selection.md) | LangGraph-native for S/A tier; OpenHands for B/C tier |
| [ADR-003](docs/decisions/ADR-003-budget-gate-implementation.md) | Polling (v0.1) → event-driven Prometheus/Alertmanager (v0.2+) |
| [ADR-004](docs/decisions/ADR-004-verification-pipeline-ordering.md) | Sequential with early exit — fail fast, 1–2x token cost vs 3–4x for parallel |
| [ADR-005](docs/decisions/ADR-005-kanban-state-backend.md) | Hybrid: GitHub Projects (display) + Postgres (orchestration state) |

## Status

**Research complete. ADRs accepted. Ready for scaffolding.**

See [docs/research/build-vs-buy.md](docs/research/build-vs-buy.md) for full landscape analysis.
