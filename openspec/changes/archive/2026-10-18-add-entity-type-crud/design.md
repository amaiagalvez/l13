## Context
Add a new `EntityType` resource to the `entities13` package for categorizing entities. The key difference from existing resources is multilingual name fields (name_eu, name_es, name_fr, name_en) instead of a single `name` field.

## Goals
- Implement full CRUD following existing canonical patterns
- Support 4 languages with euskera as required primary language
- Maintain unique active name per language (partial indexes)
- Full i18n parity (eu/es/fr/en) verified by tests
- Zero breaking changes to existing Entity resource

## Non-Goals
- No changes to `Entity` model or its relations
- No bridge package changes (entity-projects13)
- No API resources - only web UI with Flux
- No new domain packages

## Decisions

### Multilingual Name Storage
**Decision**: Store 4 separate columns (`name_eu`, `name_es`, `name_fr`, `name_en`) rather than JSON or translation table.
**Rationale**: 
- Simple, performant, follows Laravel conventions
- Partial unique indexes work natively per column
- Easy to query, validate, and display
- No JOIN overhead for list views

### Unique Index Strategy
**Decision**: Create 4 partial unique indexes, one per language column.
**Rationale**: 
- Each language's name must be unique independently
- Partial index (WHERE deleted_at IS NULL) allows reuse after soft delete
- Matches existing `addUniqueActiveNameIndex` helper pattern

### Primary Display Language
**Decision**: Use `name_eu` as primary display in lists/selectors; provide language tabs to view/edit others.
**Rationale**: 
- Euskera is mandatory and primary locale
- Selectors need a single stable text field
- Tabs allow viewing all translations without clutter

### Validation Rules
**Decision**: `name_eu` required, others nullable but unique when present.
**Rationale**: 
- Business requirement: euskera name is mandatory
- Other languages optional but must be unique if provided
- Consistent with "name" field in existing resources

### Select Options Query
**Decision**: Return `name_eu` as `text` field; search across all 4 languages.
**Rationale**: 
- Select2-style dropdown needs single display text
- Users may search in any language
- Primary locale (eu) shown by default

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| 4x unique indexes increase migration complexity | Use helper function similar to `addUniqueActiveNameIndex` |
| Validation messages in 4 locales must stay in sync | Add i18n parity test (existing pattern) |
| Form with 4 fields may be tall | Use Flux tabs or accordion for languages |
| Search across 4 columns may be slower | Add composite index if needed; current dataset small |

## Migration Plan
1. Create migration with 4 name columns + common columns
2. Add 4 partial unique indexes in same migration
3. Run migration in demo13
4. Verify with `EntityTypeMigrationTest`

## Open Questions
- Should the select options search across all 4 languages or only name_eu? (Current: all 4)
- Should the list view show all 4 names or just name_eu with hover/tooltip for others? (Current: tabs)
- Do we need a fallback display when name_eu is somehow missing? (Not possible - required field)