---
name: zone-rules-http
description: "Pointer to the HTTP/domain zone rules (canonical resource pattern, store/update, delete/restore, active flag, form requests, policies, queries, transformers). Load when editing app/Http/**, app/Policies/**, app/Queries/**, app/Transformers/**, packages/*/src/Http/**, packages/*/src/Policies/**, packages/*/src/Queries/**, packages/*/src/Transformers/**."
---

# Zone Rules: HTTP / Domain

**Read `.github/instructions/http.instructions.md`** — it is the single source of truth for this zone and Copilot injects it automatically by `applyTo`. This skill exists only so the non-Copilot clients can reach the same rules by loading a skill; do not restate or copy its content here, or the two will drift.

Invariants: `openspec/specs/basics-crud-base/spec.md`, `openspec/specs/basics-trash-archived/spec.md`, `openspec/specs/entities-domain/spec.md`, `openspec/specs/projects-domain/spec.md`.
