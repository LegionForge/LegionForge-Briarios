# ADR-004: Verification Pipeline Ordering — Sequential vs. Parallel

**Date:** 2026-06-16
**Status:** Accepted
**Deciders:** JP Cruz

---

## Context

After an agent produces a diff, Briarios runs it through a verification pipeline before surfacing it for human review:

1. **Test Agent** — runs CI, reports pass/fail and test delta
2. **Review Agent** — reads diff, checks scope and security
3. **Eval Agent** — checks diff against issue's Done-When criteria
4. **Pentest Agent** — (S tier only) checks whether fix introduces new vulns

Two execution models: sequential (each agent waits for the previous) or parallel (all agents run simultaneously, results collected).

## Decision

**Sequential execution with early exit on failure.**

```
Test Agent
  → PASS: continue  |  FAIL: stop, report, return to Assigned
Review Agent
  → PASS: continue  |  FAIL: stop, report, return to Assigned
Eval Agent
  → PASS: continue  |  FAIL: stop, report, return to Assigned
Pentest Agent (S tier only)
  → PASS: continue  |  FAIL: stop, report, return to Assigned
Human Gate
```

## Rationale

### The dependencies are real, not artificial

The pipeline steps have genuine data dependencies:

- Running Eval Agent (did the change meet Done-When?) is meaningless if Test Agent says CI is broken — the change doesn't work yet
- Running Pentest Agent on a change that Review Agent flagged as out-of-scope wastes tokens on the wrong artifact
- Parallel execution pays for all four agents even when the first check makes the rest irrelevant

### Cost: sequential with early exit wins decisively

Research data:
- A 3-agent parallel pipeline consumes ~29,000 tokens vs. ~10,000 for sequential — nearly 3x
- A UIUC study found multi-agent systems consume 4–220x more tokens than single-agent counterparts
- Parallel saves wall-clock time (collapses to slowest agent) but adds ~950ms coordination overhead

For a verification pipeline where most failures will be caught at step 1 (CI), sequential with early exit pays approximately:
- **Failing runs:** 1x (stop at first failure)
- **Passing runs:** ~2x (all steps complete, pentest adds cost only on S tier)

Parallel pays 3–4x on every run regardless of outcome.

### "Fail fast" is a security principle

Sequential maps directly to the security posture: stop at the first signal that the change is wrong. Don't invest more resources verifying a broken change — return it to the agent for revision and start over.

## Consequences

**Positive:**
- Significant token cost reduction (1–2x vs 3–4x for parallel)
- Fail-fast behavior: first failure stops the chain immediately
- Simpler coordination — no fan-out/fan-in merge step needed
- Easier to reason about: each step's output is the next step's input context

**Negative:**
- Wall-clock time is sum of all steps (not max of all steps as in parallel)
- A slow CI run (Test Agent) holds up Review and Eval even if they'd be fast
- Acceptable: verification is not time-critical; correctness is

## Alternatives Considered

1. **Fully parallel** — rejected; 3x token cost; pays for all agents even when first check fails
2. **Parallel Test+Review, sequential Eval+Pentest** — rejected; adds coordination complexity for marginal wall-clock gain; Test and Review don't actually depend on each other but their outputs both feed Eval, making the merge step non-trivial
3. **Parallel all, abort-on-first-failure** — rejected; race condition semantics; hard to reason about which failure triggered the abort

## Sources

- [Sequential vs Parallel Tool/Reasoning in AI Agents](https://medium.com/@speaktoharisudhan/sequential-vs-parallel-reasoning-in-ai-agents-9286040e1f6b)
- [Parallel Concurrency in Production AI Agents](https://zylos.ai/research/2026-04-26-parallel-concurrency-agent-execution/)
- [Scale Parallel AI Agents Without Losing Quality 2026](https://getunblocked.com/blog/scale-parallel-ai-agents/)
- [Single vs Sequential vs Parallel AI Agents for Real Workflows](https://www.geeky-gadgets.com/ai-agent-patterns/)
