# Architecture Notes

Keep this document concise.

Update it when important architectural decisions are made.

---

## System Purpose

Authenticated users manage entities, their projects, and project epics. These resources can be
created, edited, searched, paginated, archived, activated, soft-deleted, restored, and permanently
deleted from a trash view.

---

## Main Modules / Domains

- `entities13`: entity controllers, Form Requests, policy, queries, transformers,
  Eloquent model, views, routes, migrations and entity development/edge-case seeders.
  It works without projects or the entity/project bridge.
- `projects13`: project and epic controllers, Form Requests, policies, queries,
  transformers, Eloquent models, factories, views, routes and migrations. Each epic
  belongs to one project. Projects work without entities or the bridge. Epics have comments
  (author and creation time), added from the epic edit drawer. Planning and timeline
  views and project/epic development/edge-case seeders are part of this package.
- `entity-projects13`: the optional bridge that makes each project belong to a entity.
  It owns the cross-domain schema, relationships, validation, queries, deletion guards
  and UI contributions, using the base packages' extension contracts and configuration.
- `basics13`: shared infrastructure, shared form/list scripts and styles, common JSON
  translations, browser-runner support, and reusable Fortify authentication/account
  settings. Its default user model, auth flows, migrations, factory and seeder are
  configurable by the host and contain no entity/project business logic.
- The host application retains its authentication configuration and middleware wiring,
  shared shell/navigation, dashboard, asset bundling and coordination of the installed
  packages' development seeders, plus integration tests for the installed packages.
- Authentication and account settings are provided by `basics13`: Laravel Fortify, email
  verification, passkeys, two-factor authentication, profile and password management.
  Hosts configure `auth.providers.users.model` to select the default package model or
  their own authenticatable Eloquent model.
- Package behavior is tested in each package with its own Testbench fixtures. Cross-domain
  tests, including the integrated browser flows, belong to `entity-projects13`; the host
  retains authentication/settings integration tests and installed-package coordination checks.
- Browser support declares Dusk in the development dependencies of `basics13` and
  `entity-projects13`, so package analysis does not need the host's development libraries.
- Policies accept Laravel's authentication contract. Epic comment authors, factories and
  development seeders use `auth.providers.users.model`, not a concrete host user class.

---

## Authentication / Authorization

- Fortify provides authentication and account security features.
- The `active` flag on users is read but never written by the application: nothing in `app/` sets
  it to false. Deactivating a user is a pending capability (user CRUD, `todo.md`), so the flag can
  only be set out of band for now. The three resources do have archive/activate endpoints.
- Inactive users cannot authenticate with passwords, two-factor challenges, passkeys,
  or remember-me cookies. A web middleware checks persisted user activity and logs out
  existing sessions on their next request after deactivation.
- Entity and project routes require `auth` and `verified` middleware.
- `EntityPolicy` is the authorization boundary for entity list, create, update,
  delete, restore, and force-delete actions. The current product scope permits every
  authenticated verified user to perform these actions.
- `ProjectPolicy` applies the same authorization boundary to project actions.
- `EpicPolicy` applies the same boundary to epic actions and to adding comments.

---

## Database

Engine: MySQL-compatible MariaDB in Docker. PHPUnit uses the `mysql` connection
and the `laravel_phpunit` database configured in `phpunit.xml`; Dusk keeps
`laravel_dusk` so browser migrations and PHPUnit transactions do not interfere.
The host test suite does not use SQLite.

Entities, projects, epics, and users have an `active` boolean, defaulting to true.
Resource index lists include only active, non-deleted records; archived lists include
only archived, non-deleted records and append the last modification date. Trash lists
include every deleted record regardless of `active`; restoring preserves that flag.
Archiving does not cascade to children. Index rows offer archiving instead of
deletion when a entity has projects or a project has epics, including deleted children.
Database-backed form selectors (select, select2, checkbox and radio) show only active,
non-deleted records when creating. When editing, they also retain the selected
non-deleted record even if it is archived; other archived records are not offered.
This applies to entity types on entities, entities on projects, and projects on
epics.
The `active` flag is never mass assignable on users or resources. Resource activation
changes only through the dedicated archive/activate actions, not through create,
update, or other request payloads.

Important constraints: entity and project names are unique among non-deleted records,
including archived records.
Soft-deleted names may be reused. Restoring a deleted record whose name is already
used by a non-deleted record is rejected with a conflict message. Projects require one
entity and have a nullable end date that cannot precede the required start date.
Entities with projects cannot be moved to the trash or permanently deleted.
Epic names are unique per project among non-deleted epics, including archived ones. Epic start and end dates are
optional, but an end date requires a start date and must be later than it. Projects
with epics (including trashed epics) cannot be moved to the trash or permanently
deleted. Epic comments are removed when their epic is permanently deleted and keep
a null author when the user is deleted.

Important transactions: no multi-step business transaction or queued write flow exists
currently. The delete guards in `entities13` and `projects13` lock their parent row with
`lockForUpdate()` inside their own `DB::transaction()`, so a child created between the
check and the delete makes the delete wait and then fail.

---

## Queues / Async Processing

- The default local queue driver is database-backed, but no application jobs or
	asynchronous business workflows are currently defined.

---

## Frontend

Blade / Livewire / Vue / Inertia:

- Blade with Livewire and Flux components, bundled by Vite Plus and Tailwind CSS.
- Host entry points import installed package assets through `vendor/amaia`; shared scripts
  and CSS can also be published with `basics13-assets`, and the entity selector with
  `entity-projects13-assets`. The host retains its branding and asset-bundling setup.
- Laravel Dusk covers the entity CRUD end to end. Projects and epics have Dusk coverage of
  their CRUD and of the parent selector, but not of their trash, archived lists or drawer;
  those flows are covered by feature tests only.

---

## External Services

- No application-specific external service integration was identified.
- Docker Compose provides MariaDB, phpMyAdmin, and MailHog for local development.

---

## Important Architectural Decisions

- Domain packages remain independent: `entities13` and `projects13` must not reference
  each other or `entity-projects13`. The bridge may reference both domains; the domains
  may use existing shared infrastructure in `basics13`.
- These boundaries cover implicit dependencies (schema, table/route/view names, payload
  keys, UI conditions, factories, seeders and tests), not only imports and Composer entries.
  Base extension points must be domain-neutral and retain standalone defaults.
  Cross-domain implementations and tests belong in the bridge or, for installed-package
  integration tests, the host application. Existing coupling is not a precedent.
- Keep entity and project list querying and presentation transformation separate
	from controllers.
- Enforce non-deleted entity-name uniqueness in both Form Request validation and the
	database; soft-deleted names may be reused. During creation, if a name is in the
	trash, the user can choose to create a new record or restore the existing one.
- Enforce project-name uniqueness in both Form Request validation and the database,
	while allowing names belonging to soft-deleted projects to be reused. During
	creation, if a name is in the trash, the user can choose to create a new record or
	restore the existing one.
- When the bridge is installed, it enforces the project-to-entity relationship with
  a foreign key and dynamically registered Eloquent `belongsTo` / `hasMany` relationships;
  neither base model declares the other domain.
- Keep browser tests in a separate CI job because they require a browser and a test
	database.

---

## Deliberately Rejected Patterns

Record patterns that should not be introduced without a concrete new requirement.

- No repository, service, DTO, domain-layer, CQRS, or event-sourcing abstraction is
	justified by the current application size.

---

## Production Assumptions

- CI runs PHP 8.3 and Node 22; local Docker currently runs PHP 8.4.
- Administrative Docker ports and development credentials are for local development
	only.
- Production deployments must provide a non-debug environment and isolated database
	credentials.

---

## Known Technical Debt

- CI now includes Dusk, but the browser job must be observed once in GitHub Actions to
	confirm the runner's Chrome/ChromeDriver paths.
