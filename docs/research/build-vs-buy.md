# Build vs. Buy — Landscape Research

**Date:** 2026-06-16
**Author:** JP Cruz + Claude Sonnet 4.6
**Status:** Complete — recommendation: bolt-on meta-layer

---

## What We Were Looking For

A production-ready system for:
1. Triaging GitHub issues by security risk, blast radius, complexity, and blocking relationships
2. Routing issues to appropriately-sized AI agents (different model tiers)
3. Gating parallel agent spawning based on token/CPU/GPU budget
4. Running independent verification agents (test, review, eval, pentest) before any merge
5. Security-native guardrails (scope enforcement, no-direct-to-main, mandatory risk assessment)

---

## What Already Exists

### OpenHands — closest match for the coding agent piece
- **65K+ GitHub stars**, actively maintained, MIT license
- GitHub issue → autonomous agent → PR resolution via `fix-me` label trigger
- GitHub Actions integration (fully automated pipeline)
- Agentic loop: receive task → plan → write code → execute → observe → iterate
- **OpenHands Index** (Jan 2026): live leaderboard evaluating models on issue resolution, greenfield apps, frontend dev, testing, info gathering
- **Verdict: consume this.** Don't reimplement the coding agent.

### Anthropic Managed Agents (public beta, May 2026)
- Lead agent orchestrates; sub-agents fan out in parallel
- Built-in guardrail: **sub-agents cannot spawn their own sub-agents** (kills runaway cost multiplier)
- Key finding from Anthropic's own research: "token usage explains 80% of performance variance"
- **Verdict: use for orchestration API** when managing Claude-based agent fleets.

### LangGraph (already in LegionForge)
- Graph-based, models workflows as directed state machines with conditional edges
- Best for precise control over branching, retries, human-in-the-loop steps
- MIT license, LangGraph Platform available (paid, optional)
- **Verdict: extend this** for the Briarios workflow graph rather than rebuild.

### CrewAI 0.95 (Feb 2026)
- Role-based "crew" abstraction — easy to set up, good for demos
- Added async crew runner (experimental) and improved Anthropic/Google tool routing
- **Weakness:** limited fine-grained control; abstractions fight you at production complexity
- **Verdict: skip.** LangGraph gives more control for what we need.

### Microsoft Agent Framework 1.0 (GA Apr 2026)
- Consolidated AutoGen + Semantic Kernel into a single SDK (Python + .NET)
- Production-ready, well-documented
- **Weakness:** Microsoft ecosystem assumptions; overkill for a focused meta-layer
- **Verdict: skip.**

### Open Multi-Agent (OMA)
- TypeScript-native; converts goals to task DAGs automatically
- Confirmed production use
- **Weakness:** TypeScript-only, newer project, less ecosystem
- **Verdict: watch, don't adopt yet.** Python-first is right for LegionForge ecosystem.

---

## What Doesn't Exist Anywhere

| Gap | Why it matters |
|-----|---------------|
| Security-risk triage scoring (blast radius × CVE severity × complexity × blocking) | No orchestrator understands issue security context |
| Model tier assignment by issue type (S/A/B/C → Opus/Sonnet/Haiku/local) | Existing routers are task-type agnostic, not security-aware |
| Multi-resource budget gate (token + CPU + GPU/VRAM awareness) | Token-only budgets exist; hardware-aware gating does not |
| Independent verification pipeline (pentest + eval + test agents, each stateless) | No existing orchestrator enforces separation of concerns in verification |
| Security-native guardrails (scope enforcement, mandatory risk assessment, no-direct-to-main) | General orchestrators have no security posture |

These five gaps are Briarios's reason to exist.

---

## Recommendation: Bolt-On Meta-Layer

**Don't build what others have built. Build the layer that doesn't exist.**

```
Consume:
  - OpenHands (coding agent execution + GitHub Actions integration)
  - LangGraph (workflow graph, already in LegionForge)
  - Anthropic Managed Agents API (orchestration for Claude-based fleets)

Build (Briarios):
  - Triage scorer
  - Model-tier router
  - Multi-resource budget gate
  - Verification pipeline coordinator
  - Security-native guardrails
```

This is weeks of focused work, not months of framework reinvention. And the bolt-on surface gives LegionForge users (and eventually others) a layer they can't get elsewhere.

---

## Open Questions for ADR

1. **Primary coding agent:** OpenHands vs. direct LangGraph agent with tool access? OpenHands is more autonomous; LangGraph gives more control. Hybrid possible (OpenHands for C/B tier, LangGraph for A/S tier)?
2. **Budget gate implementation:** polling loop vs. event-driven (Prometheus metrics → gate webhook)?
3. **Verification pipeline:** sequential (cheaper, slower) vs. parallel (faster, higher token cost)?
4. **Kanban backend:** GitHub Projects native API vs. embedded state in Postgres (LegionForge already has DB)?

---

## Sources

- [OpenHands](https://www.openhands.dev/)
- [OpenHands Index — Jan 2026](https://www.openhands.dev/blog/openhands-index)
- [Claude Agent SDK & Managed Agents — Q2 2026](https://zylos.ai/research/2026-04-20-claude-agent-sdk-managed-agents-architecture/)
- [Best open source agent frameworks 2026](https://www.firecrawl.dev/blog/best-open-source-agent-frameworks)
- [7 Multi-Agent Orchestration Platforms: Build vs Buy 2026](https://www.augmentcode.com/tools/multi-agent-orchestration-platforms-build-vs-buy)
- [AI Agent Frameworks Compared 2026](https://pecollective.com/blog/ai-agent-frameworks-compared/)
- [Anthropic and OpenAI Agent Orchestration 2026](https://flocker.md/blog/anthropic-openai-agent-orchestration/)
