---
name: ux-design
description: "Apply before creating or changing any view, Blade component or style in this app. Covers lists, forms, drawers, modals, navigation, empty states, Tailwind v4 and the Basque (eu) copy. Not for backend-only changes."
---

# UX y diseño de interfaz

Sistema de diseño de la app (clientes → proyectos → épicas). Aplícalo antes de tocar vistas.
Sigue siempre las convenciones existentes (`AGENTS.md`, componentes `x-list.*`, Flux gratuito).
Tokens y detalles: [reference/tokens.md](reference/tokens.md).

## Reglas de oro

1. Una pantalla, una tarea principal. Una sola acción primaria visible, con texto.
2. Nada destructivo sin confirmación. Mover a papelera, archivar y borrar definitivamente piden confirmación en modal (`x-list.confirm-modal`) que explica la consecuencia y cómo recuperarlo; el éxito se anuncia con toast.
3. Un solo sistema de componentes: Flux + componentes Blade. No mezclar estilos nuevos.
4. Móvil y teclado primero.
5. Menos ruido: sin bordes dobles ni mayúsculas en etiquetas y cabeceras. La profundidad se expresa con las sombras de elevación de `tokens.md` (tarjeta, elevada, superpuesta), no con sombras sueltas.
6. Estilo visual de referencia: [Vireo · Sales](https://vireo.jawad.work/src/html/dashboards/sales.html), solo como inspiración de jerarquía y aspecto (radios amplios, sombras suaves, tarjetas de métricas). Plantilla con copyright: no copies su HTML, CSS ni clases `ax-*`; se reconstruye con Flux + Tailwind.

## Estructura de página

- Cabecera: H1 con el recurso + descripción de una línea + botón primario con texto a la derecha (p. ej. "Bezero berria"). Nunca solo un icono `+`.
- Migas de pan con la jerarquía real (Bezeroak › Proiektua › Epika). Sin "Dashboard" en listados raíz.
- Barra de herramientas: búsqueda a la izquierda (atajo `/`), filtros a la derecha.
- Pestañas Activos | Papelera (n) dentro de cada recurso, en lugar de iconos sueltos arriba a la derecha.
- Sidebar: marca del producto, textos en euskera, sin enlaces del starter kit (Repository, Documentation). Iconos distintos por sección. Enlaces agrupados por tarea con encabezado, contador opcional y activo con fondo tenue de acento.
- Topbar de 60 px: búsqueda global y menú de usuario.
- Panel de inicio: banda de bienvenida con 2–3 cifras y dos acciones, una métrica destacada, una sola tarjeta con 4 métricas en celdas (icono, etiqueta, valor, variación) y tablas de "recientes". Cada cifra enlaza a su listado filtrado.

## Gráficas

- ApexCharts aprobado (2026-10-22) para el panel de inicio y la planificación: componente `x-basics13::chart` (Alpine `chart` en basics13); el host declara la dependencia npm y la expone como `window.loadApexCharts` (import diferido, chunk propio). Úsalo solo cuando una tendencia o comparación diga más que un número.
- Colores de la serie desde los tokens (`brand-*`, `success`, `danger`), con modo oscuro.
- Cada gráfica con alternativa textual (`aria-label` con la conclusión o tabla `sr-only`) y sin animación con `prefers-reduced-motion`.

## Listados

- Columnas: nombre (texto, no interactivo: se edita con el lápiz de la fila), dato útil (nº de hijos, fechas), acciones a la derecha. No poner editar/borrar pegados a la izquierda.
- Acciones: objetivo ≥32px (`size="sm"`, 44px en táctil), tooltip y `aria-label`. Hoy cada fila tiene como mucho dos (editar + archivar o mover a la papelera); si alguna llega a tres o más, agruparlas en un menú "⋯".
- Acción bloqueada (candado): tooltip con el motivo y enlace a los hijos.
- Fechas legibles y localizadas con el formato normativo de cada idioma: euskera `2026-10-02` (año-mes-día, Euskaltzaindia), castellano `02-10-2026`, francés `02/10/2026`, inglés `Oct 2, 2026`. Fuente única: `x-list.local-time` y `formatLocalDateTime`.
- Cabeceras de tabla en sentence case, 12–13px, `text-zinc-600` (contraste AA).
- Paginación 15 por defecto, con "n–m / total".
- Vacío: `x-list.empty-state` (un icono, una frase y, por el slot, el botón de crear; sin botón si la búsqueda acotó la lista). Carga: la búsqueda marca `aria-busy` y atenúa las filas. Móvil: filas → tarjetas (CSS de `.resource-list-table` + `data-label` desde la cabecera; no hace falta marcado por listado).

## Formularios y drawers

- Uno o dos campos → modal pequeño. Drawer solo para entidades con relaciones o comentarios (épicas).
- Foco automático en el primer campo. Enter guarda, Esc cierra, aviso si hay cambios sin guardar.
- Pie fijo: Utzi / Gorde. Un botón desactivado debe explicar por qué o quedar activo y avisar.
- Validación al salir del campo, con mensaje específico y cómo arreglarlo: automática en los formularios `data-track-changes` (vacío obligatorio → `validation.required`; otra regla → el texto del icono de info). Si la regla ya está escrita bajo el control (`x-forms.field-hint`), esa línea se pone en rojo; no se repite.
- Foco visible: anillo de 2px color de marca (`focus-visible:ring-2 focus-visible:ring-brand-600 focus-visible:ring-offset-2`). Nunca quitarlo.
- Fechas: selector nativo bien estilado. Explicar la regla "fin posterior al inicio", no solo bloquearla.
- Comentarios: iniciales, fecha relativa, campo de texto siempre visible al final.

## Feedback

- Toast (Flux) para éxito. Callout solo para errores que bloquean.
- El mismo verbo en botón y toast ("Gorde" → "Gordeta").
- Conflicto de nombre en papelera: dos opciones claras (restaurar el existente / crear uno nuevo) y la consecuencia de cada una.
- Bloqueos por dependencias: explicar y enlazar a los elementos que bloquean.

## Copy (euskera)

- Verbos activos, sentence case. Nada de "Aceptar" ni "Enviar".
- Usa las claves de `lang/eu.json`. Si añades un texto, añádelo en los 4 locales (hay test de paridad). No inventes traducciones: si dudas, deja la clave en inglés y avisa.
- El euskera suele ser más largo: comprueba que nada se rompe en móvil.

## Accesibilidad y movimiento

- Contraste AA, `focus-visible`, iconos solos con `aria-label`, no depender solo del color. Compruébalo con axe-core (Playwright) en claro y oscuro, escritorio y móvil, también con drawer y modal abiertos; no a ojo.
- `prefers-reduced-motion` ya está cubierto globalmente en `basics13/resources/css/app.css` y en `x-basics13::chart`.
- Movimiento solo como respuesta a una acción (abrir drawer, confirmar), 150–200 ms, respetando `prefers-reduced-motion`. Sin animaciones decorativas de entrada.

## Límites del proyecto

- Conserva todos los `data-test` (los tests Dusk dependen de ellos).
- Flux gratuito: no uses componentes Pro. Lo que falte se resuelve con Alpine + Tailwind.
- No cambies dependencias ni crees carpetas base sin aprobación; aprobadas: ApexCharts. Sustituir select2/jQuery requiere aprobación explícita.
- Blaze: no pliegues (`fold`) componentes con estado global (`$errors`, auth, sesión).
- Tailwind v4: tokens en `@theme` (`resources/css/app.css`), `gap-*` en vez de márgenes entre hermanos, y soporte de modo oscuro si la pantalla ya lo tiene.
- Tras cambiar vistas: build de Vite, Pint, tests y Dusk afectados.

## Antes de terminar

- [ ] Una acción primaria con texto, y H1 presente.
- [ ] Ninguna acción destructiva sin modal de confirmación.
- [ ] Todo operable con teclado, foco visible.
- [ ] Textos en los 4 locales, sentence case.
- [ ] Probado en móvil y modo oscuro.
- [ ] `data-test` intactos.