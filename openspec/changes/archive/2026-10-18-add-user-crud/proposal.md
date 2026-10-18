## Why
`basics13` ya provee el modelo `User`, su stack de autenticación y la gestión de cuenta, pero no
ofrece la administración de cuentas como recurso: hoy no se pueden listar, crear, editar, archivar ni
enviar a la papelera usuarios desde la aplicación. Añadir ese CRUD al paquete base permite reutilizar
la administración de cuentas en cualquier host, apoyándose en el modelo que el host configure.

## What Changes
- Añadir el recurso canónico de usuarios en `basics13`: Controller + TrashController +
  ArchivedController + Policy + ListQuery + SelectOptionsQuery + ListTransformer + FormRequests.
- Resolver el modelo desde `auth.providers.users.model`, como ya hacen las acciones de Fortify, para
  no acoplarse al modelo concreto del paquete. Como el binding implícito por type-hint no admite un
  modelo variable, se registra la policy para el modelo configurado (`Gate::policy`) y se enlaza
  `{user}` explícitamente (`Route::model`).
- Reutilizar `ProfileValidationRules` y `PasswordValidationRules` para nombre, correo, notas y
  contraseña en lugar de duplicar reglas.
- Mantener el correo único **global** (incluida la papelera), como excepción intencionada al patrón de
  unicidad entre activos de los recursos de dominio.
- Añadir vistas Flux (listado + formulario), rutas `users.*` (incluido `users.options`) cargadas como
  en los paquetes de dominio, y claves i18n en los 4 locales y pruebas Feature en el paquete.

## Impact
- Affected specs: `basics-auth` (nuevos requirements de gestión de cuentas y unicidad de correo).
- Affected code: `packages/basics13/` (policies, requests, queries, transformers, controllers, vistas,
  rutas, lang, tests, `ServiceProvider`) y `openspec/specs/package-boundaries/design.md` (corrige la
  propiedad de `User`, que aún figura como de `projects/demo13`).
- Sin cambios destructivos: no se añade ni modifica ninguna migración; el correo ya es único global.
