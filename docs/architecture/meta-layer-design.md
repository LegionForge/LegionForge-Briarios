# Briarios Meta-Layer — Architecture Design

**Date:** 2026-06-16
**Status:** Draft
**Author:** JP Cruz + Claude Sonnet 4.6

---

## Layer Map

```
┌──────────────────────────────────────────────────────────────┐
│                     ISSUE INTAKE                             │
│  GitHub Issues → Triage Scorer → Tier Assignment            │
└────────────────────────────┬─────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────┐
│                     BUDGET GATE                              │
│  token spend + active agents + CPU load + VRAM headroom     │
│  ALL conditions pass → open lane    ANY fail → queue        │
└────────────────────────────┬─────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────┐
│                  MODEL-TIER ROUTER                           │
│  S → Opus    A → Sonnet    B → Sonnet/Haiku    C → Haiku    │
└────────────────────────────┬─────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼───────┐  ┌────────▼───────┐  ┌───────▼────────┐
│  OpenHands     │  │  LangGraph     │  │  Anthropic     │
│  (coding agent │  │  (workflow     │  │  Managed       │
│   execution)   │  │   graph)       │  │  Agents API    │
└────────┬───────┘  └────────┬───────┘  └───────┬────────┘
         └───────────────────┼───────────────────┘
                             │ diff + branch
┌────────────────────────────▼─────────────────────────────────┐
│                VERIFICATION PIPELINE                         │
│                                                              │
│  1. Test Agent     → runs CI, reports pass/fail             │
│  2. Review Agent   → diff scope check + security scan       │
│  3. Eval Agent     → verifies Done-When criteria met        │
│  4. Pentest Agent  → (S tier only) new vuln check          │
│                                                              │
│  Each agent is stateless — sees artifact only, not prior    │
│  agent's reasoning                                           │
└────────────────────────────┬─────────────────────────────────┘
                             │
                      Human Gate (required)
                      no merge without explicit approval
```

---

## Triage Scorer

### Scoring Matrix

| Axis | 1 | 3 | 5 |
|------|---|---|---|
| **Security risk** | cosmetic | functional gap | exploitable CVE |
| **Blast radius** | isolated file | subsystem | cross-cutting |
| **Complexity** | unambiguous | some judgment | ADR needed |
| **Blocking** | standalone | blocks 1 issue | blocks 2+ issues |

**Total score → Tier:**
- 16–20 → **S** (Security)
- 10–15 → **A** (Complex)
- 5–9 → **B** (Bug)
- 1–4 → **C** (Quick win)

### ADR Gate
Issues with complexity score = 5 (ADR needed) are flagged `blocked: adr-needed` and held out of the agent queue until an ADR document is written and merged.

---

## Budget Gate

```python
class BudgetGate:
    MAX_PARALLEL = 3               # concurrent agent lanes
    DAILY_TOKEN_BUDGET = 500_000   # tokens (configurable)
    MAX_CPU_PERCENT = 80.0
    MIN_VRAM_GB = 2.0
    GH_API_HEADROOM = 100          # requests remaining

    def can_open_lane(self) -> tuple[bool, str]:
        checks = [
            (self.active_lanes < self.MAX_PARALLEL,     "max parallel lanes reached"),
            (self.tokens_today < self.DAILY_TOKEN_BUDGET, "daily token budget exhausted"),
            (self.cpu_percent < self.MAX_CPU_PERCENT,   "CPU overloaded"),
            (self.vram_free_gb > self.MIN_VRAM_GB,      "insufficient VRAM"),
            (self.gh_api_remaining > self.GH_API_HEADROOM, "GitHub API rate limit near"),
        ]
        for passed, reason in checks:
            if not passed:
                return False, reason
        return True, "ok"
```

All values configurable via `briarios.yaml`. Defaults are conservative.

---

## Model-Tier Router

```python
TIER_MODEL_MAP = {
    "S": "claude-opus-4-7",           # security — max reasoning
    "A": "claude-sonnet-4-6",         # complex bugs, ADRs
    "B": "claude-sonnet-4-6",         # simple bugs (Haiku if budget tight)
    "C": "claude-haiku-4-5-20251001", # quick wins, cosmetic, docs
}

# Local model fallback (when cloud budget exhausted)
TIER_LOCAL_FALLBACK = {
    "B": "llama3.1:8b",
    "C": "qwen2.5:3b",
}
```

S and A tier issues never fall back to local models — correctness > cost for security and architecture.

---

## Verification Pipeline Detail

### Test Agent
- Runs project CI (`make ci` or equivalent)
- Reports: pass/fail, test delta (new failures introduced), coverage change
- Blocks merge on any new failure

### Review Agent
- Independent read of the diff (no context from the coding agent)
- Checks: files touched vs. issue scope, no credentials/secrets introduced, no scope creep
- Flags but does not block on warnings (human decides)

### Eval Agent
- Reads the issue's Done-When criteria
- Reads the diff
- Binary verdict: Done-When met / not met
- Blocks merge if not met

### Pentest Agent (S tier only)
- Reads the security issue + fix
- Attempts to identify: new attack surface introduced, incomplete fix (bypass possible), hardening gaps
- Output: structured threat assessment
- Blocks merge if new vulns found

---

## Kanban State Machine

```
Backlog → Assigned → In Progress → Verification → Human Review → Done
                                       │
                              (any verification fail)
                                       │
                                   Back to Assigned
```

Implemented against GitHub Projects API. Each issue card carries:
- Assigned tier (S/A/B/C)
- Assigned model
- Agent run ID
- Verification results
- Human reviewer

---

## Open Questions (from ADR research)

1. **Primary coding agent:** OpenHands vs. direct LangGraph agent? Recommendation: OpenHands for B/C tier (autonomous, fast); LangGraph-native agent for A/S tier (more control, human-in-the-loop).
2. **Budget gate:** polling vs. event-driven (Prometheus → webhook)? Start with polling; migrate to event-driven at scale.
3. **Verification pipeline:** sequential vs. parallel? Sequential is cheaper and sufficient for initial implementation.
4. **Kanban backend:** GitHub Projects API vs. Postgres? GitHub Projects for visibility; Postgres for internal state.
