---
description: "Use when changing packages, dependencies or cross-domain integrations. Keep domain packages independent and put entity/project coupling in the existing bridge."
applyTo: "packages/**"
---

# Package boundaries

Follow [AGENTS.md](../../AGENTS.md#package-boundaries-mandatory) and read
[ARCHITECTURE.md](../docs/architecture/ARCHITECTURE.md) before changing a package.

- Allowed: `entity-projects13` -> `entities13` and `projects13`; domain packages -> existing shared infrastructure in `basics13`.
- Forbidden: `entities13` -> `projects13`, `projects13` -> `entities13`, either domain -> `entity-projects13`, or cross-domain business logic moved into `basics13`.
- Put anything requiring knowledge of both entities and projects in `entity-projects13`, including schema, validation, queries, relationships, deletion guards, selectors and UI.
- Base packages use their own domain-neutral contracts, configuration and standalone defaults. Reuse these extension points; the bridge registers the concrete implementations. See [EntityProjectsIntegration](../../packages/entity-projects13/src/Integrations/EntityProjectsIntegration.php) and its [ServiceProvider](../../packages/entity-projects13/src/ServiceProvider.php).
- Do not disguise dependencies as strings, container lookups, table names, route/view names, domain-specific payload keys or UI conditions. The rule also covers assets, translations, factories, seeders and tests, not only PHP imports or Composer entries.
- Keep standalone tests in the base packages; put cross-domain tests in the bridge or host integration suite. Preserve behavior when the bridge is absent.
- If an extension point is insufficient, ask before adding an abstraction or dependency. Do not copy existing coupling as a precedent.

> Invariantes de esta zona: `openspec/specs/package-boundaries/spec.md`.
