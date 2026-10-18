## Why
The application needs a way to categorize entities by type (e.g., "Enterprise", "SME", "Individual", "Partner"). These types must support multilingual names in Basque, Spanish, French, and English to match the project's i18n requirements.

## What Changes
- Create new `EntityType` model in `entities13` package with fields: `name_eu`, `name_es`, `name_fr`, `name_en`
- Add migration with unique active name index per language (composite partial index)
- Implement full CRUD: Controller, TrashController, ArchivedController (extends shared InactiveController)
- Create FormRequest with validation for 4 language fields (required in EU, optional in others)
- Add ListQuery and SelectOptionsQuery with multilingual search
- Add Transformer for list and select options
- Create Flux UI views: list.blade.php with language tabs, form.blade.php with 4 language inputs
- Add language files for es, eu, fr, en (validation, messages, actions)
- Add Factory with multilingual fake data
- Write Feature tests: CRUD, Trash, Archived, SelectOptions, Migration, Integration
- Add routes: resource + archived + trash + options

## Impact
- Affected specs: `entities-domain` (new resource follows same patterns)
- Affected code: `packages/entities13/` (new model, controllers, requests, queries, transformers, views, lang, migrations, factories, tests, routes)
- No breaking changes - adds new resource alongside existing Entity
- Follows existing package boundaries: stays within `entities13` domain