# Database / concurrency — raw specialist report (2026-10-21)

Live schema was checked through Boost and matches the migrations.

### DB-001: Comment creation releases the epic lock before inserting, so a comment can land on a trashed epic
- Severity: MEDIUM · Confidence: HIGH
- File: packages/projects13/src/Http/Controllers/EpicCommentController.php:42-54
- Problem: `lockForUpdate` runs in its own `DB::transaction` (committed at line 45). The `trashed()` check and `save()` then run unlocked.
- Impact: this violates openspec/specs/epics-comments ("El borrado se bloquea en cuanto hay comentarios").
- Recommendation: run the lock, the check and the save in one transaction, as `EpicController::store` does.

### DB-002: Model `deleting` hooks call `$model->lockForUpdate()` without executing a query, so no lock is taken
- Severity: LOW · Confidence: HIGH [verified]
- File: packages/projects13/src/Models/Epic.php:85; packages/projects13/src/Models/Project.php:87
- Problem: the comments claim TOCTOU protection. The real guards are in packages/projects13/src/ServiceProvider.php:47-66.
- Recommendation: delete both `booted()` hooks.

### DB-003: Advanced text filter LIKE has no ESCAPE clause
- Duplicate of LAR-002 · File: packages/basics13/src/Queries/ListQueryBase.php:199-206

### DB-004: ARCHITECTURE.md describes epic-comment cascade and null-author rules that the schema and spec do not have
- Severity: LOW · Confidence: HIGH
- File: .github/docs/architecture/ARCHITECTURE.md:100-101; packages/projects13/database/migrations/2026_09_30_175707_create_epic_comments_table.php:17-18
- Problem: `user_id` is NOT NULL with RESTRICT, and `epic_id` is RESTRICT.
- Recommendation: fix the doc.

### DB-005: The epic delete guard ignores trashed comments
- Severity: LOW · Confidence: MEDIUM (latent: no code soft-deletes comments today)
- File: packages/projects13/src/ServiceProvider.php:64
- Recommendation: use `comments()->withTrashed()->exists()`.

Checked and fine:
- The generated `active_name` column has unique indexes.
- Restore and force-delete races are handled through the index and FK mapping.
- Parent guards lock the parent and count trashed children.
- Child-create locks the parent, with a consistent lock order.
- No destructive migrations.
- FKs are indexed.
- Filter columns come from the server.
