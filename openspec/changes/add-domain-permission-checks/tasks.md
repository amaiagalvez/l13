# Tasks

## 1. entities13

- [ ] 1.1 Migración de datos: permisos `entities.*` y `entity_types.*`, y `entities.manage` y `entity_types.manage` en `admin`. Idempotente. Verificación: Feature test de que repetirla no duplica y de que `admin` los tiene.
- [ ] 1.2 `EntityPolicy` y `EntityTypePolicy` con `hasPermissionTo()`. Los actores de los tests existentes reciben `admin`. Verificación: Unit tests de las policies con y sin permiso, Feature tests de 403 en listado, edición y borrado definitivo, y la suite del paquete en verde.

## 2. projects13

- [ ] 2.1 Migración de datos: permisos `projects.*` y `epics.*`, y `projects.manage` y `epics.manage` en `admin`. Verificación: Feature test igual que 1.1.
- [ ] 2.2 `ProjectPolicy` y `EpicPolicy` con `hasPermissionTo()`. Comentar exige `epics.edit`. Planning y timeline exigen `projects.view`. Verificación: Unit tests de las policies y Feature tests de 403 en listados, planning, comentario y borrado definitivo.

## 3. entity-projects13

- [ ] 3.1 Autorizar las opciones de cliente del formulario de proyecto con `projects.create` o `projects.edit`. Verificación: Feature tests con y sin `entities.view`, y 403 sin permisos de proyecto.
- [ ] 3.2 Los flujos Dusk cruzados usan un actor con `admin`. Verificación: suite Dusk en verde.

## 4. Traducciones

- [ ] 4.1 Etiquetas de cada nuevo slug de permiso en es/eu/fr/en en el paquete que lo declara. Verificación: tests de traducciones de cada paquete en verde.

## 5. Verificación final

- [ ] 5.1 Suites de `entities13`, `projects13`, `entity-projects13` y `demo13`, Dusk, PHPStan nivel 7 y Pint en Docker. La cobertura se mantiene ≥ 80 %.

## Workflow follow-up

- Al archivar: actualizar `.github/docs/architecture/ARCHITECTURE.md`, donde `EntityPolicy`, `ProjectPolicy` y `EpicPolicy` dejan de permitir todo a cualquier usuario verificado.
