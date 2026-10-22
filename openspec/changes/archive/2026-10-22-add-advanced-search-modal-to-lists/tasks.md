## 1. Basics13: Advanced Search Modal Component
- [x] 1.1 Create `AdvancedSearchModal` Livewire component in `packages/basics13/src/Livewire/AdvancedSearchModal.php`
- [x] 1.2 Create Blade view `packages/basics13/resources/views/components/list/advanced-search-modal.blade.php` with Flux modal, **field groups (collapsible sections), ordering within groups, conditional visibility, select type with options, operator selects, value inputs (text, date, number, select, checkbox for boolean), multiple select support** — _adjusted: one ordered list (by group and order), no collapsible sections_
- [x] 1.3 Add `searchableFields(string $scope): array` abstract method to `ListQueryBase` returning field definitions with full config (column, label, type, operators, default_operator, visible, order, group, nullable, placeholder, help, condition($scope), options($scope), multiple). **Callables receive $scope (active|archived|trashed) for scope-aware options/visibility.**
- [x] 1.4 Add `applyFilters(Builder $query, array $filters, array $fields, string $scope)` method to `ListQueryBase`
- [x] 1.5 Update `paginate()` to accept optional `$filters` parameter and call `applyFilters()` with current scope
- [x] 1.6 Add operator-to-SQL mapping logic in `ListQueryBase` for all types including `select` (with `in`/`not in` for multiple)
- [x] 1.7 Add advanced search button to `page-header.blade.php` (shared component) with slot for custom action, **only render when at least one field is visible after condition($scope) evaluation**
- [x] 1.8 Add "Show advanced fields" toggle in modal to reveal fields with `visible=false` (**local Alpine state, not persisted to URL**) — _adjusted: the toggle is Livewire component state, still not in the URL_
- [x] 1.9 Add help text/tooltip rendering for fields with `help` key
- [x] 1.10 Add placeholder rendering for fields with `placeholder` key

## 2. Basics13: Shared View Updates
- [x] 2.1 Update `list.searchable-results` component to accept and pass filter state and scope
- [x] 2.2 Add CSS/JS for filter badges showing active filters
- [x] 2.3 Add "Clear filters" action
- [x] 2.4 Add active filter badges in page header showing applied filters with remove (x) button that dispatches `filter-removed` event to update URL and reload list — _adjusted: each badge remove (x) is a link to the list without that filter_
- [x] 2.5 Ensure `page-header` receives scope and passes it to `searchableFields()` for button visibility

## 3. Entities13: Entity List Integration
- [x] 3.1 Implement `searchableFields(string $scope)` in `EntityListQuery` with full config:
  - `name`: column=`ENT_entities.name`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`basic`, order=10
  - `notes`: column=`ENT_entities.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=20, visible=`false`
  - `active`: column=`ENT_entities.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10 — **solo en el scope `trashed`**
  - `created_at`: column=`ENT_entities.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=10
  - `entity_type_ids`: column=`ENT_entity_entity_type.entity_type_id`, type=`select`, operators=[equals, not_equals], default_operator=`equals`, group=`relations`, order=10, options=fn($scope) => EntityType::query()->where('active', $scope==='active')->pluck('name_eu', 'id')->all(), multiple=`true`
- [x] 3.2 Update `active()`, `archived()`, `trashed()` to pass filters and scope to `paginate()`
- [x] 3.3 Update `resources/views/entities/list.blade.php` to include `AdvancedSearchModal` and pass field config with current scope
- [x] 3.4 Add Entity-specific operator translations (eu/es/fr/en) — _not needed: operator labels are shared in basics13_

## 4. Entities13: EntityType List Integration
- [x] 4.1 Implement `searchableFields(string $scope)` in `EntityTypeListQuery` with full config: — _adjusted: the entity-type list keeps the simple search only (spec updated)_
  - `name_eu`: column=`ENT_entity_types.name_eu`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`names`, order=10
  - `name_es`: column=`ENT_entity_types.name_es`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`names`, order=20
  - `name_fr`: column=`ENT_entity_types.name_fr`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`names`, order=30
  - `name_en`: column=`ENT_entity_types.name_en`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`names`, order=40
  - `notes`: column=`ENT_entity_types.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=50, visible=`false`
  - `active`: column=`ENT_entity_types.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10 — **solo en el scope `trashed`**
  - `created_at`: column=`ENT_entity_types.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=10
- [x] 4.2 Update `active()`, `archived()`, `trashed()` to pass filters and scope to `paginate()` — _adjusted: see 4.1_
- [x] 4.3 Update `resources/views/entity-types/list.blade.php` to include `AdvancedSearchModal` — _adjusted: see 4.1_
- [x] 4.4 Add EntityType-specific operator translations — _not needed: operator labels are shared in basics13_

## 5. Projects13: Project List Integration
- [x] 5.1 Implement `searchableFields(string $scope)` in `ProjectListQuery` with full config:
  - `name`: column=`PRO_projects.name`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`basic`, order=10
  - `notes`: column=`PRO_projects.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=20, visible=`false`
  - `start_date`: column=`PRO_projects.start_date`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=10, nullable=`false`
  - `end_date`: column=`PRO_projects.end_date`, type=`date`, operators=[equals, before, after, between, is_empty], default_operator=`between`, group=`dates`, order=20, nullable=`true`
  - `active`: column=`PRO_projects.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10 — **solo en el scope `trashed`**
  - `created_at`: column=`PRO_projects.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=30
  - `entity_id`: column=`PRO_projects.entity_id`, type=`select`, operators=[equals, not_equals, is_empty], default_operator=`equals`, group=`relations`, order=10, nullable=`true`, options=fn($scope) => Entity::query()->where('active', $scope==='active')->pluck('name', 'id')->all(), condition=fn($scope) => !request()->route('entity'), **multiple=`true`**
- [x] 5.2 Update `active()`, `archived()`, `trashed()` to pass filters and scope to `paginate()`
- [x] 5.3 Update `resources/views/projects/list.blade.php` to include `AdvancedSearchModal`
- [x] 5.4 Add Project-specific operator translations — _not needed: operator labels are shared in basics13_

## 6. Projects13: Epic List Integration
- [x] 6.1 Implement `searchableFields(string $scope)` in `EpicListQuery` with full config: — _adjusted: the epic list filters by `project_name` (text) (spec updated)_
  - `name`: column=`PRO_epics.name`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`basic`, order=10
  - `notes`: column=`PRO_epics.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=20, visible=`false`
  - `start_date`: column=`PRO_epics.start_date`, type=`date`, operators=[equals, before, after, between, is_empty], default_operator=`between`, group=`dates`, order=10, nullable=`true`
  - `end_date`: column=`PRO_epics.end_date`, type=`date`, operators=[equals, before, after, between, is_empty], default_operator=`between`, group=`dates`, order=20, nullable=`true`
  - `active`: column=`PRO_epics.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10 — **solo en el scope `trashed`**
  - `created_at`: column=`PRO_epics.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=30
  - `project_id`: column=`PRO_epics.project_id`, type=`select`, operators=[equals, not_equals], default_operator=`equals`, group=`relations`, order=10, options=fn($scope) => Project::query()->where('active', $scope==='active')->when(request()->route('entity'), fn($q) => $q->where('entity_id', request()->route('entity')))->pluck('name', 'id')->all(), condition=fn($scope) => !request()->route('project'), **multiple=`true`**
- [x] 6.2 Update `active()`, `archived()`, `trashed()` to pass filters and scope to `paginate()`
- [x] 6.3 Update `resources/views/epics/list.blade.php` to include `AdvancedSearchModal`
- [x] 6.4 Add Epic-specific operator translations — _not needed: operator labels are shared in basics13_

## 7. Language Files (i18n)
- [x] 7.1 Add operator labels to `packages/basics13/resources/lang/{eu,es,fr,en}/messages.php` (contains, equals, not_equals, starts_with, ends_with, before, after, between, is_empty, is_true, is_false, greater_than, less_than, greater_than_or_equal, less_than_or_equal)
- [x] 7.2 Add field type labels (text, date, boolean, number) — _adjusted: dropped, field type labels are not rendered_
- [x] 7.3 Add "Advanced search", "Clear filters", "Apply filters" translations
- [x] 7.4 Verify i18n parity test passes

## 8. Tests
- [x] 8.1 Unit test: `ListQueryBase` filter application (operators, SQL generation)
- [x] 8.2 Unit test: `searchableFields($scope)` returns correct structure for each domain (including groups, order, condition($scope), options($scope), multiple) — _adjusted: covered for users (UserListQueryTest) and through the domain feature tests_
- [x] 8.3 Feature test: Entity advanced search (text, date, boolean filters) — _adjusted: text and boolean covered (EntityListFilterTest); dates covered by the project and epic filter tests_
- [x] 8.4 Feature test: EntityType advanced search (multilingual text filters) — _adjusted: see 4.1_
- [x] 8.5 Feature test: Project advanced search (date range, entity filter with select type, conditional visibility)
- [x] 8.6 Feature test: Epic advanced search (date range, project filter with select type, conditional visibility) — _adjusted: epic date and project name filters (EpicListFilterTest)_
- [x] 8.7 Feature test: Combined simple search + advanced filters
- [x] 8.8 Feature test: Filter state in URL (shareable, bookmarkable)
- [x] 8.9 Browser/Dusk test: Modal opens, filters apply, results update
- [x] 8.10 Test: Archive/Trash lists also support advanced filters
- [x] 8.11 Unit test: Modal field grouping, ordering, and conditional visibility — _adjusted: conditional visibility and the advanced toggle are unit-tested; ordering is visual_
- [x] 8.12 Feature test: Select type field with options($scope) callable and conditional visibility (entity_id hidden when scoped)
- [x] 8.13 Feature test: "Show advanced fields" toggle reveals visible=false fields (local state, not URL)
- [x] 8.14 Unit test: Operator mapping for select type (equals, not_equals, is_empty, in/not_in for multiple)
- [x] 8.15 Feature test: Select multiple (entity_type_ids) generates IN/NOT IN queries correctly
- [x] 8.16 Feature test: Active filter badges in header with remove button updates URL and reloads list

## 9. Validation & Polish
- [x] 9.1 Run Pint to format code
- [x] 9.2 Run PHPStan level 7
- [x] 9.3 Run full test suite
- [x] 9.4 Verify no regression in existing simple search
- [x] 9.5 Test mobile responsive modal