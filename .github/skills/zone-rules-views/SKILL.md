---
name: zone-rules-views
description: "View/i18n/UX rules: canonical forms, lists, Flux components, assets, headers, toolbars, empty states, Basque copy, accessibility. Load when editing resources/views/**, lang/**, resources/lang/**, resources/js/**."
---

# Zone Rules: Views / i18n / UX

## Before Creating/Changing Views
Apply `.github/skills/ux-design/SKILL.md` before touching views, Blade components, styles, or UI scripts.

## Canonical Form
- `resources/views/customers/form.blade.php` is the reference.
- Every resource form MUST wrap content in `<x-forms.tracked-resource>` (`resources/views/components/forms/tracked-resource.blade.php`):
  - Header, CSRF, `_method`, dirty tracking, double-submit guard
  - Cancel/submit footer with `{prefix}-cancel`/`{prefix}-submit` data-tests live only there
- Field markup stays local per resource but always follows: `flux:field` → `flux:label` (red `*` when required) → control with `data-test` → `flux:error`

## Canonical List
- `resources/views/customers/list.blade.php` + `resources/views/components/list/*`
- Table and pagination go through `<x-list.table>`
- Do not copy shared markup back into a view
- Keep existing `data-test` names; row actions pass the edit payload via `data-payload`

## Components
- Use Flux components where one exists; reuse `x-list.*` before creating new components.

## Assets
- Build: `docker compose run --rm --no-deps --entrypoint npm laravel13-npm <cmd>` (e.g., `run build`)
- After editing CSS/JS run the build; if UI does not refresh, ask user to start dev (`-p 5173:5173 ... run dev -- --host 0.0.0.0`)

## Page Structure
- One primary action with text, the H1 and the search field live in the header.
- Breadcrumbs with real hierarchy (Customers › Project › Epic). No "Dashboard" in root lists.
- Toolbar: search left (shortcut `/`), filters right.
- Tabs Active | Trash (n) inside each resource, not loose icons top-right.
- Sidebar: product brand, Basque texts, no starter kit links (Repository, Documentation). Distinct icons per section.

## Lists
- Columns: name (click opens edit), useful datum (child count, dates), actions right. No edit/delete stuck left.
- Actions: target ≥32px (44px touch), tooltip and `aria-label`. More than two → "⋯" menu.
- Blocked action (lock): tooltip with reason + link to children.
- Dates readable and localized. Never ISO (`2026-10-02`).
- Table headers sentence case, 12–13px, `text-zinc-600` (AA contrast).
- Pagination 15 default, with "n–m / total".
- Empty: message + create button. Loading: skeleton. Mobile: rows → cards.

## Forms & Drawers
- 1–2 fields → small modal. Drawer only for entities with relations or comments (epics).
- Autofocus first field. Enter saves, Esc closes, warn if unsaved changes.
- Fixed footer: Utzi / Gorde. Disabled button must explain why or stay active and warn.
- Validation on blur, specific message + how to fix.
- Visible focus: 2px brand ring (`focus-visible:ring-2 focus-visible:ring-brand-600 focus-visible:ring-offset-2`). Never remove.
- Dates: native picker well styled. Explain "end after start" rule, don't just block.
- Comments: initials, relative date, text field always visible at bottom.

## Feedback
- Toast (Flux) for success. Callout only for blocking errors.
- Same verb in button and toast ("Gorde" → "Gordeta").
- Trash name conflict: two clear options (restore existing / create new) + consequence of each.
- Dependency blocks: explain and link to blocking elements.

## Copy (Basque / Euskera)
- Active verbs, sentence case. No "Aceptar" nor "Enviar".
- Use keys from `lang/eu.json`. If adding text, add in all 4 locales (parity test exists). Default locale is `eu`.
- Basque is often longer: check nothing breaks on mobile.

## Accessibility & Motion
- AA contrast, `focus-visible`, lone icons with `aria-label`, don't rely on color alone.
- Motion only as response to action (open drawer, confirm), 150–200ms, respect `prefers-reduced-motion`. No decorative entrance animations.

## Project Limits
- Preserve all `data-test` (Dusk tests depend on them).
- Flux free: no Pro components. Missing = Alpine + Tailwind.
- No dependency changes or base folders without approval. Replacing select2/jQuery requires explicit approval.
- Blaze: don't fold components with global state (`$errors`, auth, session).
- Tailwind v4: tokens in `@theme` (`resources/css/app.css`), `gap-*` over margins between siblings, dark mode if screen already has it.
- After changing views: Vite build, Pint, tests and affected Dusk.

## Before Finishing Checklist
- [ ] One primary action with text, and H1 present.
- [ ] No destructive action without undo or proper confirmation.
- [ ] All keyboard operable, visible focus.
- [ ] Texts in 4 locales, sentence case.
- [ ] Tested on mobile and dark mode.
- [ ] `data-test` intact.