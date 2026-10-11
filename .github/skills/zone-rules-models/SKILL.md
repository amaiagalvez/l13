---
name: zone-rules-models
description: "Model rules: fillable, policies, casts, relationships, soft deletes, unique active names, deletion guards, column requirements. Load when editing app/Models/** or packages/*/src/Models/**."
---

# Zone Rules: Models

## Core Rules
- Models declare `#[Fillable]`, `#[UsePolicy]`, `casts()` and relationships only.
- Validation lives in Form Requests, authorization in policies, list querying in `app/Queries`.
- A derived read-only label is allowed when the same string is built for several surfaces (e.g., `Project::fullName()`).

## Soft Deletes & Unique Active Names
- Soft deletes per resource; names are unique among active records, enforced in the database (`App\Support\Database\UniqueConstraintViolation` absorbs the race).
- Both come from helpers in `app/Support/Database/helpers.php`:
  - `addCommonColumns($table)` adds `notes`, `active`, timestamps, soft deletes and the `['deleted_at', 'id']` index
  - `addUniqueActiveNameIndex($table, $scope)` adds the active-name unique index for the current driver

## Deletion Guard
- A record is refused deletion while it has children, trashed ones included.
- The guard is a `deleting` hook registered in `AppServiceProvider::blockDeletionWithChildren()`, which locks the row inside its own transaction.
- This makes delete safe, so a plain `$model->delete()` is protected too — callers must not lock the row again.

## Relationships
- Use explicit `belongsTo`/`hasMany` return types.
- Trashed children count where deletion is blocked (`withTrashed()`).

## New Column Requirements
A new column needs **three things together**:
1. Migration
2. `#[Fillable]` / `casts()` entry
3. Request rules

`ModelSchemaParityTest` enforces the first two.

A column a form edits needs a **fourth**: it travels to the drawer in the transformer's `editPayload()` (and in `payloadFor()` where the drawer is rehydrated by id), and the field is bound to it with `x-model`; a column nothing carries is emptied every time the form is opened.

## Notes Field
- `notes` is always nullable, so an insert that leaves it out keeps working.
- `nullable|string|max:` set to `config('validation.max_length.longtext')`
- Declared on `User`, `Customer`, `Project`, `Epic` and `EpicComment`

## EpicComment Special Case
- `epic_comments` is the only table whose `deleted_at` and `active` come from the helper **without a model behind them**.
- `EpicComment` declares neither `SoftDeletes` nor a cast for the flag — do not rely on them.

## Parent Relations
- A parent relation is never optional: `projects.customer_id` and `epics.project_id` are NOT NULL foreign keys with `restrictOnDelete`.
- Every `name` is NOT NULL.
- Read those relations directly — no `?->`, no `'—'` fallback.
- Never write a test that fabricates the missing state with `setRelation()`.