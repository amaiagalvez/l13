# Spec-Driven Development (OpenSpec)

Manual corto de uso. El guion completo lo llevan las skills `openspec-*` y las reglas del proyecto
están en `openspec/config.yaml`; este fichero explica **qué es esto, para qué sirve y qué comando se toca en cada momento**.

---

## Qué es

Una capa que sitúa el "por qué" de las reglas del monorepo en un sitio que la IA consulta **antes**
de escribir código, y que deja registrado por qué cambió cada cosa.

No sustituye a:

- las **skills** (`.github/skills/`) → cómo se escribe el código
- las **zone rules** (`.github/instructions/`) → reglas de estilo por ruta
- los **tests** → lo que se verifica de verdad
- el **código** → lo que las specs describen

Las specs solo guardan lo que **ninguna de las tres cosas de arriba ya dice**.

---

## Las tres carpetas

```
openspec/
├── config.yaml     Contexto y reglas por artefacto/operación. Punteros a skills, no copias. Se edita a mano.
├── specs/          LO QUE ESTÁ CONSTRUIDO. Verdad actual. No se edita a mano.
│   └── <capability>/
│       ├── spec.md     Invariantes (≤ ~500 palabras)
│       └── design.md   Opcional: decisión a justificar, mapa de ficheros
└── changes/        LO QUE DEBERÍA CAMBIAR. Proposals en curso.
    ├── <change-id>/    proposal.md + tasks.md + design.md? + specs/<capability>/spec.md
    └── archive/        YYYY-MM-DD-<change-id>/  ← proposals ya mergeados
```

Regla mental: **`specs/` es el presente, `changes/` es el futuro, `archive/` es el pasado.**

---

## Los 10 capabilities

| Capability | De qué va |
|-----------|-----------|
| `package-boundaries` | Dirección de dependencias y qué cuenta como acoplamiento |
| `basics-auth` | Stack de autenticación reutilizable (Fortify + Livewire) |
| `basics-crud-base` | Controller delgado, Form Request, list queries |
| `basics-trash-archived` | Papelera, restore con conflicto, borrado definitivo, transacciones, carreras de nombre único |
| `basics-shared-views` | Componentes de lista y formulario, estado vacío |
| `entities-domain` | Invariantes de cliente |
| `projects-domain` | Invariantes de proyecto |
| `epics-comments` | Epic y sus comentarios |
| `planning-timeline` | Las dos pantallas agregadas (solo lectura) |
| `entity-projects-bridge` | Único punto de cruce entre dominios |

No hay una capability por paquete ni por clase. Si al añadir una spec te preguntas *"¿esto ya lo
deduce el agente leyendo el código, una skill o un test?"*, la respuesta es sí → **no la escribas**.

---

## Comandos

CLI oficial con versión fijada (las skills llaman a `openspec`, así que tiene que estar en el PATH):

```bash
npm i -g @fission-ai/openspec@1.14.1
openspec config set delivery skills   # solo skills, sin comandos duplicados por cliente
```

No uses `@forztf/openspec`: es una copia no oficial y abandonada.

| Comando | Para qué |
|---------|----------|
| `list` | Changes activos |
| `list --specs` | Specs existentes, con nº de requirements |
| `show <id> --type spec` | Leer una spec |
| `show <change-id> --json --deltas-only` | Deltas de un proposal, para depurar |
| `validate <change-id> --strict` | Validar un proposal |
| `validate --specs --strict --no-interactive` | Validar todas las specs |
| `validate --all --strict` | Validar todo |
| `archive <change-id> --yes` | Archivar un change ya mergeado y volcarlo a `specs/` |
| `view` | Dashboard interactivo |
| `update` | Regenera las skills `openspec-*` tras subir de versión el CLI |
| `init --tools claude,github-copilot,opencode` | Solo para repos nuevos o para migrar |

> **Deprecados, no usar:** `openspec spec list` y `openspec change list` (avisan por pantalla).
> Equivalentes vigentes: `openspec list --specs` y `openspec list`.

### Antes de cualquier tarea

```bash
openspec list                      # ¿hay ya un proposal que toque esto?
openspec list --specs              # ¿ya existe la capability?
openspec show <capability> --type spec   # leer solo esa spec, no las diez
```

### Skills (los tres clientes)

Una sola copia en `.github/skills/openspec-*`, que Copilot lee directamente y Claude/OpenCode a
través de sus enlaces. Se activan por nombre o pidiéndolo ("openspec propose …"):

| Skill | Para qué |
|-------|----------|
| `openspec-explore` | Pensar el cambio antes de proponerlo |
| `openspec-propose` | Crear el change con proposal, specs, design y tasks |
| `openspec-update-change` | Revisar un change ya creado |
| `openspec-apply-change` | Implementar sus tasks |
| `openspec-sync-specs` | Volcar deltas a `specs/` sin archivar |
| `openspec-archive-change` | Archivar tras el merge |

Son generadas: no se editan a mano; las reglas del proyecto van en `openspec/config.yaml`.

---

## El flujo

1. **Explorar** — `list`, `list --specs`, leer la spec afectada.
2. **Decidir** — ¿proposal o arreglo directo? (ver "Cuándo NO abrir proposal").
3. **Crear** — `openspec/changes/<verb-led-id>/` con proposal + tasks + deltas.
4. **Validar** — `validate <id> --strict` hasta que salga verde.
5. **Pedir aprobación** — parar aquí. No se implementa antes.
6. **Implementar** — seguir `tasks.md` en orden, marcando `- [x]`.
7. **Archivar** — tras el merge, `archive <id> --yes` en su propio PR.

Un proposal = un cambio atómico = un PR.

### Cuándo NO abrir un proposal

- Un bug que **restaura** lo que la spec ya dice → se arregla directo.
- Typo, formato, comentario.
- Dependencia no rompipe.
- Configuración o tests de comportamiento existente.

Si el bug es real pero **ninguna spec lo cubría**, ese hueco *es* el proposal.

---

## Cómo escribir una spec

### Regla de oro

Una spec dice lo que el código **no** dice. Si el agente puede deducirlo con `cat` o `rg` en
segundos, sobra.

| No va en `spec.md` | Dónde vive |
|-------------------|-----------|
| Listas de campos, `#[Fillable]`, casts | el modelo (y `ModelSchemaParityTest`) |
| Árboles de ficheros, listas de clases | `ls` / `rg` |
| Extractos de código | el fichero, o `design.md` |
| Estilo, Pint, PHPStan | skill `php-conventions` |
| Dónde van los tests, traits, Dusk | skill `test-rules` |

### Plantilla

```markdown
# Nombre legible

## Purpose
Qué invariantes cubre y por qué existe. Mínimo 50 caracteres.

## Requirements

### Requirement: <invariante>
Los <sujeto> MUST <comportamiento no evidente en el código>.

#### Scenario: <caso límite>
- **WHEN** <condición>
- **THEN** <resultado>
```

Dos avisos que costan sangre:

- **La primera línea de cada requirement debe llevar `MUST` o `SHALL`.** El validador solo mira esa
  línea; un `MUST` en la segunda no cuenta y falla con *"Requirement must contain SHALL or MUST"*.
- **Prohibiciones van como `MUST NOT`**, nunca `NO MUST`.

### Presupuesto

Cada spec: **≤ 60 líneas / ~500 palabras**. Se mide:

```bash
for f in openspec/specs/*/spec.md; do
  printf "%-46s %3s palabras ~%4s tokens\n" "$f" "$(wc -w < "$f")" "$(( $(wc -w < "$f") * 13 / 10 ))"
done
```

Si te pasas, casi siempre es porque estás describiendo el *qué* en vez del *por qué no puede ser de
otra manera*. Recorta, no crezcas.

---

## Cómo escribir un delta

Los deltas van en `openspec/changes/<id>/specs/<capability>/spec.md`, **uno por capability
afectada**, y usan operaciones:

```markdown
## ADDED Requirements        (nuevo)
## MODIFIED Requirements     (cambia el comportamiento)
## REMOVED Requirements      (se elimina)
## RENAMED Requirements      (solo cambia el nombre)
```

Dos cosas que rompen el archivado:

1. **`MODIFIED` reemplaza el requirement entero.** Hay que pegar el bloque completo desde
   `openspec/specs/<capability>/spec.md` y editarlo. Pegar solo el trozo nuevo borra los escenarios
   que no se copiaron. Si no cambias el comportamiento, usa `ADDED`.
2. **Cada requirement necesita al menos un `#### Scenario:`** (4 almohadillas). `### Scenario:` o
   `- **Scenario:**` no cuentan y fallan en silencio.

Para cambios solo de documentación o tooling, archivar con `--skip-specs` para no tocar `specs/`:

```bash
openspec archive <id> --skip-specs --yes
```

---

## `design.md`

Crear **solo** si hay una decisión que justificar: un trade-off, un patrón nuevo, una migración
complicada, una carrera de concurrencia. Ahí sí van rutas, mapas y extractos.

Ya existen tres, y **no resuelven** lo que documentan: describen discrepancias entre el código y la
documentación que están pendientes de decidir.

- `specs/basics-trash-archived/design.md` — ~~el código dice `ArchivedController`/`archive`, la doc dice
  `InactiveController`/`deactivate`;~~ **RESUELTO**: documentación unificada con el código (`archive`/`archived`); `app/…` vs `packages/*/src/…`; claves de config.
- `specs/package-boundaries/design.md` — mapa de propietarios y dónde vive cada invariante.
- `specs/projects-domain/design.md` — `entity_id` nullable vs `NOT NULL`; `UniqueProjectName` es
  código muerto.

Cuando una spec documenta un comportamiento que **cambia**, va como delta `MODIFIED` o `REMOVED`,
no se edita a mano.

---

## Errores frecuentes

| Mensaje | Causa |
|---------|-------|
| `Requirement must contain SHALL or MUST keyword` | El `MUST`/`SHALL` no está en la **primera línea** del requirement |
| `Requirement must have at least one scenario` | Falta `#### Scenario:` con 4 almohadillas |
| `Change must have at least one delta` | Falta `changes/<id>/specs/<capability>/spec.md` con cabecera `## ADDED…` |
| `Purpose section is too brief` | Menos de 50 caracteres en `## Purpose` |
| Escenarios que no se detectan | Revisar con `show <id> --json --deltas-only` |

---

## Mantenimiento

- Si `openspec/specs/` pasa de 10 capabilities, o una spec se acerca a 60 líneas, **algo se está
  duplicando**. Revisar en vez de crecer.
- Una sesión normal carga **1-4 specs** (la fila que toca del mapa de `AGENTS.md`), no las 10.
- Las skills `openspec-*` las regenera `openspec update`: **no editarlas a mano**.
- La sección OpenSpec de `AGENTS.md` vive una sola vez; el contexto y las reglas, solo en
  `openspec/config.yaml`.