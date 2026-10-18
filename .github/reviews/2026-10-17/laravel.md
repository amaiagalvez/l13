# Laravel review: customers13

Scope: the five supplied PHPStan errors and directly related changes. Read-only review.

- MEDIUM, HIGH confidence: `src/Transformers/CustomerListTransformer.php:83`.
  PHPStan cannot prove that `customer_type_ids` is a `list<string>`. The current
  pipeline already calls `values()`, so this is not evidence of nonsequential
  runtime keys. Preserve string IDs and the list contract; use explicit native
  array reindexing if necessary and validate with PHPStan.
- MEDIUM, HIGH confidence: the removed edge-case seeders remain referenced by
  their feature tests. Composer already maps the seeders namespace correctly;
  regenerating autoload files cannot restore deleted classes.

Validation: package PHPStan ran in `webdevops/php-apache-dev:8.4` and exited 1,
reproducing exactly the five supplied errors. No application code was modified.
