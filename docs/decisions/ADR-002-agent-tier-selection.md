# ADR-002: Agent Selection by Issue Tier — LangGraph vs. OpenHands

**Date:** 2026-06-16
**Status:** Accepted
**Deciders:** JP Cruz

---

## Context

Briarios routes issues to agent execution layers based on tier (S/A/B/C). Two candidate agents exist for the coding work: LangGraph-native agents (already in LegionForge) and OpenHands (autonomous SWE agent, 65K+ stars, MIT). The question is which to use for which tier.

The core architectural difference:

- **LangGraph:** state is a persistent checkpoint graph. Every node reads/writes shared state. Supports human-in-the-loop interruptions mid-run, fault tolerance, conditional branching on intermediate results.
- **OpenHands:** state is the filesystem + shell history + LLM context window. Autonomous loop within a sandboxed container. "Set and forget" — the agent plans, codes, tests, and iterates without intervention.

## Decision

**Split by tier. LangGraph-native for S and A. OpenHands for B and C.**

| Tier | Agent | Rationale |
|------|-------|-----------|
| S (security) | LangGraph-native | Human-in-the-loop checkpoints are non-negotiable for CVE fixes and security hardening. Need explicit control over which files the agent reads/writes. Changes must pause at a human gate before the verification pipeline runs. |
| A (complex bugs, ADRs) | LangGraph-native | ADR-gated issues require a human approval step before implementation begins. LangGraph checkpoints enforce this naturally. |
| B (simple bugs) | OpenHands | Well-scoped, lower blast radius. Autonomous execution within a sandboxed container is appropriate. |
| C (quick wins) | OpenHands | Pure "set and forget" — label cleanup, doc stubs, noqa suppressions, cosmetic refactors. Autonomy is the point. |

## Rationale

LangGraph's checkpoint architecture is the decisive factor for S/A tier. A security fix that proceeds autonomously past a human review point is a security anti-pattern — the tool enforcing the gate should be the same tool running the agent, so the gate is structural, not procedural.

OpenHands's autonomy is a feature, not a risk, for B/C tier. Lower blast radius means a mistake is a revision, not an incident.

## Cost Implication

LangGraph agents carry more orchestration overhead than OpenHands's autonomous loop. This is acceptable for S/A tier because the cost of a mistake exceeds the cost of the overhead. For B/C tier, OpenHands's lighter footprint is the right tradeoff.

## Consequences

**Positive:**
- Human oversight is structurally enforced at S/A tier — no procedural workaround possible
- B/C tier gets the speed and efficiency of autonomous execution
- Briarios already has LangGraph (LegionForge dependency) — no new runtime for S/A tier

**Negative:**
- Two different agent runtimes to integrate and maintain
- OpenHands container lifecycle adds operational complexity for B/C tier
- LangGraph agents for S/A tier require more prompt engineering for security-context awareness

## Alternatives Considered

1. **OpenHands for all tiers** — rejected; no structural human-in-the-loop for S/A; filesystem-state autonomy is a risk at security tier
2. **LangGraph for all tiers** — rejected; overkill overhead for quick wins; OpenHands already solves the autonomous coding problem well
3. **Single agent with mode switching** — rejected; conflates two fundamentally different state models

## Sources

- [LangGraph vs. OpenHands: The 2026 Agent Framework Showdown](https://interconnectd.com/blog/33/)
- [OpenHands Software Agent SDK](https://arxiv.org/pdf/2511.03690)
- [AI Agent Frameworks Compared 2026](https://pecollective.com/blog/ai-agent-frameworks-compared/)
