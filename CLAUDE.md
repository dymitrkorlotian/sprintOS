# Sprint OS: project context

A local-first personal operating system (Mac first, then web and phone): sprints, tasks, notes, links, files and a context graph, with AI automating the manual rituals. This is a from-scratch rebuild of [Sprint](https://github.com/dymitrkorlotian/sprint), the web app, which stays in use meanwhile. Sprint's `docs/vision.md`, `docs/architecture/context-graph.md` and `docs/specs/` are the product's requirements and concepts, not its design.

## Current phase: ARCHITECTURE

- No application code until the founding architecture, `docs/decisions/0001-*.md`, is **Accepted** by the user.
- The bar, set by the user: top-notch technology choices (optimized, secure, performant, stable), decided from first principles and on evidence. Earlier choices in Sprint are inputs, not constraints.
- Decided already: start from an empty database (no migration from Sprint); no export feature for now; never require a home server; about $0 running cost for the user; the data model can serve many tenants later.

## Several agents at once

Work is coordinated by the orchestrator (Agent 0) on the "Agent board", issue [#57 in `dymitrkorlotian/sprint`](https://github.com/dymitrkorlotian/sprint/issues/57), until this repo gets its own board. Work only on what the board assigns to your session; if nothing is, stop and say so.

## Conventions

- Branch per task, PR into `main`, merged by Agent 0 once ready.
- Decisions are ADRs in `docs/decisions/`, numbered `NNNN-title.md`. When work changes a decision, update the ADR in the same change.
- Plain, short English in docs.
