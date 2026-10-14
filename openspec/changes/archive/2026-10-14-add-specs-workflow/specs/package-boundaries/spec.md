## ADDED Requirements

### Requirement: Las specs documentan invariantes, no el código
Una spec MUST describir lo que ni el código ni los tests ni las skills expresan ya. Cada spec MUST
limitarse a 60 líneas y las listas de campos, rutas y extractos MUST ir en `design.md`.

#### Scenario: Se propone añadir un campo a un modelo
- **WHEN** un proposal afecta a un modelo existente
- **THEN** la spec del dominio no enumera sus campos y el test de paridad sigue siendo la fuente de
  verdad sobre el esquema

### Requirement: Los cambios de comportamiento pasan por un proposal
Una modificación del comportamiento de un recurso MUST registrarse como proposal en
`openspec/changes/` antes de implementarse.

#### Scenario: Cambiar la semántica del restore
- **WHEN** va a cambiarse qué ocurre al restaurar un registro con conflicto de nombre
- **THEN** existe un proposal con delta `MODIFIED` para `basics-trash-archived` validado con `--strict`