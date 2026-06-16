# ADR-001: Bolt-On Meta-Layer vs. Standalone Framework

**Date:** 2026-06-16
**Status:** Accepted
**Deciders:** JP Cruz

---

## Context

LegionForge-Briarios was conceived as a parallel agentic workstream system: triage GitHub issues by priority, route them to appropriately-sized AI agents, gate parallel spawning on resource budgets, and run independent verification before any merge.

The question: build the full stack from scratch, or build only the layer that doesn't exist and consume mature tools for the rest?

## Decision

**Build a security-native meta-layer. Consume OpenHands, LangGraph, and Anthropic Managed Agents for execution.**

## Rationale

- OpenHands already solves the GitHub issue → coding agent → PR pipeline (65K+ stars, production-proven, MIT)
- LangGraph already solves the workflow graph problem and is already in LegionForge
- Anthropic Managed Agents already solves the orchestration + token budget problem for Claude-based fleets
- None of these solve: security-aware triage, model-tier routing, multi-resource budget gating, or independent verification pipelines
- Building the full stack would take months and produce worse primitives than what's already available

## Consequences

**Positive:**
- Scope is tightly bounded — only build what doesn't exist
- Faster to a working system (weeks vs. months)
- Inherits production maturity from OpenHands and LangGraph
- Briarios value is concentrated in the five novel capabilities

**Negative:**
- Dependency on OpenHands API stability
- Anthropic Managed Agents is still in public beta (May 2026) — may change
- Meta-layer means Briarios is harder to understand without knowing the underlying tools

## Alternatives Considered

1. **Full standalone framework** — rejected; recreates solved problems poorly
2. **Pure CrewAI extension** — rejected; abstractions limit control at production complexity
3. **Microsoft Agent Framework** — rejected; ecosystem lock-in, overkill for focused meta-layer
