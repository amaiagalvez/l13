# Code Review

## Executive Summary

Focused, read-only review of the five reported customers13 PHPStan errors.
All five were reproduced. Three accepted findings cover the analysis failure,
stale seeder references, and an additional seeder/test behavior mismatch.
This is not a full repository audit.

## Detected Stack

- Package: `amaia/customers13`, Composer PHP constraint `^8.3`.
- Validation runtime: local Docker image `webdevops/php-apache-dev:8.4`.
- Declared development tools: Larastan `^3.9`, Testbench `^11.3`,
  PHPUnit `^12.0`, Pint `^1.27`, Dusk `^8.7`.
- PHPStan configuration: level 7, package source and tests included.
- Frontend, queues, Redis, Horizon and CI/CD: outside this focused review.
- Database: architecture documents MariaDB; live database not inspected.

## Checks Executed

- Package Git status and relevant diff: inspected; existing changes preserved.
- Docker PHPStan: **FAILED**, exit 1, exactly five errors.

  ```bash
  docker run --rm --pull=never --entrypoint "" \
    -u "$(id -u):$(id -g)" -e XDEBUG_MODE=off -e HOME=/tmp \
    -v "$PWD/packages":/packages \
    -v "$PWD/packages/customers13":/app-docker \
    -w /app-docker webdevops/php-apache-dev:8.4 \
    ./vendor/bin/phpstan analyse --no-progress --no-interaction
  ```

- PHPUnit: **NOT RUN**. Relevant feature tests use `RefreshDatabase`;
  test-database isolation was not established.
- Dusk, build, audit, Pint: **NOT RUN**, outside the supplied failure scope.

## Critical Findings

None identified in scope.

## High Findings

None identified in scope.

## Medium Findings

### TYPE-001 — List return contract is not established for static analysis

Severity: MEDIUM  
Category: Static analysis  
File: packages/customers13/src/Transformers/CustomerListTransformer.php  
Line: 83  
Confidence: HIGH

Problem: `editPayload()` promises `customer_type_ids: list<string>`, but PHPStan
infers `array<int, decimal-int-string>`.

Evidence: reproduced `return.type` error. The implementation already calls
`values()->all()`, and the diff removes the parameter's `Customer` PHPDoc.

Impact: package analysis fails. No malformed runtime list was demonstrated.

Recommendation: retain string conversion and the list return contract. Make
reindexing explicit with native `array_values()` around the resulting ID array
if needed, and verify the correction with package PHPStan. Keep the concrete
model type evident without changing the inherited method's native signature.

### TEST-001 — Feature tests reference deleted seeders

Severity: MEDIUM  
Category: Testing  
File: packages/customers13/tests/Feature/Seeders/EdgeCaseCustomerSeederTest.php  
Line: 16  
Confidence: HIGH

Problem: tests still invoke removed `EdgeCaseCustomerSeeder` and
`EdgeCaseCustomerTypeSeeder` classes.

Evidence: Git marks both seeder files deleted. The second test file invokes its
missing class at lines 16, 28 and 40. PHPStan reports all four missing-class
errors. Composer already defines the seeders namespace mapping.

Impact: static analysis fails and those tests cannot execute their seeding.

Recommendation: align tests with the intended seeder migration. Customer-type
edge fixtures now live in `DevelopCustomerTypeSeeder`; the old customer edge
fixtures are not supplied by `DevelopCustomerSeeder`. Decide whether those
customer fixtures remain required before removing their coverage.

### TEST-002 — Development seeder counts changed without matching tests

Severity: MEDIUM  
Category: Correctness / Testing  
File: packages/customers13/database/seeders/DevelopCustomerTypeSeeder.php  
Line: 24  
Confidence: HIGH

Problem: five fixed types are created in addition to the requested bulk count.

Evidence: `run(20)` creates five fixtures then twenty loop iterations, whereas
`DevelopCustomerTypeSeederTest.php:18` expects twenty total rows. Expected
source-derived state counts are now 15 active, 6 archived and 4 trashed, rather
than the asserted 12, 5 and 3. The default-count test also expects twenty rows.

Impact: correcting missing-class errors alone leaves further expected feature
test failures. PHPUnit was not run; these are predictions from the code.

Recommendation: explicitly settle whether the argument counts bulk records or
all records, then align seeder tests and migrated fixture coverage together.

## Low Findings

None identified in scope.

## Informational Findings

No application, test, dependency, configuration or database changes were made.

## Security

Not assessed in this focused static-analysis review.

## Bugs / Correctness

See TEST-002.

## Database

No database commands executed.

## Performance

Not assessed.

## Architecture

No new abstraction or package dependency is needed to address these findings.

## Testing

See TEST-001 and TEST-002; raw observations in [testing.md](testing.md).

## Production

No production-impact claims established in this scope.

## Maintainability

Keep fixture migration and its tests synchronized.

## Rejected Findings

- Autoload configuration defect: the seeders namespace is already mapped.
- Runtime non-list bug: not demonstrated; `values()` is already present.
- Five independent defects: four analysis errors share the deleted-seeder cause.
- Proven PHPUnit failures: no PHPUnit execution occurred.

## Action Plan

- P1: resolve TYPE-001 and update stale seeder references with preserved coverage.
- P1: settle and test the development seeder's intended count semantics.
- P2: rerun package PHPStan, then targeted seeder tests on an isolated database.

Raw Laravel observations: [laravel.md](laravel.md).
