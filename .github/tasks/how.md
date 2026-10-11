# Guía de uso con tu configuración actual. Todo cuelga de tres piezas que ya tienes: 

* AGENTS.md (contexto que Copilot carga en cada sesión vía copilot-instructions.md)
* .github/instructions/ (reglas que se inyectan solas al tocar cada ruta: http, models, views y tests)
* los agentes de .github/agents/ y los flujos guiados (skills) de .github/skills/

## Reutilización en OpenCode y Claude Code

Los originales siguen en `.github/agents/` y `.github/skills/`. Ejecuta los dos scripts desde la raíz del repositorio después de clonar o al añadir o renombrar un agente o una skill:

```
bash testing13/agents/link-opencode-assets.sh
bash testing13/agents/link-claude-assets.sh
```

Ambos localizan la raíz del repositorio automáticamente y son seguros de volver a ejecutar:

* OpenCode descubre agentes en `.opencode/agents/`; son symlinks a `.github/agents/*.agent.md`. El instalador intenta reiniciar automáticamente su servicio en segundo plano con `opencode service restart`.
* Claude Code descubre agentes en `.claude/agents/`; son symlinks directos a `.github/agents/*.agent.md`, sin copiar ni adaptar el frontmatter. Los permisos de Copilot/OpenCode no se convierten a restricciones de herramientas de Claude.
* Las skills quedan enlazadas en `.opencode/skills/` y `.claude/skills/`, respectivamente, desde `.github/skills/`.
* Los archivos `.github/instructions/*.instructions.md` siguen siendo instrucciones de Copilot. OpenCode y Claude necesitan seguir las indicaciones de zona de `AGENTS.md` o de las propias skills; el `applyTo` de Copilot no se aplica en esos clientes.

Los scripts eliminan primero los symlinks previos en sus carpetas de agentes y vuelven a crearlos. Claude retira también las copias generadas por la versión anterior del script; otros archivos normales se conservan. Los enlaces están en `.gitignore`.

Las rutas `.claude/{agents,skills}/` y `.opencode/{agents,skills}/` son de solo lectura en VS Code, tanto al abrir la carpeta como el workspace. Edita agentes y skills desde `.github/`; los cambios de contenido se reflejan inmediatamente en los enlaces. Ejecuta los scripts para reconstruirlos tras añadir, borrar o renombrar archivos. Este bloqueo es del editor: los symlinks no impiden escribir en los originales desde un terminal u otro editor.

Si el script no encuentra el CLI o falla el reinicio, informa del problema; en ese caso ejecuta `opencode service restart` manualmente.

# Punto de partida común (los 3 flujos)

* Abre VS Code en la raíz del repo y comprueba en el chat: Boost MCP activo (laravel-boost en el selector de tools) y modo Agent (no "Ask": necesitas que edite archivos).
Nueva sesión por tarea. El contexto de AGENTS.md + las instrucciones de ruta entran solos; no se lo repitas.
* Los comandos seguros (php artisan test, pint --dirty, make:, npm run build, docker compose up) se ejecutan sin pedirte clic. Si pide otra cosa, revísala.

## Añadir una funcionalidad
1. Activa la skill `new-feature` y pásale la descripción:
```
skill new-feature — CRUD de facturas: listado con búsqueda, formulario, papelera y restore, con i18n
```
2. Déjalo seguir su guion: lee arquitectura/reglas de ruta → plan corto (aquí corrige el alcance si algo no te cuadra) → tests Feature rojos → implementa siguiendo el patrón de los hermanos (XController + XTrashController, Form Request, policy, XListQuery, Transformers, componentes list/, Flux).
3. Supervisa los puntos de control, no el proceso:
   * El plan inicial: ¿querías eso o algo distinto?
   * Los lang/*.json en los 4 locales y los data-test si hay UI.
4. Verificación final: el agente ya ha corrido los tests y los 4 tests de arquitectura (los de `AGENTS.md`) y te ha dado su resultado real. Tú comprueba que cuadra con el diff y, si tocaba UI, que el build se hizo (auto-aprobado) o levanta dev (-p 5173:5173 ... run dev).
5. Revisa el diff y commitea tú.

Sin skill también funciona: describe la feature y el agente acabará en lo mismo. La skill ya no aporta la regla de no montar abstracciones de más (eso vive ahora en `AGENTS.md`, siempre activo); lo que aporta es el guion fijo: plan antes de escribir, TDD, los 4 tests de arquitectura al final y el informe.

Si lo que quieres es un **recurso CRUD entero** (modelo + migración + vistas), usa la skill `new-resource` y pásale el nombre: `skill new-resource — Invoice`. Monta el set completo calcado a Customer (XController + XTrashController, XRequest/XListRequest/XRestoreRequest, policy, XListQuery, XListTransformer, vistas con x-list.table y x-forms.tracked-resource, lang en 4 locales) siguiendo TDD, y termina exigiendo pint, PHPStan y los tests de arquitectura en verde. `new-feature` sigue siendo lo indicado para features que no son un recurso completo.

## Revisar incongruencias
Cuatro niveles, de menos a más:

### Rápido
* "Revisa los cambios sin commitear de mi rama"	
* Informe en el chat; solo entonces se carga review-rules.md

### Puntual
* Selecciona un agente concreto del selector del chat (#security-reviewer, #database-reviewer…)	
* Auditoría de un área, READ-ONLY

### Completa
* skill `full-review`
* Informe en .github/reviews/YYYY-MM-DD/CODE-REVIEW.md con IDs estables (SEC-001, DB-004…) tras el abogado del diablo

### Consistencia entre recursos
* skill `consistency-review`
* Compara customers/projects/epics (o el recurso nuevo) contra los patrones canónicos de forms y listados y lista las divergencias con IDs (PAT-001…). Útil tras varios `new-resource` seguidos: la forma del último recurso nuevo se va desvianzando sin que nadie se dé cuenta.

Flujo recomendado (el de tu README): `full-review` → abre el CODE-REVIEW.md → selecciona los hallazgos que te convenzan → sesión nueva → skill `fix-review` con los IDs:

```
skill fix-review — SEC-002 y DB-004
```
Esa skill reverifica que el hallazgo sigue existiendo, hace el cambio mínimo y ejecuta los tests. La revisión es asesoría: verifica a mano seguridad, auth y migraciones destructivas.

## Arreglar bugs
* Bug en comportamiento → sesión nueva y descríbelo con stack/error: "El botón de restore no aparece en la papelera de epics tras borrar con comentarios" → él reproduce con el test más estrecho, distingue si es bug de código o de test, cambio mínimo, Pint, rerun.
* Tests rojos → skill `fix-tests` (tests/Feature/Epics o --filter=...). Distingue bug/test/entorno y nunca dice "en verde" sin haberlo ejecutado.
* Bug de UI/JS → pega error de consola (Boost MCP browser-logs si hay pestaña de debug) → el fix va con Dusk solo si es comportamiento JS (regla de tests.instructions.md).
* Hallazgo de revisión → no lo repitas: usa la skill `fix-review` con su ID para que respete la evidencia original.

## Comprobaciones automáticas (no hace falta pedirlas)
Al correr `php artisan test` (y en la CI) pasan también cuatro tests de arquitectura que vigilan la coherencia del proyecto:
* **tests/Unit/ArchitectureTest.php** — cada recurso trae su set completo (controller, controller de papelera, requests, policy, list query, transformer, vistas list/form) y los controllers son thin: sin `DB::`/`Schema::`/`dd()`/`paginate()` y con FormRequest declarado en todo endpoint que recibe input.
* **tests/Unit/ValidationCoverageTest.php** — todo lo que un modelo puede asignar en masa (`#[Fillable]`) y todo lo que los controllers leen con `$request->string()/boolean()` tiene su regla en un FormRequest. Sube un campo, una migration o un flag de formulario sin validar y se pone rojo.
* **tests/Unit/ModelSchemaParityTest.php** — modelos y esquema no se desincronizan: una columna nueva sin `#[Fillable]` falla hasta que la añades (o la declaras como columna de servidor).
* **tests/Unit/ResourceUniformityTest.php** — los recursos no se desvían entre sí: mismo orden de reglas y mismo `reuse_deleted_name` en los `XRequest`, mismas reglas de búsqueda en los `XListRequest`, mismas habilidades de policy con su call-site HTTP, factories con los mismos estados, mismo estado `archived` en todo modelo con flag `active`, mismo page size y desempate en las list queries, y el mismo sobre de acciones y `editPayload` allowlisted en los transformers. Un recurso nuevo que se desvíe se pone rojo aquí.

El resto del automatismo:
* `composer ci:check` = `config:clear` + pint --test + PHPStan (nivel 9) + suite completa con cobertura mínima del 80% y Clover en `storage/logs/coverage.xml` (el mismo umbral que exige la CI).
* Tests en paralelo en local: `DX php artisan test --parallel --compact <ruta>` — los flags van **antes** de la ruta. Nunca en CI: los jobs comparten una BD. Requiere el grant `laravel\_test\_%` (ya aplicado, y en `packages/basics13/tooling/mysql/init/02-parallel-test-databases.sql` para volúmenes nuevos).
* Restaurar desde la papelera ya pide `XRestoreRequest` (policy + `resolve_name_conflict` validado). Si añades un endpoint con input, la validación va en su FormRequest, nunca en el controller.
* CI (master y PRs): cache de Composer y npm, job Dusk, y se salta entera si el diff solo toca `.md`. Dependabot cubre composer y npm.

## Reglas del juego
* Recordar una convención: record-rule está apagado → edita a mano .github/instructions/<zona>.instructions.md (http, models, views, tests; mismo applyTo, añade el bullet). Ahí vive todo lo que no quieras repetir.
* Tokens: una sesión = una tarea; ciérrala. Las reglas de ruta solo entran cuando se toca la ruta, y las skills solo cuando las activas.
* Arquitectura (repositorios, DTOs, servicios…): el prompt y review-rules.md lo bloquean sin causa concreta — si de verdad lo necesitas, dilo explícitamente.
* Tras cada sesión: git diff + tests tú mismo + commit.