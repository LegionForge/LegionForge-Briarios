# ADR-003: Budget Gate Implementation — Polling vs. Event-Driven

**Date:** 2026-06-16
**Status:** Accepted
**Deciders:** JP Cruz

---

## Context

Before spawning any agent lane, Briarios must check five resource conditions:
1. Active lane count < MAX_PARALLEL
2. Token spend today < DAILY_TOKEN_BUDGET
3. CPU load < 80%
4. GPU/VRAM headroom > MIN_VRAM_GB
5. GitHub API requests remaining > headroom threshold

Two implementation approaches: polling (check on a fixed interval) or event-driven (react to threshold-crossing signals).

## Decision

**v0.1: Polling. v0.2+: migrate to event-driven (Prometheus Alertmanager → webhook).**

### v0.1 Implementation

```python
class BudgetGate:
    def can_open_lane(self) -> tuple[bool, str]:
        # token spend: Postgres counter (fast, no rate limit)
        # active lanes: Postgres row count (fast)
        # CPU: psutil.cpu_percent() — local, instant
        # VRAM: nvidia-smi or Metal perf counters — local, instant
        # GitHub API: X-RateLimit-Remaining header cached from last request
        ...
```

All five checks are cheap local reads or cached values. No external API calls in the hot path. Poll interval: configurable, default 10s when a lane is queued.

### v0.2+ Migration Path

Replace the polling loop with:
- Prometheus Alertmanager rule fires when CPU > 80% or VRAM < MIN_VRAM_GB
- Alertmanager → webhook → Briarios budget gate sets `lane_blocked=True` in Postgres
- Budget gate checks Postgres flag instead of polling metrics directly
- Flag clears when Alertmanager sends resolved notification

## Rationale

### Why not event-driven from day one?

The budget gate is **not in the hot path**. It runs once per issue assignment, before a multi-minute agent run. The latency difference between polling and event-driven at this frequency is irrelevant — we're gating a task that takes minutes, not milliseconds.

Event-driven adds:
- Alertmanager configuration and alert rule authoring
- Webhook receiver endpoint in Briarios
- "Resolved" notification handling to re-open lanes
- Testing of the alert firing → gate reaction loop

That complexity is warranted at scale. It is overhead at v0.1 with one developer and three maximum parallel lanes.

### Research data

- Event-driven reduces system latency by 70–90% vs polling — relevant at scale, not at our current frequency
- Prometheus scrape overhead: 2.2ms average, <0.25% per scrape — negligible either way
- Polling overwhelms infrastructure when agents and data sources multiply — not our constraint at v0.1

## Consequences

**Positive (v0.1):**
- Simple implementation, easy to test, no new infrastructure
- All five checks are local reads — no external dependencies in the gate
- Easy to reason about: "check before spawn, block if any fail"

**Negative (v0.1):**
- 10s poll interval means a freed lane takes up to 10s to be picked up — acceptable
- Polling loop runs even when no issues are queued — minor waste, acceptable

**v0.2+ migration:** straightforward — replace the polling loop with a Postgres flag check; keep the same `can_open_lane()` interface.

## Sources

- [Event-Driven vs. Polling Architectures for Agent Triggers](https://agentblueprint.substack.com/p/event-driven-vs-polling-architectures)
- [Event-Driven AI Agent Architecture Guide 2026](https://fast.io/resources/ai-agent-event-driven-architecture/)
- [Prometheus Metrics for Serverless environments](https://groups.google.com/g/prometheus-developers/c/FPe0LsTfo2E/m/yS7up2YzAwAJ)
