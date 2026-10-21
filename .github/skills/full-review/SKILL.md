---
name: full-review
description: Run the complete read-only Laravel audit and save a dated CODE-REVIEW.md
disable-model-invocation: true
---
# FULL LARAVEL CODE REVIEW

Perform the complete READ-ONLY audit defined in
`.github/agents/review-orchestrator.agent.md`, following it step by step:

1. Read `.github/docs/review-rules.md` and `.github/docs/architecture/ARCHITECTURE.md`.
2. Discover the real stack from `AGENTS.md` and the repository; never assume optional technologies.
3. Run only safe checks that already exist. DO NOT modify code, tests, dependencies, lockfiles, configuration or database data.
4. Run the specialist reviews in `.github/agents/` in risk order: Laravel, Security, Database (with concurrency), Performance, Business Logic; then Architecture, Testing, API, Frontend, Maintainability (with clean code); then DevOps (with observability and dependencies). Skip specialists that do not apply.

Launch each specialist as a subagent by its ID (the file name): `#agent-id` in Copilot, the `subagent` tool in OpenCode, the Agent tool in Claude (the `.opencode/agents/` and `.claude/agents/` symlinks point at these files).
5. Challenge every finding with `.github/agents/devil-advocate.agent.md` and drop unsupported, duplicated or stylistic ones.
6. Write only `.github/reviews/YYYY-MM-DD/CODE-REVIEW.md` (never at the repository root) with the sections and finding format of the orchestrator, stable IDs (SEC-001, BUG-001, DB-001...) and an action plan ordered by impact with no purely aesthetic suggestions.

Avoid overengineering: recommend the simplest solution that addresses a demonstrated problem, reuse existing patterns, and do not propose speculative abstractions, extra layers or dependencies without a concrete need.

DO NOT modify application code.
