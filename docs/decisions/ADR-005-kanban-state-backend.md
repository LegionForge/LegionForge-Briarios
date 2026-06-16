# ADR-005: Kanban State Backend — GitHub Projects vs. Postgres

**Date:** 2026-06-16
**Status:** Accepted
**Deciders:** JP Cruz

---

## Context

Briarios needs to track issue state across the pipeline (Backlog → Assigned → In Progress → Verification → Human Review → Done). Two candidate backends: GitHub Projects (native, human-visible) or Postgres (LegionForge already has it, fast, queryable).

## Decision

**Hybrid: GitHub Projects for human-visible display, Postgres for internal orchestration state.**

| State type | Audience | Backend |
|-----------|----------|---------|
| Issue tier, assigned model, PR link, pipeline stage, verification verdict | Humans | GitHub Projects (GraphQL API, Projects v2) |
| Active lane count, token spend today, VRAM available, agent run IDs, verification step results, audit trail | Briarios engine | Postgres |

**Sync direction:** Postgres → GitHub Projects (one-way). GitHub Projects is a display layer, not a source of truth.

## Rationale

### Two different audiences, two different query patterns

**GitHub Projects** is optimized for human consumption:
- Native visibility — anyone with repo access sees the board without additional tooling
- No new infrastructure — it already exists
- Kanban column drag-and-drop, filtering, grouping — human UX features
- Rate limited at 5,000 requests/hour (authenticated) — fine for human-frequency updates, problematic for high-frequency orchestration reads

**Postgres** is optimized for machine consumption:
- Fast SQL queries — budget gate checks active lane count dozens of times per minute
- No rate limits — internal service
- Transactional — budget counter updates must be atomic
- Already running in LegionForge — zero additional infrastructure
- Full audit trail with timestamps, indexed, queryable

### Orchestration state has no business in GitHub Projects

Fields like `vram_available_gb`, `token_spend_today`, `agent_run_id`, `verification_step_3_result` are not human-relevant at the kanban level — they're internal bookkeeping. Pushing them to GitHub Projects wastes API quota and pollutes the human-visible board with machine noise.

### Rate limit math

Budget gate checks 5 conditions, potentially every 10 seconds, across up to 3 active lanes. That's up to 900 checks/hour — 18% of the authenticated rate limit just for budget polling. Using Postgres for internal state keeps GitHub API budget for meaningful updates (stage transitions, PR links, final verdicts).

## Implementation

```
Postgres tables (Briarios-owned):
  - briarios_lanes          (id, issue_id, tier, model, status, agent_run_id, created_at)
  - briarios_budget         (date, token_spend, lane_count)
  - briarios_verification   (lane_id, step, result, detail, timestamp)

GitHub Projects (display):
  - Column = pipeline stage  (mapped from Postgres status on transition)
  - Custom fields: Tier, Model, PR URL, Verification verdict
  - Updated via GraphQL mutation on each Postgres state transition
```

## Consequences

**Positive:**
- Budget gate reads are instant (Postgres, no rate limit)
- Human board stays clean and readable
- Audit trail is queryable SQL, not GitHub API pagination
- No rate limit risk from orchestration chatter

**Negative:**
- Sync bridge adds a code path: every Postgres state transition must also update GitHub Projects
- State can momentarily diverge (Postgres updates; GitHub Projects update fails) — needs retry logic
- Two systems to monitor

**Mitigation for divergence:** GitHub Projects updates are best-effort with retry. Postgres is source of truth. If GitHub Projects falls behind, it self-heals on next transition. Staleness in the display layer is acceptable; staleness in the orchestration layer is not.

## Alternatives Considered

1. **GitHub Projects only** — rejected; rate limit risk for high-frequency orchestration reads; no structured audit trail; can't store machine-relevant internal state cleanly
2. **Postgres only** — rejected; no human-visible board without building a custom UI; GitHub Projects is already there and free
3. **Redis for orchestration state** — rejected; LegionForge has Postgres already; Redis adds infra for marginal speed gain that isn't needed at this scale

## Sources

- [GitHub Projects v2 GraphQL API](https://docs.github.com/en/graphql/reference/objects#projectv2)
- [Implementing a Kanban Board with GitHub](https://ones.com/blog/implementing-kanban-board-github-react/)
- [Event-Driven Architecture for AI Agents](https://atlan.com/know/event-driven-architecture-for-ai-agents/)
