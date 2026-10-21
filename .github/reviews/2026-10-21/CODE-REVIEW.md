# Code Review: add-roles-permissions-system

**Date:** 2026-10-21  
**Change ID:** `add-roles-permissions-system`  
**Reviewer:** Full Review (spec audit)  
**Status:** Proposal phase — not yet implemented

---

## Executive Summary

The proposal introduces a dynamic RBAC system in `basics13` with roles, permissions, user-role assignment, and a user locale preference (`lang`). Management (CRUD) lives in `basics13`; enforcement happens in domain packages (`entities13`, `projects13`). The spec is well-structured and respects package boundaries overall. This review identifies **3 CRITICAL**, **5 HIGH**, **8 MEDIUM**, **4 LOW**, and **3 INFO** findings that must be addressed before implementation.

---

## Findings

### CRITICAL-001: CHECK constraint in basics13 references domain modules

**Category:** Architecture / Package Boundaries  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (line 47)  
**Problem:** The `permissions` table CHECK constraint includes `'clients', 'projects', 'entities'` which are domain package modules. `basics13` must remain domain-neutral per `specs/package-boundaries`.  
**Evidence:** Line 47: `CHECK module IN ('users', 'roles', 'permissions', 'clients', 'projects', 'entities')` with note "extensible via migration by domain packages". But the constraint is defined in basics13 migration.  
**Impact:** Violates package boundaries; basics13 cannot be installed standalone without domain modules; domain packages cannot extend CHECK constraint without dropping/recreating (MySQL limitation).  
**Recommendation:** Remove domain modules from basics13 CHECK constraint. Keep only `('users', 'roles', 'permissions')`. Domain packages should NOT extend via CHECK constraint (MySQL doesn't support ALTER TABLE ... MODIFY CHECK easily). Instead, validate module in application layer (FormRequest/Policy) or use a separate `modules` reference table.  
**Confidence:** HIGH

---

### CRITICAL-002: Wildcard `manage` permission logic undefined in HasRoles

**Category:** Business Logic / Security  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 75-94), `design.md` (line 102)  
**Problem:** Spec requires `hasPermissionTo('projects.edit')` to return true when user has `projects.manage`, but `HasRoles` concern methods don't define this logic. The wildcard `*.manage` for superadmin is also not implemented.  
**Evidence:** Spec lines 81-84: "Wildcard manage implica todas las acciones" scenario. Design line 102: `hasPermissionTo(string): bool` method exists but no wildcard handling described.  
**Impact:** Permission checks will not work as specified; superadmin won't have implicit access; each domain package would implement differently → inconsistency.  
**Recommendation:** Define explicit algorithm in `HasRoles::hasPermissionTo()`:
1. Check exact permission match
2. Check `{module}.manage` permission
3. Check `*.manage` permission (superadmin)
Add `hasModuleAccess(string $module)` that checks `{module}.manage` OR `*.manage`. Document in spec.  
**Confidence:** HIGH

---

### CRITICAL-003: UserPermissionsSummary couples basics13 to domain modules

**Category:** Architecture / Package Boundaries  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 333-357), `spec.md` (line 152)  
**Problem:** The User Permissions Summary view hardcodes columns for domain modules: `clients`, `projects`, `entities`, `entity_types`. basics13 cannot know about domain modules.  
**Evidence:** Design line 340: "One column per module: `users`, `roles`, `permissions`, `clients`, `projects`, `entities`, `entity_types`". Spec line 152: same list.  
**Impact:** basics13 view breaks when domain packages not installed; violates "basics13 domain-neutral" principle; domain packages cannot add columns without modifying basics13.  
**Recommendation:** Make modules dynamic: `UserPermissionsSummaryQuery` discovers modules from `Permission::distinct('module')->pluck('module')`. View iterates dynamic columns. Domain packages' permissions automatically appear when seeded.  
**Confidence:** HIGH

---

### HIGH-001: role_user FK to users table breaks custom user model referential integrity

**Category:** Database / Architecture  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 49-57), `spec.md` (lines 214-224)  
**Problem:** `role_user.user_id` FK references `users.id` (basics13 default table). When host uses custom user model (e.g., `App\Models\CustomUser` with table `custom_users`), FK points to wrong table. Application-level validation is insufficient for data integrity.  
**Evidence:** Design line 57: "FK references the default `users` table... referential integrity for custom user models is enforced at application level". Spec line 222: "FK `user_id` referencia `users.id` (tabla por defecto de basics13)".  
**Impact:** Orphaned role assignments if custom user deleted; no DB-level cascade; host must manually sync.  
**Recommendation:** Option A: Make `role_user` polymorphic (`model_type`, `model_id`) → true multi-model support. Option B: Document clearly that basics13 User model is required for roles; custom user models must extend basics13 User. Option C: Use `morphToMany` in `HasRoles` with `role_user` having `model_type`/`model_id` (requires migration change).  
**Confidence:** HIGH

---

### HIGH-002: syncRoles([]) lockout prevention logic undefined

**Category:** Business Logic / Security  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 141-144)  
**Problem:** "remover todos los roles del último usuario con acceso de admin" — how is "último usuario con acceso de admin" determined? No definition of "acceso de admin" (role? permission?); race condition possible between check and sync.  
**Evidence:** Spec line 142: scenario mentions "último usuario con acceso de admin" without definition.  
**Impact:** Unclear implementation; potential lockout of all admins; race condition in concurrent requests.  
**Recommendation:** Simplify: `syncRoles([])` always allowed (explicit admin action). Add separate "protect last superadmin" check in `RolePolicy::delete()` or via UI confirmation. Or define "admin access" as `hasPermissionTo('users.manage')` and check count before sync.  
**Confidence:** HIGH

---

### HIGH-003: hasPermissionTo cache behavior ambiguous for same-request changes

**Category:** Performance / Business Logic  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 184-189)  
**Problem:** Scenario says "en el mismo request puede retornar true (cacheado) o false (sin cache) — comportamiento documentado". This is undefined behavior — callers cannot rely on result.  
**Evidence:** Spec lines 186-187: explicit ambiguity allowed.  
**Impact:** Bugs in UI where permission check after role update returns stale data; inconsistent UX.  
**Recommendation:** Define clear contract: `hasPermissionTo()` uses `once()` cache per request. Provide `refreshPermissions()` or `loadPermissions()` to invalidate. After role/permission change, redirect (new request) or call `loadPermissions()`. Document: "Within same request, returns cached value; changes visible on next request."  
**Confidence:** HIGH

---

### HIGH-004: EntitySelectOptionsQuery creates cross-package dependency

**Category:** Architecture / Package Boundaries  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 380-388), `tasks.md` (lines 295-300)  
**Problem:** `EntitySelectOptionsQuery` lives in `entities13` but is used by `projects13` ProjectFormComponent. This creates `projects13` → `entities13` dependency, violating "domain → never domain".  
**Evidence:** Design line 387: "`projects13` usa `EntitySelectOptionsQuery` vía container/Livewire, sin importar modelos de `entities13`". Tasks line 295: query in `entities13`.  
**Impact:** Domain packages coupled; cannot install projects13 without entities13; circular dependency risk.  
**Recommendation:** Move selector queries to `entity-projects13` bridge (which can reference both). Or define interface in `basics13` (`SelectOptionsProvider`) that domain packages implement, bridge wires them.  
**Confidence:** HIGH

---

### HIGH-005: active flag on roles/permissions behavior undefined

**Category:** Business Logic  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 11-15, 46-49)  
**Problem:** Roles/permissions have `active` flag and partial unique index on `active=true AND deleted_at IS NULL`. But spec doesn't define: Does inactive role = users lose permissions? Can inactive permission be assigned to role? What happens to existing role-permission links when permission archived?  
**Evidence:** Spec requires `active` flag but no scenarios for inactive state behavior. Design shows `active()` scope but no usage in `HasRoles`.  
**Impact:** Inconsistent behavior; users may retain permissions from inactive roles; UI confusion.  
**Recommendation:** Add scenarios:
- Inactive role: `hasRole()` returns false, permissions not granted
- Inactive permission: not assignable to new roles, existing links ignored in `hasPermissionTo()`
- Update `HasRoles::hasPermission()` to filter `where('active', true)` on both roles and permissions.  
**Confidence:** HIGH

---

### MEDIUM-001: forceDelete action naming inconsistent

**Category:** Convention / Maintainability  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 204-212), `spec.md` (line 57)  
**Problem:** Actions: `view`, `create`, `edit`, `delete`, `archive`, `restore`, `forceDelete`, `manage`. `forceDelete` uses camelCase; others single word.  
**Evidence:** Design line 211: `forceDelete` vs line 204-210: all lowercase.  
**Impact:** Inconsistent permission slugs (`projects.forceDelete` vs `projects.delete`); typo-prone; breaks convention.  
**Recommendation:** Use `force_delete` (snake_case) or `forceDelete` everywhere consistently. Prefer snake_case per Laravel convention (e.g., `forceDelete` method but `force_delete` in routes/policies).  
**Confidence:** MEDIUM

---

### MEDIUM-002: assigned_by in role_user pivot adds complexity without clear requirement

**Category:** Maintainability / YAGNI  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 49-55), `spec.md` (line 125)  
**Problem:** `role_user` pivot includes `assigned_by` FK and `assigned_at`. No scenario requires audit trail; `getRoleAssignments()` returns it but no UI/feature uses it.  
**Evidence:** Design line 55: `assigned_by` column. Spec line 125: "assigned_at y assigned_by (auditoría)". No scenario tests it.  
**Impact:** Extra column, FK, complexity; YAGNI.  
**Recommendation:** Remove `assigned_by` and `assigned_at` from pivot. Keep simple `role_id`, `user_id`. Add audit later if needed.  
**Confidence:** MEDIUM

---

### MEDIUM-003: Language class uses emoji flags — rendering inconsistent

**Category:** Frontend / UX  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 128-133)  
**Problem:** Flags are emoji (`🇪🇺`, `🇪🇸`, `🇫🇷`, `🇬🇧`). Render differently across OS/browsers; not accessible; not suitable for production UI.  
**Evidence:** Design lines 129-132: emoji flags in `SUPPORTED` array.  
**Impact:** Poor UX on Windows/Linux; accessibility issues (screen readers).  
**Recommendation:** Use ISO 3166-1 alpha-2 codes (`eu`→`ES`, `es`→`ES`, `fr`→`FR`, `en`→`GB`) with SVG flag assets or CSS flag icons. Or remove flags from class; let views handle presentation.  
**Confidence:** MEDIUM

---

### MEDIUM-004: Language class should be PHP Enum (PHP 8.4)

**Category:** Code Quality / Modern PHP  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 120-185), `tasks.md` (line 30)  
**Problem:** `Language` class with `const SUPPORTED` array. PHP 8.4 supports enums with methods — better type safety, IDE support, immutability.  
**Evidence:** Tasks line 30: "Create `Language` class... (or Enum)". Design shows class implementation.  
**Impact:** Missed opportunity for modern PHP; stringly-typed locale codes.  
**Recommendation:** Use `enum Language: string { case EU = 'eu'; case ES = 'es'; case FR = 'fr'; case EN = 'en'; }` with methods `nativeName()`, `flag()`, `isRtl()`, `isValid()`.  
**Confidence:** MEDIUM

---

### MEDIUM-005: No API endpoints for role/permission management

**Category:** API / Completeness  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 242-263)  
**Problem:** Routes only define web (Blade/Flux) endpoints. No API resources for SPA/mobile/external integration.  
**Evidence:** Design lines 245-263: only GET/POST/PUT/DELETE for web controllers.  
**Impact:** Cannot build headless admin or integrate with external systems.  
**Recommendation:** Add API resource routes (`Route::apiResource('roles', RoleApiController::class)`) returning JSON via API Resources. Or document that API is out of scope for this PR.  
**Confidence:** MEDIUM

---

### MEDIUM-006: UserRequest validation for lang not specified

**Category:** Completeness  
**File:** `openspec/changes/add-roles-permissions-system/design.md` (line 116), `tasks.md` (line 94)  
**Problem:** `UserRequest` should validate `lang` but not listed in Phase 2 Form Requests.  
**Evidence:** Design line 116: "Validation: In `UserRequest`, `lang` ∈ `Language::codes()`". Tasks Phase 2 doesn't include `UserRequest`.  
**Impact:** Missing validation; invalid locale could be stored.  
**Recommendation:** Add `UserRequest` to Phase 2 tasks with `lang` validation rule.  
**Confidence:** MEDIUM

---

### MEDIUM-007: Permission slug collision between basics13 and domain packages

**Category:** Database / Architecture  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (line 209-210)  
**Problem:** Domain packages add permissions for same modules (`projects`, `entities`) but basics13 has no mechanism to prevent slug collisions or coordinate seeding order.  
**Evidence:** Spec line 210: "se añaden permisos para módulos `clients`, `projects`, `entities`, `entity_types`". No coordination mechanism.  
**Impact:** Duplicate slug errors on seeder run; race conditions.  
**Recommendation:** Use seeder `firstOrCreate` with `slug` + `module` unique. Document seeding order: basics13 first, then domain packages. Or use `module` prefix in all permissions (already done).  
**Confidence:** MEDIUM

---

### MEDIUM-008: hasModuleAccess vs hasPermissionTo('module.manage') duplication

**Category:** API Design / Consistency  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 75-94), `design.md` (line 103)  
**Problem:** `hasModuleAccess('projects')` checks `projects.manage` OR `*.manage`. `hasPermissionTo('projects.manage')` checks exact permission. Two methods for similar purpose — confusion.  
**Evidence:** Spec lines 78-79: `hasModuleAccess` defined. Design line 103: method listed.  
**Impact:** Developers unsure which to use; inconsistent code.  
**Recommendation:** Deprecate `hasModuleAccess`; use `hasPermissionTo('projects.manage')` for module access check. Or make `hasModuleAccess` explicit alias with clear docs.  
**Confidence:** MEDIUM

---

### LOW-001: Role/permission form slug auto-generation not specified

**Category:** UX / Completeness  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 112-115)  
**Problem:** Form scenario says "slug (auto-generado desde nombre, editable, único)" but no detail on algorithm (Str::slug? transliteration? uniqueness handling?).  
**Evidence:** Spec line 113: "slug (auto-generado desde nombre, editable, único)".  
**Impact:** Implementation variance; potential UX issues if slug conflicts.  
**Recommendation:** Specify: `Str::slug($name)` + uniqueness check (append `-1`, `-2`); editable by user; validate on blur via AJAX.  
**Confidence:** LOW

---

### LOW-002: No soft-delete cascade for role_permission pivot

**Category:** Database / Data Integrity  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 59-63)  
**Problem:** `permission_role` pivot has FK CASCADE on both sides. When role soft-deleted, pivot rows remain (FK not triggered). Orphaned pivot rows accumulate.  
**Evidence:** Design line 62-63: FK CASCADE on `permission_role`. Soft deletes don't trigger FK.  
**Impact:** Orphaned pivot data; `hasPermission()` may include permissions from trashed roles if not filtered.  
**Recommendation:** Add `deleted_at` to pivot (full soft delete) or use model events to detach on delete. Document in spec.  
**Confidence:** LOW

---

### LOW-003: Superadmin wildcard `*.manage` — pattern not defined

**Category:** Completeness  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 86-89)  
**Problem:** Scenario says superadmin has `*.manage` but no such permission exists in seeders (only `module.manage`). How is `*.manage` represented?  
**Evidence:** Spec line 88: "usuario tiene rol `superadmin` con permiso `*.manage`". Design seeders show `*.manage` notation but not actual permission.  
**Impact:** Implementation ambiguity; seeder won't create `*.manage`.  
**Recommendation:** Either create actual permission `*.manage` (module `*`, action `manage`) or implement wildcard logic in `HasRoles` without DB permission. Document chosen approach.  
**Confidence:** LOW

---

### LOW-004: Missing scenario for role/permission restore after trash

**Category:** Completeness  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md` (lines 96-103)  
**Problem:** CRUD requirement includes ArchivedController and TrashController but no scenarios for restore from trash or archive.  
**Evidence:** Spec lines 96-103: only list/create/edit scenarios for roles/permissions.  
**Impact:** Restore functionality untested/unspecified.  
**Recommendation:** Add scenarios for restore from trash, restore from archive.  
**Confidence:** LOW

---

### INFO-001: Language class native names duplicate translation keys

**Category:** i18n / Duplication  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 128-133, 145-148)  
**Problem:** `Language::nativeNames()` returns hardcoded native names. Translation files also need locale names for UI. Duplication of data.  
**Evidence:** Design line 147: "Locale native names come from `Language::nativeNames()` (not translation files)".  
**Impact:** Two sources of truth for locale names; maintenance burden.  
**Recommendation:** Keep native names in Language class for programmatic use; translation keys for UI strings (e.g., "Language", "Preferred language"). Document separation.  
**Confidence:** LOW

---

### INFO-002: No mention of spatie/laravel-permission compatibility

**Category:** Dependencies / Ecosystem  
**File:** All spec files  
**Problem:** Project may have existing spatie/laravel-permission usage or plans. No mention of migration path or coexistence.  
**Evidence:** No reference to spatie in any spec file.  
**Impact:** Future migration friction; potential confusion.  
**Recommendation:** Add note: "This replaces spatie/laravel-permission; migration guide in separate doc" or "Compatible with spatie via adapter".  
**Confidence:** LOW

---

### INFO-003: Config file basics13.php only has route_prefix

**Category:** Configuration / Completeness  
**File:** `openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md` (lines 422-427)  
**Problem:** Config file only defines `route_prefix`. Could include `supported_locales`, `default_role`, `permission_cache_ttl`, etc. for future extensibility.  
**Evidence:** Design lines 422-427: minimal config.  
**Impact:** Future changes require config publish/merge.  
**Recommendation:** Add placeholder keys with defaults for extensibility.  
**Confidence:** LOW

---

## Action Plan (Ordered by Impact)

| Priority | Finding | Action |
|----------|---------|--------|
| 1 | CRITICAL-001 | Remove domain modules from basics13 CHECK constraint; use application-layer validation |
| 2 | CRITICAL-002 | Implement wildcard `manage` logic in `HasRoles::hasPermissionTo()` |
| 3 | CRITICAL-003 | Make UserPermissionsSummary modules dynamic (discover from Permission table) |
| 4 | HIGH-001 | Decide on polymorphic `role_user` vs documented basics13 User requirement |
| 5 | HIGH-002 | Simplify `syncRoles([])` lockout or define "admin access" precisely |
| 6 | HIGH-003 | Define clear cache contract: cached per-request, refresh via `loadPermissions()` |
| 7 | HIGH-004 | Move `EntitySelectOptionsQuery` to `entity-projects13` bridge or define interface |
| 8 | HIGH-005 | Define `active` flag behavior: filter in `HasRoles`, prevent assignment of inactive |
| 9 | MEDIUM-001 | Standardize action names: `force_delete` (snake_case) |
| 10 | MEDIUM-002 | Remove `assigned_by`/`assigned_at` from `role_user` pivot (YAGNI) |
| 11 | MEDIUM-003 | Replace emoji flags with ISO codes + SVG assets |
| 12 | MEDIUM-004 | Convert `Language` class to PHP Enum |
| 13 | MEDIUM-005 | Add API resource routes or document out of scope |
| 14 | MEDIUM-006 | Add `UserRequest` to Phase 2 tasks with `lang` validation |
| 15 | MEDIUM-007 | Document seeder coordination; use `firstOrCreate` |
| 16 | MEDIUM-008 | Unify `hasModuleAccess` / `hasPermissionTo('module.manage')` |
| 17 | LOW-001 | Specify slug auto-generation algorithm |
| 18 | LOW-002 | Handle pivot soft-delete cascade |
| 19 | LOW-003 | Define `*.manage` representation (DB permission or code logic) |
| 20 | LOW-004 | Add restore scenarios for roles/permissions |
| 21 | INFO-001 | Document Language class vs translation files separation |
| 22 | INFO-002 | Add spatie/laravel-permission compatibility note |
| 23 | INFO-003 | Extend config file with placeholder keys |

---

## Architecture Compliance Check

| Rule | Status | Notes |
|------|--------|-------|
| Package boundaries (bridge → domains) | ⚠️ PARTIAL | Violated by CRITICAL-001, CRITICAL-003, HIGH-004 |
| No domain → domain deps | ❌ FAIL | HIGH-004: projects13 → entities13 via EntitySelectOptionsQuery |
| No domain → basics13 model deps | ✅ PASS | Domain packages use `$user->hasPermissionTo()` only |
| `basics13` domain-neutral | ❌ FAIL | CRITICAL-001, CRITICAL-003: domain modules in basics13 |
| Specs = invariants not in code | ✅ PASS | New capability spec created |
| One atomic change = one PR | ✅ PASS | Single change ID |

---

## Recommendation

**Do not implement until CRITICAL and HIGH findings are resolved.** The three CRITICAL issues (package boundaries, wildcard logic, dynamic summary) are architectural blockers. The five HIGH issues (FK strategy, lockout, cache contract, cross-domain query, active flag) are significant behavioral gaps.

Once resolved, validate with `openspec validate --strict` and proceed to implementation following `tasks.md`.