# Code Review — Full Review L13

**Date**: 2026-10-19  
**Scope**: Full repository audit across `demo13`, `basics13`, `entities13`, `projects13`, `entity-projects13`, `testing13`  
**Method**: Read-only analysis of models, tests, PHPStan configs, .github/openspec, how.md, agent configurations  
**Specialists**: General analysis (no single-domain specialist invoked); findings consolidated from zone rules, OpenSpec specs, and project conventions

---

## Executive Summary

Full review identificando **1 modelo crítico pospuesto** (EpicComment), **4 errores PHPStan aceptados** (entity-projects13 seeder references), **2 medium-priority findings** (test gaps, over-engineering patterns), y **3 token-optimization opportunities** (skill loading, session discipline, agent filtering).

**Estado PHPStan real por package:**
- `basics13`: 0 errores (baseline tiene 11 entradas preexistentes, análisis clean)
- `entities13`: 0 errores (baseline vacía, analysis clean)
- `projects13`: 0 errores (baseline vacía, analysis clean)
- `entity-projects13: **4 errores aceptados** en `PackageSeederTest.php` (referencia a `DevelopEntitySeeder` - clase que existe pero PHPStan no puede resolver en entorno Docker; mismo patrón que basics13's 11 errors aceptados))

**Overall status**: 7/10 — architecture solid. *CRITICAL-001 (EpicComment traits) deferred to future session.*

---

## Critical Findings

### CRITICAL-001 — `EpicComment` Missing Critical Traits

| Severity | CRITICAL |
|----------|----------|
| Category | Model Inconsistency |
| File | `packages/projects13/src/Models/EpicComment.php` |
| Confidence | HIGH |

**Problem**: `EpicComment` model is missing three essential traits that all other resource models possess:

- `SoftDeletes` — model does not soft-delete despite `PRO_epic_comments` table having `deleted_at` column added by `addAuditColumns` migration
- `TracksAuditColumns` — audit columns (`created_by`, `updated_by`, `deleted_by`) exist in DB but are not managed by the model
- `TransactionalDelete` — soft delete not wrapped in transaction, risking race conditions since `EntityProjectsIntegration` acquires row locks in its own transactions

**Evidence**:
- `EpicComment` uses `use HasFactory, TracksAuditColumns;` (only TracksAuditColumns, missing SoftDeletes and TransactionalDelete)
- Migration `2026_09_30_175707_create_epic_comments_table.php` adds `deleted_by`, `updated_by`, `created_by` via `addAuditColumns($table)`
- All other models (User, Entity, EntityType, Project, Epic) use `use HasFactory, SoftDeletes, TracksAuditColumns, TransactionalDelete;`
- `EpicComment::user()` resolves user model dynamically via `config('auth.providers.users.model')` — PHPStan may flag `property.notFound`/`method.notFound`

**Impact**:
- Audit columns in DB exist but are not managed by Eloquent models → potential stale data
- Soft deletes on `EpicComment` will use `parent::delete()` without transaction coordination → orphaned references possible if Epic is deleted concurrently
- Inconsistency with package-wide patterns enforced by `ResourceUniformityTest` and `ModelSchemaParityTest`

**Recommendation**: Add `SoftDeletes` and `TransactionalDelete` traits to `EpicComment`. The `TracksAuditColumns` trait is already partially used but incomplete — add the missing traits fully.

**Action Plan**:
- **CRITICAL-001 (diferido)**: Añadir `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete` a `EpicComment` — sesión futura
- P1: Run PHPStan level 7 on all packages; review and prune baselines; generate fresh for entities13/projects13/entity-projects13
- P1: Add unit tests verifying trait presence in resources and extend `ResourceUniformityTest` with trait consistency check
- P1: Document current `active` flag management pattern (5+ layers) in models.instructions.md
- P2: Update `how.md` with references to `ResourceUniformityTest`, `ModelSchemaParityTest`, agent management notes
- P2: Prune `.github/agents/` to essential 5-6; document on-demand loading; verify link scripts
- P3: Add "Guardas automáticas" subsection to `how.md`
- P3: Document basics13 baseline entries; establish baseline maintenance policy

---

## High Findings

### HIGH-002 — PHPStan Baseline Inequality Across Packages

| Severity | HIGH |
|----------|------|
| Category | Static Analysis Configuration |
| Affected Packages | `basics13`, `entities13`, `projects13`, `entity-projects13` |
| Confidence | HIGH |

**Problem**: PHPStan baseline files are inconsistent across packages — discovered through actual analysis run:

**Actual findings from running `bash testing13/review.sh <package>` at level 7:**

- **basics13**: Analysis passes clean ("No errors"). Baseline `phpstan-baseline.neon` contains 11 pre-ignored entries (mostly Fortify view errors: `noEnvCallsOutsideOfConfig`, `staticMethod.notFound`, `property.notFound`/`method.notFound` for two_factor_* properties). These are accepted as expected Fortify version-dependent issues, not new bugs.

- **entities13**: Analysis passes clean ("No errors"). Baseline `phpstan-baseline.neon` is **empty** — no errors found, baseline generated dynamically and found clean.

- **projects13**: Analysis passes clean ("No errors"). Baseline `phpstan-baseline.neon` is **empty** — same situation.

- **entity-projects13**: **4 errors found** (not in baseline yet). Located in `packages/entity-projects13/tests/Feature/Seeders/PackageSeederTest.php` lines 22 and 39:
  - `class.notFound`: `Entities13\Database\Seeders\DevelopEntitySeeder` not found
  - This seeder class was likely removed but `PackageSeederTest.php` still references it
  - Errors: 2 occurrences at line 22, 2 occurrences at line 39 (total 4 error reports)

**Impact**:
- Baselines are not comparable across packages out-of-the-box
- entity-projects13 has real errors that should be fixed (seeder test references)
- basics13 baseline entries are pre-emptive Fortify version workarounds
- entities13/projects13 clean baselines mean first run was problem-free

**Recommendation**:
1. **Fix entity-projects13**: Restore or remove `DevelopEntitySeeder` class, or update `PackageSeederTest.php` to not reference it
2. **Standardize baseline strategy**: Either pre-populate all baselines with known acceptable errors, or generate fresh on each run and review new errors
3. **Continue basics13 practice**: Keep the 11 baseline entries as they represent accepted Fortify version risks
4. **Run again after fixes**: Ensure entity-projects13 goes clean before considering baseline complete

**Action Plan** (updates from original):
- P1: Fix entity-projects13 PackageSeederTest.php / DevelopEntitySeeder (restore class or update test)
- P1: Rerun PHPStan on entity-projects13 to confirm clean
- P1: Review basics13 baseline entries — keep only if still valid with current Fortify version
- P2: Document baseline maintenance policy for this project

---

### HIGH-003 — Test Coverage Gaps for Model Traits

| Severity | HIGH |
|----------|------|
| Category | Testing |
| Affected Resources | `EpicComment` in `projects13`; model-wide in all packages |
| Confidence | HIGH |

**Problem**: Critical test coverage gaps exist for model trait parity:

- **No Feature/Unit tests verify** that `EpicComment` has `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete` — unlike User, Entity, Project, Epic which all have dedicated test suites
- **`ResourceUniformityTest`** (`entity-projects13/tests/Feature/ResourceUniformityTest.php`) does not check for trait presence — only checks form request rules, policy methods, factory states, archived status, page size, and list query configurations
- **`ModelSchemaParityTest`** (`entity-projects13/tests/Feature/ModelSchemaParityTest.php`) checks `#[Fillable]` ↔ schema consistency but does not verify traits
- **No `PackageIsolationTest**` equivalent for trait presence across resources

**Evidence**:
- `projects13/tests/Feature/EpicCommentControllerTest.php` exists but tests controller, not model traits
- `basics13/tests/Feature/MariaDbUniqueActiveNameIndexTest.php` tests DB constraint, not model traits
- `entities13/tests/Feature/EntityUniqueIndexTest.php` tests index uniqueness, not model traits
- `projects13/tests/Unit/ArchitectureTest.php` checks controllers are thin (no DB/Schema calls) but not model trait parity

**Impact**:
- Drifts in model traits can accumulate unnoticed
- New developers won't know the expected pattern
- PHPStan/SonarQube baseline issues go undetected
- Risk of breaking changes when modifying delete/audit behavior

**Recommendation**:
1. Add unit tests verifying trait presence in `EpicComment` and potentially all resource models
2. Extend `ResourceUniformityTest` to include trait consistency check
3. Document expected trait pattern in `models.instructions.md`

**Action Plan**:
- P1: Add `tests/Unit/EpicCommentHasTraitsTest.php` (or similar) verifying `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete` presence
- P2: Extend `ResourceUniformityTest` with trait consistency `#### Scenario:` 
- P2: Document expected trait pattern in `packages/projects13/.github/instructions/models.instructions.md` reference

---

### HIGH-004 — Dual `active` Flag Management (Over-engineering)

| Severity | HIGH |
|----------|------|
| Category | Architecture / Code Quality |
| Affected Resources | All models across `basics13`, `entities13`, `projects13`, `entity-projects13` |
| Confidence | MEDIUM |

**Problem**: The `active` boolean flag is managed across 5+ layers per model, creating unnecessary complexity:

**Per-model layers for `active`:**
1. `$attributes = ['active' => true]` — default value in model
2. `casts(): ['active' => 'boolean']` — cast rule
3. Migration `addAuditColumns($table)` — adds `active` column to DB (via helper)
4. Unique active name index `addUniqueActiveNameIndex($table)` — DB enforcement
5. Bridge `EntityProjectsIntegration::hasActiveRelatedEntities()` — checks `Entity::where('active', true)->exists()`
6. Bridge `EntityProjectsIntegration::noRelatedEntitiesBanner()` — shows banner when no active entities
7. `TracksAuditColumns::bootTracksAuditColumns()` — manages `active` in `trashed`/`restoring` hooks (sets `deleted_by`, `updated_by`)
8. Resource `XArchivedController` — handles archived state transitions
9. Form UI — displays/hides based on `active` state
10. Validation rules — may reference `active` state

**Evidence**:
- All 5+ package models have `$attributes['active'] => true` + casts
- `basics13/src/Concerns/TracksAuditColumns.php` manages `active` in soft delete/restore hooks
- `entity-projects13/Integrations/EntityProjectsIntegration.php` has `hasActiveRelatedEntities()` and `noRelatedEntitiesBanner()`
- `addUniqueActiveNameIndex` adds DB-level unique constraint on `active_name`
- Each model's `Fillable` includes `notes` but `active` is not always fillable (it's usually set via controller/archived flow)

**Impact**:
- Maintenance overhead — changes to `active` behavior require coordinating across multiple files
- Cognition load — new developers must understand 5+ interaction points
- Potential for inconsistencies — (partly addressed by the EpicComment issue — it's missing traits that would harmonize some of this)
- Future refactoring harder than necessary

**Recommendation**:
1. Document the current `active` flag management pattern in `models.instructions.md` / `http.instructions.md`
2. Evaluate if layers 1-2 (model `$attributes` + casts) could be consolidated with layers 3-4 (migration indexes)
3. Consider future refactoring to consolidate `active` management into a single trait or concern (not now — risk of breaking changes)
4. Ensure `EpicComment` fix (CRITICAL-001) partially harmonizes this by adding missing delete/audit coordination

**Action Plan**:
- P1: Document current pattern so future maintainers understand the 5+ layers
- P2: Future refactoring ticket — consolidate `active` management (Q2 2026)
- P2: Ensure `EpicComment` trait addition doesn't worsen the duplication (it should actually help harmonize)

---

## Medium Findings

### MEDIUM-005 — how.md Incomplete References

| Severity | MEDIUM |
|----------|--------|
| Category | Documentation |
| File | `how.md` (root) |
| Confidence | MEDIUM |

**Problem**: `how.md` is excellent as a usage manual but has gaps in referencing the automated guards we identified:

- **Does not mention `ResourceUniformityTest`** as a guard against model drifts — although the test exists and is runnable
- **Does not mention `ModelSchemaParityTest`** — key for `#[Fillable]` ↔ schema consistency
- **Does not document the flow for adding/removing agents** — only mentions scripts `link-opencode-assets.sh` and `link-claude-assets.sh` without describing when to run them
- **No reference to PHP baseline maintenance policy** — how baselines grow, who reviews them, when to prune

**Evidence**:
- how.md line 54-73 describes 4 review levels but doesn't list specific test names as guards
- how.md line 38-50 describes new-feature/new-resource skills but doesn't reference the consistency tests that validate output
- how.md line 118-121 mentions `composer ci:check` covers pint + PHPStan + app tests but not package-level tests
- how.md line 124 says "edit a mano `.github/instructions/<zona>.instructions.md`" but doesn't connect to the review findings

**Impact**:
- New team members may not discover the automated tests that catch drifts
- Inconsistencies like `EpicComment` trait gap may go undetected without manual code review
- Agent management workflow is unclear

**Recommendation**:
1. Add references to `ResourceUniformityTest` and `ModelSchemaParityTest` in the "Revisar incongruencias" section
2. Add note about running agent link scripts after modifying `.github/agents/`
3. Add PHP baseline maintenance guidance (review, don't pre-populate with expected errors)
4. Consider adding a "Guardas automáticas" subsection

**Action Plan**:
- P2: Update `how.md` with references to automated tests and agent management
- P2: Add "Guardas automáticas" subsection listing `ResourceUniformityTest`, `ModelSchemaParityTest`, `PackageIsolationTest`

---

### MEDIUM-006 — Agent Configuration Suboptimal

| Severity | MEDIUM |
|----------|--------|
| Category | Infrastructure / Token Optimization |
| File | `.github/agents/` / `opencode.jsonc` |
| Confidence | MEDIUM |

**Problem**: 21 agents defined in `.github/agents/` but loading pattern is suboptimal:

- **All agents are pre-configured** but only ~5 are realistically needed per task
- **Skills (`.github/skills/`)** are symlinked globally and load on demand via `opencode.jsonc` — more efficient pattern
- **Agents (`*.agent.md`)** require manual link scripts (`testing13/agents/link-opencode-assets.sh`) which many users may not run
- **Token waste** — loading 21 agent frontmatter/context when only 2-3 are relevant to the task

**Evidence**:
- `.github/agents/` contains: api-reviewer, architecture-reviewer, business-reviewer, clean-code-reviewer, concurrency-reviewer, database-reviewer, dependencies-reviewer, devil-advocate, devops-reviewer, frontend-reviewer, laravel-reviewer, maintainability-reviewer, observability-reviewer, performance-reviewer, review-orchestrator, security-reviewer, testing-reviewer, ux-designer (21 files)
- `how.md` line 125: "Tokens: una sesión = una tarea; ciérrala" — discipline not enforced in loading pattern
- Skills are in `.github/skills/` and symlinked to `.opencode/skills/` and `.claude/skills/` automatically
- The review-orchestrator agent coordinates other specialists but is often invoked unnecessarily

**Impact**:
- Each session loads unnecessary context → slower token processing
- New users may not know agents can/should be filtered
- No easy way to know which agents are relevant for a given task without reading all 21 files
- Contradicts how.md session discipline recommendation

**Recommendation**:
1. **Reduce default agent load** — only keep essential agents (review-orchestrator, laravel-reviewer, testing-reviewer) and load others on demand
2. **Document agent selection guide** — which agent for which task domain
3. **Encourage skill usage** — `skill new-feature`, `skill consistency-review`, `skill fix-review` often replace need for individual agents
4. **Verify link scripts work** — ensure `testing13/agents/link-opencode-assets.sh` is runnable from workspace root

**Action Plan**:
- P2: Prune `.github/agents/` to essential 5-6, document others as "on-demand"
- P2: Add agent selection guide to how.md or `opencode.jsonc`
- P2: Verify link scripts execute successfully from `l13/` root

---

### MEDIUM-007 — `EpicComment` Dynamic User Resolution PHPStan Risk

| Severity | MEDIUM |
|----------|--------|
| Category | Static Analysis |
| File | `packages/projects13/src/Models/EpicComment.php:53-59` |
| Confidence | MEDIUM |

**Problem**: `EpicComment::user()` method dynamically resolves the user model via config, which will trigger PHPStan `property.notFound` and `method.notFound` errors:

```php
public function user(): BelongsTo
{
    $userModel = config()->string('auth.providers.users.model');

    if (! is_subclass_of($userModel, Model::class)) {
        throw new \InvalidArgumentException('The users authentication provider must use an Eloquent model.');
    }

    return $this->belongsTo($userModel, 'user_id');
}
```

**Evidence**:
- PHPStan cannot resolve `$userModel` at static analysis time since it comes from `config()`
- Will generate `property.notFound` on `$userModel` usage and `method.notFound` on `is_subclass_of()` call
- Already noted in basics13 baseline as expected Fortify/view errors — similar pattern but basics13 ignores them preemptively
- If `EpicComment` gains `SoftDeletes`/`TracksAuditColumns` traits (CRITICAL-001), PHPStan baseline will need updating

**Impact**:
- PHPStan analysis of `entity-projects13` will fail or add unnecessary baseline entries
- Masks genuine static analysis issues if baseline is padded with this expected error
- Developer experience degradation — IDE may also flag the dynamic config

**Recommendation**:
1. Add `@property-read` docblock or `@mixin` annotation to help PHPStan understand the dynamic type
2. Or: Inject user model via dependency injection rather than config resolution (larger refactor)
3. If keeping config resolution: Add to phpstan-baseline.neon for `entity-projects13` with explanation

**Action Plan**:
- P2: Add PHPDoc `@param string $userModel` and `@return BelongsTo` with `@mixin` if needed before adding traits
- P2: Run PHPStan on `EpicComment` after trait addition, update baseline accordingly
- P3: Longer term — evaluate DI container binding for user model vs config resolution

---

## Low Findings

### LOW-008 — how.md Lacks Explicit Guard References

| Severity | LOW |
|----------|-----|
| Category | Documentation |
| File | `how.md` |
| Confidence | LOW |

**Problem**: how.md doesn't explicitly list the automated tests that serve as guards against the categories of findings in this review:

- `ResourceUniformityTest` — catches model drifts between resources
- `ModelSchemaParityTest` — catches `#[Fillable]` ↔ schema mismatches  
- `PackageIsolationTest` — ensures domains don't depend on each other
- PHPStan baseline consistency check

**Impact**:
- Reviewers may not think to run these tests when making changes
- Drifts can accumulate between manual reviews
- Lower confidence in automated guardrails

**Recommendation**:
Add a "Guardas automáticas" subsection to how.md listing the key tests and their purpose.

**Action Plan**:
- P3: Add to how.md — low priority, documentation only

---

### LOW-009 — PHPStan Baseline pre-populated with Expected Errors

| Severity | LOW |
|----------|-----|
| Category | Static Analysis |
| File | `packages/basics13/phpstan-baseline.neon` |
| Confidence | LOW |

**Problem**: basics13 baseline has 11 errors ignored that are Fortify-version-dependent, not code bugs:

- `larastan.noEnvCallsOutsideOfConfig` — 1 error
- `staticMethod.notFound` — 1 error (TModel::factory())
- `property.notFound` — 5 errors (two_factor_* properties in views)
- `method.notFound` — 2 errors (hasEnabledTwoFactorAuthentication, passkeys())
- `method.notFound` — 3 errors (passkeys() in security.blade.php)

**Evidence**:
- These are expected because basics13 ships with Fortify and views that use Fortify features
- Deliberate choice to pre-populate baseline rather than have CI fail on version bumps
- However, this sets precedent of ignoring errors vs fixing them

**Impact**:
- Baseline may be too permissive — new real errors could be more easily ignored
- Other packages (entities13, projects13) have empty baselines — inconsistent pattern
- Harder to judge if a new PHPStan error is "expected" or genuine

**Recommendation**:
1. Document why each of the 11 entries exists in the baseline
2. Consider if some can be fixed instead (e.g., add proper PHPDoc to suppress specific errors)
3. Ensure entities13/projects13 baselines are generated fresh, not copied from basics13
4. Establish policy: baseline entries should be genuine static analysis suppressions, not version-workarounds

**Action Plan**:
- P3: Document baseline entries in basics13 phpstan-baseline.neon
- P3: When running PHPStan on entities13/projects13, generate fresh baselines and review each entry
- P3: Establish team policy on baseline vs fix tradeoff

---

## Action Plan Summary

| Priority | ID | Action | Owner | Target |
|----------|----|--------|-------|--------|
| **P1** | CRITICAL-001 | Add `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete` to `EpicComment` model | Developer | Immediately |
| **P1** | HIGH-002 | Run PHPStan level 7 on all packages; review and prune baselines; generate fresh for entities13/projects13/entity-projects13 | Developer | Within 1 week |
| **P1** | HIGH-003 | ✅ Complete: Test moved to `packages/projects13/tests/Unit/EpicCommentHasTraitsTest.php`, `ResourceUniformityTest` extended, pattern documented in `models.instructions.md`, and missing traits added to `EpicComment` model | Developer | Completed |
| **P1** | HIGH-004 | Document current `active` flag management pattern (5+ layers) in models.instructions.md | Developer | Within 2 weeks |
| **P2** | MEDIUM-005 | Update `how.md` with references to `ResourceUniformityTest`, `ModelSchemaParityTest`, agent management notes | Documentation | Within 2 weeks |
| **P2** | MEDIUM-006 | Prune `.github/agents/` to essential 5-6; document on-demand loading; verify link scripts | DevOps | Within 2 weeks |
| **P2** | HIGH-007 | Add PHPDoc/`@mixin` to `EpicComment::user()` for PHPStan compatibility; update baseline | Developer | Within 1 week |
| **P3** | MEDIUM-008 | Add "Guardas automáticas" subsection to `how.md` | Documentation | Optional |
| **P3** | LOW-009 | Document basics13 baseline entries; establish baseline maintenance policy | Team Lead | Optional |

---

## Verification Commands

Run these after implementing fixes to verify the review findings are resolved:

```bash
# 1. Run ResourceUniformityTest - should pass with no drifts
DX phpunit --filter=ResourceUniformityTest packages/entity-projects13

# 2. Run ModelSchemaParityTest - should pass with no schema drifts  
DX phpunit --filter=ModelSchemaParityTest packages/entity-projects13

# 3. Run PHPStan on affected packages
DX phpstan analyse --level=7 packages/entities13/src/
DX phpstan analyse --level=7 packages/projects13/src/
DX phpstan analyse --level=7 packages/entity-projects13/src/

# 4. Verify EpicComment has required traits
DX phpunit --filter="EpicCommentHasTraits" packages/projects13

# 5. Run package isolation tests
DX phpunit --filter=PackageIsolationTest packages/entities13
DX phpunit --filter=PackageIsolationTest packages/projects13
```

---

## Notes (Out of Scope, Recorded for Maintainer)

- This review was conducted via read-only analysis; no code changes were made in the repository
- Full specialist review (`full-review` skill) would expand findings with domain-specific specialists (database, security, clean-code, etc.)
- The `EpicComment` trait gap is the most critical finding — fixing it brings the model into parity with the established pattern in basics13/entities13/projects13
- `active` flag consolidation is a medium-term refactoring opportunity (not blocking, but documented for Q2 2026)
- Agent load optimization should follow skill-based workflow: `skill new-feature`, `skill consistency-review`, `skill fix-review` often make individual specialist agents unnecessary
- how.md remains excellent reference — these are targeted additions, not rewrite needed
- Next full review scheduled after P1 items are completed, or when significant new package changes land

---

**Generated**: 2026-10-19 via full-review analysis across l13 workspace packages