## Context
`basics13` ya posee el modelo `User`, sus acciones de Fortify y las traits de validación, pero no un
recurso de administración de cuentas. Este cambio añade el CRUD canónico de usuarios dentro del
paquete base, reutilizando todo lo que ya expone.

## Goals
- CRUD canónico (listar, crear, editar, archivar, papelera) sin reinventar el patrón de recurso.
- Mantener el contrato de modelo configurable del paquete (`auth.providers.users.model`).
- Reutilizar `ProfileValidationRules` y `PasswordValidationRules` en vez de duplicar reglas.
- Cero migraciones y cero cambios destructivos.

## Non-Goals
- No se añaden roles ni permisos (la policy permite a cualquier usuario verificado, como el resto).
- No se toca el comportamiento de login: archivar o enviar a la papelera una cuenta la deja fuera del
  acceso (`EnsureUserIsActive` y el scope de soft delete), y eso es intencionado, no un cambio de auth.
- No se delegan usuarios a un paquete nuevo: viven en `basics13`.

## Decisions

### Resolución del modelo
**Decisión**: resolver el modelo desde `auth.providers.users.model` en queries, requests y controller.
**Razón**: es el contrato ya usado por `CreateNewUser`, `ProfileValidationRules` y `FortifyServiceProvider`,
y está fijado por `projects/demo13/tests/Feature/Auth/ConfigurableUserModelTest`. Fijar
`Basics13\Models\User` rompería a los hosts que sustituyen el modelo.

### Registro de la policy con modelo variable
**Decisión**: registrar únicamente en el `ServiceProvider`
`Gate::policy(config('auth.providers.users.model'), UserPolicy::class)`, sin `#[UsePolicy]` en el modelo.
**Razón**: los atributos PHP no se heredan y `#[UsePolicy]` ata una clase concreta, así que un host que
sustituya el modelo se quedaría sin policy y el CRUD denegaría todas las habilidades. El registro
explícito cubre tanto el modelo por defecto como el sustituido, y `tests/Unit/ArchitectureTest` exige
que `Basics13\Models\User` no lleve la policy atada por atributo.

### Enlace del parámetro de ruta
**Decisión**: `Route::model('user', config('auth.providers.users.model'))` para el CRUD y el archivado;
las rutas de papelera usan `{trashedUser}` y sus requests leen `route('trashedUser')`.
**Razón**: el binding implícito del resto de recursos funciona por type-hint del controller
(`update(Entity $entity)`), imposible con un modelo configurable. `Route::model` resuelve `{user}`
sin fijar una clase, pero su scope por defecto excluye los trashed, así que la papelera no puede
compartir el nombre del parámetro: si lo hiciera, `SubstituteBindings` daría 404 antes de llegar al
request, que es quien resuelve el registro con `onlyTrashed()`.

### Queries sin integración
**Decisión**: `UserListQuery` pasa sus columnas de búsqueda literales a `paginate()` y devuelve un
objeto inerte en `integration()`.
**Razón**: `ListQueryBase` declara `integration()` abstracto porque los recursos de dominio tienen su
contrato; usuarios no tiene ninguno. No se crea una integración neutra (abstracción sin consumidor) ni
se generaliza la base compartida.

### Unicidad del correo
**Decisión**: correo único **global** (también frente a la papelera), no entre activos.
**Razón**: el correo identifica la cuenta; reutilizarlo tras un borrado lógico permitiría suplantar o
colisionar identidades, y el índice actual ya es global. Es una **excepción intencionada** al patrón
`entities-domain` / `basics-crud-base` ("único entre los no eliminados"), que aplica a los nombres
de dominio. No relajar el índice a parcial ni añadir `whereNull('deleted_at')` a la regla.

### Reutilización de validación
**Decisión**: `UserRequest` compone `ProfileValidationRules` (nombre, correo, notas) y
`PasswordValidationRules` (contraseña + confirmación).
**Razón**: son las mismas reglas del registro y del perfil; duplicarlas las haría divergir. En edición
la contraseña es opcional: sin campo enviado, se conserva la actual.

### Identidad por correo en el `TrashController`
**Decisión**: `UserTrashController` extiende el `TrashController` base, que es name-céntrico.
**Razón**: como el correo es único global, un restore nunca colisiona por correo; el pre-check
(`nameIsTaken()`/`takenBy('name')`) y el campo `resolve_name_conflict` del `RestoreRequest` no aplican
y se adaptan sin relanzar ni inventar un flag nuevo.

### Rutas
**Decisión**: `routes/users.php` cargado desde `ServiceProvider::boot()`, sin feature gate.
**Razón**: `basics13` no tiene `web.php`; se sigue el patrón de `settings.php`, y como en
`entities13`/`projects13` las rutas van dentro de `['web','auth','verified']`, no tras un flag.

## Risks / Trade-offs

| Riesgo | Mitigación |
|--------|------------|
| El correo global impide reutilizar cuentas dadas de baja | Es intencionado; el login y el registro ya lo asumen |
| El `TrashController` y `RestoreRequest` asumen `name` | Adaptar el hook a la identidad por correo; sin colisión por índice global |
| Una policy atada por atributo no alcanza a un modelo sustituido | Registrar la policy para `auth.providers.users.model` en el `ServiceProvider`, sin `#[UsePolicy]` |
| `openspec/specs/package-boundaries/design.md` está desactualizado | Corregir la propiedad de `User` en este mismo change |

## Migration Plan
1. Sin migraciones: el esquema de `users` ya existe y el correo ya es único global.
2. Implementar policy (registro para el modelo configurado), requests, queries, transformer,
   controllers, vistas, rutas y lang.
3. Pruebas Feature en `packages/basics13/tests/Feature/Users/`, incluida la del modelo configurado.
4. Corregir `package-boundaries/design.md`.

## Open Questions
- ¿Debe el alta ofrecer restaurar una cuenta en la papelera con el mismo correo, como hace el
  conflicto de nombre de clientes? (Actual: no; el correo es único global y basta el error de
  validación).
