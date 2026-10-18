## 1. Database & Model
- [ ] 1.1 Create migration `create_cum_entity_types_table.php` with fields: id, name_eu, name_es, name_fr, name_en, notes, active, timestamps, softDeletes, audit columns (created_by, updated_by, deleted_by)
- [ ] 1.2 Add composite partial unique indexes for each language: unique on (name_eu) where deleted_at is null, same for name_es, name_fr, name_en
- [ ] 1.3 Create `EntityType` model with Fillable, UsePolicy, UseFactory attributes, HasFactory, SoftDeletes, TracksAuditColumns, TransactionalDelete traits
- [ ] 1.4 Add casts for active (boolean) and dates

## 2. Factories & Seeders
- [ ] 2.1 Create `EntityTypeFactory` with multilingual fake data (name_eu required, others optional)
- [ ] 2.2 Add factory states for: active, archived, trashed

## 3. Policies
- [ ] 3.1 Create `EntityTypePolicy` with viewAny, view, create, update, delete, restore, forceDelete, archive, activate methods

## 4. Form Requests
- [ ] 4.1 Create `EntityTypeRequest` with validation: name_eu required|string|min:2|max:255|unique (ignoring deleted), name_es/name_fr/name_en nullable|string|max:255|unique (ignoring deleted), notes nullable|string, reuse_deleted_name boolean
- [ ] 4.2 Create `EntityTypeDestroyRequest` for delete authorization
- [ ] 4.3 Create `EntityTypeRestoreRequest` for restore authorization with name conflict handling
- [ ] 4.4 Create `EntityTypeListRequest` for list filters
- [x] 4.5 Create `EntityTypeSelectOptionsRequest` for select options

## 5. Queries
- [ ] 5.1 Create `EntityTypeListQuery` extending ListQueryBase with active/archived/trashed scopes and stateCounts
- [ ] 5.2 Create `EntityTypeSelectOptionsQuery` returning active types ordered by name_eu with id + text (name_eu as primary)

## 6. Transformers
- [ ] 6.1 Create `EntityTypeListTransformer` with active(), archived(), trashed() methods returning rows with language tabs
- [ ] 6.2 Create `EntityTypeSelectOptionsTransformer` for select options format

## 7. Controllers
- [ ] 7.1 Create `EntityTypeController` with index, selectOptions, store, update, destroy
- [ ] 7.2 Create `EntityTypeTrashController` with index, restore, destroy
- [ ] 7.3 Create `EntityTypeArchivedController` extending shared InactiveController with index, activate, archive

## 8. Routes
- [ ] 8.1 Add routes in `routes/web.php`: resource (index, store, update, destroy), archived (index, activate, archive), trash (index, restore, destroy), options

## 9. Views (Flux UI)
- [ ] 9.1 Create `resources/views/entity-types/list.blade.php` with Flux table, language selector tabs (eu/es/fr/en), status badges, action buttons
- [ ] 9.2 Create `resources/views/entity-types/form.blade.php` with 4 language inputs, notes textarea, Flux field components
- [ ] 9.3 Create shared list components if needed (reusing existing components from basics-shared-views)
- [x] 9.4 Add Entity Types menu item to sidebar navigation (projects/demo13/resources/views/layouts/app/sidebar.blade.php)

## 10. Language Files (i18n)
- [ ] 10.1 Create lang/eu/validation.php, messages.php, actions.php
- [ ] 10.2 Create lang/es/validation.php, messages.php, actions.php
- [ ] 10.3 Create lang/fr/validation.php, messages.php, actions.php
- [ ] 10.4 Create lang/en/validation.php, messages.php, actions.php

## 11. Tests
- [ ] 11.1 Create `EntityTypeMigrationTest` - verify table structure, indexes, foreign keys
- [ ] 11.2 Create `EntityTypeCrudTest` - create, read, update, list, search, validation, unique constraints per language
- [ ] 11.3 Create `EntityTypeTrashTest` - soft delete, restore with name conflict, force delete blocked by relations
- [ ] 11.4 Create `EntityTypeArchivedTest` - archive, activate, archived list
- [ ] 11.5 Create `EntityTypeSelectOptionsTest` - only active, ordered by name_eu, search
- [ ] 11.6 Create `EntityTypeIntegrationTest` - full workflow, policy authorization

## 12. Service Provider Registration
- [ ] 12.1 Register routes, views, translations, factories in `entities13ServiceProvider`

## 13. Validation & Polish
- [ ] 13.1 Run Pint to format code
- [ ] 13.2 Run PHPStan level 9
- [ ] 13.3 Run full test suite
- [ ] 13.4 Verify i18n parity test passes