# Testing review: customers13

Scope: seeder tests directly related to the supplied PHPStan errors.

- MEDIUM, HIGH confidence: `tests/Feature/Seeders/EdgeCaseCustomerSeederTest.php:16`
  and `tests/Feature/Seeders/EdgeCaseCustomerTypeSeederTest.php:16` still invoke
  deleted classes; the latter also invokes its missing class at lines 28 and 40.
  Update coverage to the intended replacement behavior rather than suppressing
  analysis or simply deleting the tests.
- MEDIUM, HIGH confidence: `database/seeders/DevelopCustomerTypeSeeder.php:24`
  starts creating five fixed types before the requested bulk types.
  `DevelopCustomerTypeSeederTest.php:18` still expects 20 total rows for `run(20)`,
  and its default-count test also expects 20. The current implementation would
  create 25 types: 15 active, 6 archived, 4 trashed. These are source-derived
  predictions, not executed PHPUnit results.

Database tests: NOT RUN. These tests use `RefreshDatabase`, and isolation of the
configured test database was not established during this focused review.
