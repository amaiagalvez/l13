## Why
El proyecto tiene reglas y skills en `.github/`, pero no había una capa de invariantes que la IA
consultara antes de escribir código, ni forma de registrar por qué se cambió algo.

## What Changes
- Se añade `openspec/specs/` con las 10 capabilities del monorepo
- `openspec/project.md` documenta contexto y apunta a las skills existentes
- Las zone rules referencian la spec de su zona con una línea

## Impact
- Affected specs: package-boundaries
- Affected code: solo `.github/` y `openspec/` (sin cambio de runtime)