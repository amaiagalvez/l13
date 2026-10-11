---
name: zone-rules-http
description: "HTTP/domain rules: canonical resource pattern, store/update, delete/restore, active flag, form requests, policies, queries, transformers. Load when editing app/Http/**, app/Policies/**, app/Queries/**, app/Transformers/**, packages/*/src/Http/**, packages/*/src/Policies/**, packages/*/src/Queries/**, packages/*/src/Transformers/**."
---

# Zone Rules: HTTP / Domain

## Canonical Resource Pattern
- `CustomerController` + `CustomerRequest` + `CustomerPolicy` + `Queries\Customers\CustomerListQuery` + `CustomerListTransformer`
- Mirror it (never reinvent it) when adding a resource.

## Store / Update
- Catch `QueryException` and call `UniqueConstraintViolation::rethrowAsValidationError()`.
- Restore flows return their own conflict response instead.
- `CustomerController::store` also returns JSON `201`/`409` responses for inline customer creation from the project selector; `CustomerCrudTest` fixes that contract.

## Deletion
- Deleting or force-deleting a parent is blocked if children exist, including trashed ones (`withTrashed()->exists()`).
- Customer→Project and Project→Epic work the same way.

## Active Flag
- `active` is not mass assignable: it only changes through the resource's `XInactiveController` (extends the shared `InactiveController`).
- Serves the archived list plus the `deactivate`/`reactivate` routes named `{plural}.archived.index`, `{singular}.deactivate`, `{plural}.archived.reactivate`.
- A new resource ships that controller too.

## Form Requests & Policies
- Form Requests authorize through the resource policy.
- Policies currently allow every verified user; this is documented in ARCHITECTURE.md, do not add roles unasked.

## Queries & Transformers
- List querying lives in `app/Queries`, row shaping in `app/Transformers`.
- Keep controllers thin.

## Validation Max Rules
- A `max` rule never carries a literal number.
- Reads `config('validation.max_length.string')` for a varchar-backed field or `config('validation.max_length.longtext')` for a `longText` one.
- `ValidationCoverageTest::test_string_length_limits_come_from_the_config` fails on a literal or on a key that does not resolve.