---
name: zone-rules-views
description: "Pointer to the view/i18n/UX zone rules (canonical forms, lists, Flux components, assets, headers, Basque copy, accessibility). Load when editing resources/views/**, lang/**, resources/lang/**, resources/js/**."
---

# Zone Rules: Views / i18n / UX

Two files own this zone. Read them in this order; this skill holds no rules of its own, so it cannot drift from them.

1. **`.github/instructions/views.instructions.md`** — the always-on rules for this zone (canonical form and list, `<x-list.*>`/`<x-forms.*>`, asset commands, i18n parity). Copilot injects it automatically by `applyTo`.
2. **`.github/skills/ux-design/SKILL.md`** — the design system applied before creating or changing any view, Blade component or style (page structure, lists, forms and drawers, feedback, Basque copy, accessibility, Tailwind v4 limits and the closing checklist).

Invariants: `openspec/specs/basics-shared-views/spec.md`, `openspec/specs/customers-domain/spec.md`, `openspec/specs/projects-domain/spec.md`, `openspec/specs/customer-projects-bridge/spec.md`.