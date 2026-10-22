## Why

La interfaz siempre se muestra en `config('app.locale')`, aunque la aplicación está traducida a es, eu,
fr y en. Cada usuario debería verla en su idioma. Se separó de `add-roles-permissions-system`, que no
tiene relación con esto (code review 2026-10-22, ARCH-002).

## What Changes

- Columna `users.lang` nullable con uno de los códigos `eu`, `es`, `fr` o `en`.
- Un enum `Language` en `basics13` con los códigos y sus nombres nativos (Euskara, Español, Français,
  English), que usan la validación, los selectores y el middleware. Sin banderas: un idioma no es un
  país, y el euskera no tiene bandera de estado.
- Selector de idioma en el formulario de usuario y en los ajustes de perfil, para que cada usuario
  pueda elegir el suyo sin permisos de gestión de cuentas. El listado de usuarios muestra el nombre
  nativo.
- Un middleware web que aplica el idioma del usuario autenticado y, si no tiene o no es válido, usa
  `config('app.locale')`.

## Capabilities

### New Capabilities

Ninguna.

### Modified Capabilities

- `basics-auth`: el usuario guarda su idioma preferido y la interfaz lo aplica.

## Impact

- `basics13`: migración, enum, middleware registrado en el grupo `web`, `UserRequest`, validación del
  perfil, vistas de usuario y de perfil, transformer del listado y traducciones es/eu/fr/en.
- `UserSchemaParityTest` y la factory de usuario.
- Sin dependencias nuevas. Independiente de `add-roles-permissions-system`.
