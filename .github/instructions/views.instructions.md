---
applyTo: "**/resources/views/**,**/lang/**,**/resources/lang/**,**/resources/js/**"
---

# View / i18n / UX rules

- Before creating or changing a view, Blade component, style or UI script, apply `.github/skills/ux-design/SKILL.md`.
- Canonical form: `entities13/resources/views/entities/form.blade.php`. Every resource form MUST wrap its content in `<x-forms.tracked-resource>` (shipped by `basics13`): header, CSRF, `_method`, dirty tracking, double-submit guard and the cancel/submit footer with `{prefix}-cancel`/`{prefix}-submit` data-tests live only there. Field markup stays local per resource but always follows the same pattern: `flux:field` → `flux:label` (red `*` when required) → control with `data-test` → `flux:error`.
- Canonical list: `entities13/resources/views/entities/list.blade.php` plus the shared `<x-list.*>` components in `basics13`; table and pagination go through `<x-list.table>`. Do not copy shared markup back into a view. Keep existing `data-test` names; row actions pass the edit payload via `data-payload`.
- Use Flux components where one exists; reuse `x-list.*` before creating new components.
- Assets: `docker compose run --rm --no-deps --entrypoint npm laravel13-npm <cmd>` (e.g. `run build`). After editing CSS/JS run the build; if the UI does not refresh, ask the user to start dev (`-p 5173:5173 ... run dev -- --host 0.0.0.0`).
- After changing a view, Blade component, UI script or style, open each affected screen in the browser and inspect the console for errors and warnings, including Alpine expression errors and uncaught page errors. Exercise the changed interaction/state as well as initial rendering; fix regressions caused by the change before finishing, and report any unrelated existing messages separately. If a screen redirects to login, use the development account from `packages/basics13/database/seeders/UserSeeder.php` (called by `BasicsDevelopSeeder`) as described in the `docker-commands` skill.
- One primary action with text, the H1 and the search field live in the header.
- Nothing destructive without an undo or a proper confirmation.
- Any new user-facing string goes in the owning package's `resources/lang/*.json` for all 4 locales (en/es/eu/fr, same key and `:placeholders`), in sentence case; a test checks parity. Default locale is `eu`.
- A `maxlength` attribute never carries a literal number either: it renders `config('validation.max_length.string')` or `config('validation.max_length.longtext')`, the same value the FormRequest validates, so the browser cannot stop the input the server would accept.

> Invariantes de esta zona: `openspec/specs/basics-shared-views/spec.md`, `openspec/specs/entities-domain/spec.md`, `openspec/specs/projects-domain/spec.md`, `openspec/specs/entity-projects-bridge/spec.md`.
