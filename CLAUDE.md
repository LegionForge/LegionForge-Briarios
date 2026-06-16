# CLAUDE.md — LegionForge-Briarios

## What This Project Is

A security-native orchestration meta-layer for parallel agentic development workstreams. Briarios sits on top of OpenHands, LangGraph, and Anthropic Managed Agents — it adds what they don't have: security-aware issue triage, model-tier routing, multi-resource budget gating, and an independent verification pipeline.

Named for Briareos (Βριάρεως), one of the Hecatoncheires — 50 heads, 100 arms. Holds and executes a hundred things simultaneously.

## Status

Pre-scaffolding. See `docs/research/build-vs-buy.md` for the landscape analysis and `docs/decisions/ADR-001-bolt-on-vs-standalone.md` for the core architectural decision.

## What We Build (Briarios)

- `briarios/triage/` — Issue scorer (security risk × blast radius × complexity × blocking)
- `briarios/router/` — Model-tier router (S/A/B/C → Opus/Sonnet/Haiku/local)
- `briarios/budget/` — Multi-resource budget gate (token + CPU + VRAM + GitHub API)
- `briarios/verification/` — Verification pipeline coordinator (test/review/eval/pentest agents)
- `briarios/guardrails/` — Security-native guardrails (scope enforcement, no-direct-to-main)

## What We Consume (don't reimplement)

- **OpenHands** — coding agent execution + GitHub Actions integration
- **LangGraph** — workflow graph (already in LegionForge)
- **Anthropic Managed Agents** — orchestration API for Claude-based agent fleets

## License

MIT
