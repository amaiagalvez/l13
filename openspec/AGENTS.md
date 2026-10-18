# OpenSpec Instructions (Project-Specific)

Instructions for AI coding assistants using OpenSpec for spec-driven development in this project.

## TL;DR Quick Checklist
- Search existing work: `openspec spec list --long`, `openspec list` (use `rg` for full-text search)
- Decide scope: new capability vs modify existing capability
- Pick a unique `change-id`: kebab-case, verb-led (`add-`, `update-`, `remove-`, `refactor-`)
- Scaffold: `proposal.md`, `tasks.md`, `design.md` (if needed), and delta specs per affected capability
- Write deltas: use `## ADDED|MODIFIED|REMOVED|RENAMED Requirements`; include at least one `#### Scenario:` per requirement
- Validate: `openspec validate [change-id] --strict` and fix issues
- Request approval before implementation

## Project Conventions (from `openspec/project.md`)
- Capabilities are verb-noun: `user-auth`, `payment-capture`
- Change IDs: kebab-case, verb-led, unique
- One change = one atomic change = one PR
- Specs = invariants code/tests/skills no longer express
- Changes = proposals in `openspec/changes/<id>/`
- After merge: `openspec archive <id> --yes` updates specs

## Current Capabilities (see `openspec/specs/`)
| Capability | Purpose |
|------------|---------|
| `basics-auth` | Reusable auth stack (Fortify + Livewire) |
| `basics-crud-base` | Thin controller, Form Request, unique name in two layers |
| `basics-shared-views` | Canonical Flux forms/lists, shared components |
| `basics-trash-archived` | Trash/restore, archive flag, delete guards, transactional delete, unique-name race |
| `customer-projects-bridge` | Cross-domain coupling in bridge only, domains standalone |
| `customers-domain` | Customer + CustomerType CRUD, multilingual types |
| `epics-comments` | Epic unique per project, inherits project, delete guarded by comments, comments = half-resource |
| `package-boundaries` | Dependency direction: bridge → domains, never domain → domain |
| `planning-timeline` | Shared calendar/bar/status logic, extracted collaborator |
| `projects-domain` | Project + Epic CRUD, customer_id nullable in DB, required in validation |

## When to Create a Proposal
Create proposal for: new features, breaking changes, architecture shifts, security/performance changes.
**Skip proposal for**: bug fixes restoring spec behavior, typos, config changes, tests for existing behavior.

## Spec File Format (Critical)
- Every requirement MUST have at least one `#### Scenario:` (4 hashtags)
- Use `## ADDED|MODIFIED|REMOVED|RENAMED Requirements` for deltas
- MODIFIED must include the full updated requirement text
- SHALL/MUST for normative requirements

## Project Map (from `.github/docs/architecture/ARCHITECTURE.md` and `specs/`)
- **Models, casts, columns, relationships** → `specs/basics-support`, `specs/<domain>-domain`
- **Controllers, requests, policies, queries** → `specs/basics-crud-base`, `specs/basics-trash-archived`, `specs/<domain>-domain`
- **Trash, restore, archive, force delete** → `specs/basics-trash-archived`
- **Flux views, shared components** → `specs/basics-shared-views`
- **Epics & comments** → `specs/epics-comments`
- **Planning & timeline** → `specs/planning-timeline`
- **Customer↔Project relations, imports** → `specs/customer-projects-bridge`, `specs/package-boundaries`

## CLI Essentials
```bash
openspec list                    # What's in progress?
openspec show [item]             # View details
openspec diff [change]           # What's changing?
openspec validate --strict       # Is it correct?
openspec archive <change-id> [--yes|-y]  # Mark complete
```

## Key Reminders
- Specs are truth. Changes are proposals. Keep them in sync.
- Don't add complexity without concrete need (see `ARCHITECTURE.md`).
- Package boundaries are mandatory: bridge → domains, never domains → bridge.