## Context
Add an advanced search/filter modal to all resource list views (entities, entity-types, projects, epics). The modal presents all searchable fields with appropriate operators per field type, allowing users to build complex filter queries.

## Goals
- Reusable modal component in `basics13` shared views
- Type-aware operators: text (contains, equals, not equals, starts with, ends with), date (equals, before, after, between, is empty), boolean (is true, is false), number (equals, not equals, greater than, less than, between)
- Declarative field configuration in each domain's ListQuery
- Preserve existing simple search behavior
- Full i18n support (eu/es/fr/en)
- Works with active, archived, and trashed list states

## Non-Goals
- Saved filters / filter presets
- Cross-resource search
- SQL injection risk - all filters must be parameterized
- No changes to package boundaries

## Decisions

### Component Architecture
**Decision**: Single-file Livewire component `AdvancedSearchModal` in `basics13/src/Livewire/AdvancedSearchModal.php` with Blade view in `basics13/resources/views/components/list/advanced-search-modal.blade.php`.

**Rationale**:
- Encapsulates all modal state and logic
- Receives field configuration as prop from parent list view
- Emits filter events that the list query can consume
- Follows existing Flux modal patterns in the project

### Field Configuration
**Decision**: Each ListQuery defines a `searchableFields()` method returning an array of field definitions with full modal configuration:
```php
[
    'name' => [
        'label' => 'Name',
        'type' => 'text',
        'operators' => ['contains', 'equals', 'not_equals', 'starts_with', 'ends_with'],
        'default_operator' => 'contains',
        'visible' => true,
        'order' => 10,
        'group' => 'basic',
        'placeholder' => 'search.name.placeholder',
        'help' => 'search.name.help',
    ],
    'start_date' => [
        'label' => 'Start date',
        'type' => 'date',
        'operators' => ['equals', 'before', 'after', 'between', 'is_empty'],
        'default_operator' => 'between',
        'visible' => true,
        'order' => 20,
        'group' => 'dates',
        'nullable' => false,
    ],
    'active' => [
        'label' => 'Active',
        'type' => 'boolean',
        'operators' => ['is_true', 'is_false'],
        'default_operator' => 'is_true',
        'visible' => true,
        'order' => 30,
        'group' => 'status',
    ],
    'entity_id' => [
        'label' => 'Entity',
        'type' => 'select',
        'operators' => ['equals', 'not_equals', 'is_empty'],
        'default_operator' => 'equals',
        'visible' => true,
        'order' => 40,
        'group' => 'relations',
        'nullable' => true,
        'options' => fn (string $scope) => Entity::query()->where('active', $scope === 'active')->pluck('name', 'id')->all(),
        'condition' => fn (string $scope) => !request()->route('entity'),
        'multiple' => true,
    ],
]
```

**Configuration keys**:
- `label` (required): i18n key for field label
- `type` (required): `text`, `date`, `boolean`, `number`, `select` (for foreign keys with options)
- `operators` (required): array of operator keys available for this field
- `default_operator` (optional): operator pre-selected when field is added
- `visible` (optional, default `true`): whether to show in modal
- `order` (optional, default `999`): display order within group
- `group` (optional, default `'general'`): visual grouping in modal (e.g., 'basic', 'dates', 'status', 'relations')
- `nullable` (optional, default `false`): whether field accepts null/empty values (enables `is_empty` operator)
- `placeholder` (optional): i18n key for input placeholder
- `help` (optional): i18n key for help text/tooltip
- `condition` (optional): callable receiving `$scope` (active|archived|trashed) returning bool to conditionally show field (e.g., hide `entity_id` when already scoped to an entity). **Receives current list scope so options can match.**
- `options` (optional, for `select` type): callable receiving `$scope` (active|archived|trashed) returning `[value => label]` array. **Receives current list scope so dropdown options respect the active/archived/trashed state.**
- `multiple` (optional, for `select` type, **default `true`**): whether multiple values can be selected. **All select fields are multiple by default; set `multiple=false` for single-select.**

**Rationale**:
- Keeps field knowledge in the query layer (where columns are defined)
- Allows domain-specific fields, operator subsets, and conditional visibility
- Integrates with existing `searchColumns()` pattern
- Grouping and ordering make modal usable with many fields
- Conditions handle contextual filtering (e.g., scoped views)
- **Options/conditions receive scope for consistency with list state**
- **Select fields default to multiple for flexible filtering (e.g., filter entities by multiple types)**

### Query Integration
**Decision**: Extend `ListQueryBase::paginate()` to accept an optional `$filters` array alongside `$search`. Add protected `applyFilters(Builder $query, array $filters, array $searchableFields)` method.

**Rationale**:
- Minimal change to existing query flow
- Filters are applied after base scoping, before pagination
- Maintains compatibility with simple search

### Operator Mapping
**Decision**: Map UI operators to SQL conditions in `ListQueryBase`:
- text: `contains` → `LIKE %value%`, `equals` → `= value`, `not_equals` → `!= value`, `starts_with` → `LIKE value%`, `ends_with` → `LIKE %value`
- date: `equals` → `= value`, `before` → `< value`, `after` → `> value`, `between` → `BETWEEN`, `is_empty` → `IS NULL`
- boolean: `is_true` → `= true`, `is_false` → `= false`
- number: `equals` → `=`, `not_equals` → `!=`, `gt` → `>`, `gte` → `>=`, `lt` → `<`, `lte` => `<=`, `between` → `BETWEEN`
- select: `equals` → `= value`, `not_equals` → `!= value`, `is_empty` → `IS NULL` (if nullable), `in` → `IN (...)` (if `multiple=true`)

**Rationale**:
- Clear mapping from UI to SQL
- Uses parameterized queries
- Extensible for future types
- Select type supports single/multi-value with same operators as number

### URL State
**Decision**: Advanced filters serialize to URL query parameters (`filters[field][operator]=value`). Simple search remains `search=term`. Both can coexist. The "Show advanced fields" toggle is **local Alpine state** in the modal (not persisted to URL). Active filter badges in the page header have remove buttons that dispatch `filter-removed` event to parent, which updates URL and reloads list.

**Rationale**:
- Shareable, bookmarkable filter state
- Browser back/forward works
- No conflict with existing search param
- Toggle is UX-only, not shareable state
- Badge removal triggers immediate list refresh

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Complex modal with many fields | Group fields by type; collapsible sections; search within fields |
| Performance with many filters | Indexes on commonly filtered columns; limit filter count per request |
| i18n for operator labels | Add to existing language files; parity test |
| Mobile usability | Modal is full-screen on mobile (Flux default) |

## Migration Plan
1. Create `AdvancedSearchModal` Livewire component and Blade view in `basics13` (with field groups, ordering, conditional visibility, select type, "Show advanced fields" toggle)
2. Add `searchableFields()` abstract method to `ListQueryBase` (receives `$scope` for options/conditions)
3. Implement `applyFilters()` in `ListQueryBase` (handles select type with `in` operator for multiple)
4. Update each domain's ListQuery (`EntityListQuery`, `EntityTypeListQuery`, `ProjectListQuery`, `EpicListQuery`) with field definitions including `select` types with scope-aware callables
5. Add advanced search button to `page-header.blade.php` (shared component) - only renders when visible fields exist
6. Update list views to include modal, pass field config, and render active filter badges with remove buttons
7. Add i18n translations
8. Write tests

## Open Questions
- Should the modal support AND/OR between filter groups? (Start with AND only)
- Should date fields support relative dates (last 7 days, this month)? (Future enhancement)
- Should numeric fields support "is empty"? (Yes, if nullable)
- Should `select` type support `in` operator for multiple selection? (Yes, when `multiple=true`)