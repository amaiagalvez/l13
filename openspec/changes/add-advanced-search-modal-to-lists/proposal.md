## Why
The current search in list views is limited to a simple text search across a few predefined columns. Users need a more powerful filtering capability to find records by specific field values with different operators (equals, greater than, less than, not equal, between, etc.) depending on the field type (text, date, boolean, number).

## What Changes
- Create a reusable **AdvancedSearchModal** Livewire component in `basics13` that renders a modal with filter fields for each searchable column
- Add an **Advanced Search** button to the list page header (shared component) that opens the modal
- Extend `ListQueryBase` in `basics13` to support advanced filter conditions (operators per field type)
- Define searchable field metadata (column name, label, type, available operators) in each domain's ListQuery
- Update list views in `entities13` (entities, entity-types) and `projects13` (projects, epics) to use the new advanced search
- Add advanced search parameters to the list request (separate from simple search)
- Add i18n translations for operator labels and field types in 4 languages (eu/es/fr/en)
- Write Feature tests for advanced search in each resource

## Impact
- Affected specs: `basics-shared-views` (new modal component, enhanced page header), `basics-crud-base` (ListQueryBase advanced filtering), `entities-domain` (entity/entity-type list integration), `projects-domain` (project/epic list integration)
- Affected code: `packages/basics13/` (Livewire component, shared views, ListQueryBase), `packages/entities13/` (queries, views), `packages/projects13/` (queries, views)
- No breaking changes - simple search remains the default; advanced search is additive
- Follows existing package boundaries: shared logic in `basics13`, domain-specific field definitions in each domain