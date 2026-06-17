# NEXT — Briarios

## Current State

Research complete. 5 ADRs accepted. Scaffolding Q&A in progress (1 of 5 questions resolved).

## Immediate Next Action

Continue scaffolding Q&A — Q2 through Q5 before writing any code.

**Q2:** How does Briarios talk to OpenHands?
- Option A: `ManagedAPIServer` subprocess (local/dev — starts `python -m openhands.agent_server`, health-checked)
- Option B: OpenHands Cloud API (RESTful, async, designed for external orchestrators)
- Option C: OpenHands SDK (embedded, composable)

Note: my pre-formed recommendation is in `docs/internal/scaffolding-recommendations.md` — review after JP answers.

**Q3:** Configuration format?

**Q4:** Testing strategy — unit tests first or integration from day one?

**Q5:** Protocol standards — MCP/A2A: design for now, implement v0.2?

## After Q&A — First Code

Build order confirmed:
1. `briarios/triage/` — triage scorer + request analyzer (issue-level)
2. `briarios/budget/` — QoS policy engine (monetary + time + quality + turns) + break-glass
3. `briarios/router/` — LiteLLM integration (executor layer)
4. `briarios/verification/` — sequential pipeline, early exit

## ADR Debt

ADR-003 (budget gate) needs revision — current version describes a simple token counter; actual design is a 4-dimension QoS policy engine with break-glass.

## Key Files

- `docs/research/build-vs-buy.md` — landscape analysis
- `docs/decisions/` — all 5 ADRs
- `docs/architecture/meta-layer-design.md` — component design
- `docs/internal/scaffolding-recommendations.md` — pre-formed Q&A answers (internal)
- Obsidian quickref: `Library/AI/projects/legionforge-briarios-quickref.md`
- Session notes: `Library/AI/Sessions/session-2026-06-16.md` (Session 4)
