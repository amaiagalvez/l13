## 1. Documentación
- [x] 1.1 Crear openspec/project.md
- [x] 1.2 Crear las 10 specs (≤ 60 líneas cada una)
- [x] 1.3 Referenciar specs desde las zone rules
- [x] 1.4 Validar con openspec validate --specs --strict

## Nota tras archivar
El archivado añadió estos dos requirements a `openspec/specs/package-boundaries/spec.md` y se
retiraron a mano después. Son reglas del proceso de escribir specs, no invariantes de las fronteras
entre paquetes, y además violaban el primero de los dos (una spec no debe repetir lo que ya está en
una skill). Su contenido vive en la skill `openspec-workflow` y en la sección OpenSpec de `AGENTS.md`.
`package-boundaries` queda en 4 requirements para no pasarse del presupuesto de ~500 palabras.