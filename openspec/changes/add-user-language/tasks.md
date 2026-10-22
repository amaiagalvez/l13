# Tasks

## 1. Datos (basics13)

- [x] 1.1 Migración de `users.lang` (`string(5)`, nullable). `lang` en el `Fillable` del `User` y estado de factory. Verificación: `UserSchemaParityTest` actualizado y en verde.
- [x] 1.2 Enum `Basics13\Enums\Language` (`EU`, `ES`, `FR`, `EN`) con `codes()`, `options()` (código → nombre nativo) y `nativeName()`. Sin banderas ni métodos sin uso. Verificación: Unit test de `codes()` y `options()`.

## 2. Aplicar el idioma (basics13)

- [x] 2.1 Middleware `SetUserLocale` en el grupo `web`: aplica `lang` si es un caso de `Language` (`Language::tryFrom($lang) !== null`) y, si no, deja `config('app.locale')`. Verificación: Feature tests con idioma válido, nulo y no soportado.

## 3. Formularios y listado (basics13)

- [x] 3.1 `UserRequest` y la validación del perfil aceptan `lang` en `Language::codes()` o vacío. Verificación: Feature tests de valor válido, vacío y `de` rechazado.
- [x] 3.2 Selector de idioma con `Language::options()` en `users/form` y en los ajustes de perfil. El listado de usuarios muestra el nombre nativo. Verificación: Feature tests de que el formulario y el perfil muestran las opciones y guardan el valor, y de que el listado muestra "Euskara".
- [x] 3.3 Claves "Idioma" e "Idioma preferido" en es/eu/fr/en. Verificación: los tests de `TranslationFilesTestCase` pasan.

## 4. Verificación final

- [x] 4.1 Suites de `basics13` y `demo13`, PHPStan nivel 7 y Pint en Docker. La cobertura se mantiene ≥ 80 %.
