# Tokens de diseño

Inspirados en la jerarquía y el aspecto de Vireo (radios amplios, sombras suaves de elevación), adaptados a Flux + Tailwind.
Cambia los valores aquí y en `resources/css/app.css` (`@theme`); no añadas colores, radios ni sombras sueltos en las vistas.

## Color (`@theme`)

```css
@theme {
    /* Marca: un único acento (escala brand-50…950 ya definida en app.css) */
    --color-accent: var(--color-brand-700);

    /* Semánticos */
    --color-success-600: #15803d;
    --color-warning-600: #b45309;
    --color-danger-600: #b91c1c;
}
```

- Neutros: escala `zinc-*` de Flux. Texto principal `zinc-900`, secundario `zinc-600`, bordes `zinc-200`, lienzo `zinc-50`, superficies de tarjeta `white`.
- El rojo (`danger`) solo para acciones destructivas, errores y variaciones negativas; el verde (`success`) para variaciones positivas. Siempre con signo o texto, nunca solo color.
- Acento de Flux: `--color-accent*` en `resources/css/app.css` apunta a `brand-700` (claro) y `brand-400` (oscuro).
- Degradado: solo en la métrica destacada del panel de inicio (una por página), `from-brand-600 to-brand-800` con texto blanco (AA).
- Modo oscuro: superficies en 2–3 niveles de gris (`zinc-950` lienzo, `zinc-900` tarjeta, `zinc-800` elevada), no una simple inversión. Comprueba contraste AA.

## Tipografía

- Cuerpo e interfaz: Inter (`font-sans`). Títulos (H1, títulos de tarjeta) y cifras grandes: Space Grotesk (`font-display`). Ambas se cargan con `bunny()` en `vite.config.js`. No añadir una tercera familia.
- Cifras con `tabular-nums`.
- Escala: 12 / 13 / 15 / 17 / 20 / 30 px. Título de página 30 px (600), título de sección 17–20 px (600), etiquetas y cabeceras de tabla 12–13 px (500), cuerpo y tablas 14–15 px, cifra de métrica 30 px.
- Líneas de texto de menos de 80 caracteres. Sentence case en todo; sin mayúsculas en etiquetas.

## Espaciado, radios y sombras

- Escala de 4 px. Separación entre secciones 24–32 px, dentro de tarjetas 20–24 px.
- Radios por jerarquía: controles y badges `rounded-lg` (8 px), tarjetas y tablas `rounded-2xl` (16 px), drawer y modal `rounded-3xl` (24 px), pastillas `rounded-full`.
- Sombras de elevación: solo las utilidades `shadow-card`, `shadow-raised` y `shadow-overlay` (`@utility` en `resources/css/app.css`, que leen `--elevation-*` de `:root` y `.dark`). No van en `@theme` porque Tailwind incrusta allí el valor y el modo oscuro no podría cambiarlo. Ninguna otra sombra.
- `shadow-card`: tarjetas, tablas y sidebar flotante, con borde `zinc-200` tenue (`dark:zinc-800`). `shadow-raised`: tarjeta enlazada al pasar el ratón (sin desplazamiento). `shadow-overlay`: drawer, modal, menús y toasts.

## Tamaños de control

| Elemento | Escritorio | Táctil |
|---|---|---|
| Botón / input | `h-9` (36 px) | `h-11` (44 px) |
| Botón de icono | 32 px | 44 px |
| Fila de tabla | 44–48 px | tarjeta |
| Topbar | 60 px | 56 px |
| Sidebar | 264 px (compacto 76 px) | drawer |

## Patrones de referencia

- **Cabecera de página:** migas visibles, H1 + subtítulo de una línea a la izquierda; a la derecha, como mucho una acción secundaria (`ghost`/`outline`) y la primaria con texto.
- **Tarjeta de métricas agrupadas:** una tarjeta, 4 celdas en rejilla (`grid sm:grid-cols-2 xl:grid-cols-4`) separadas por divisores; cada celda: icono en pastilla tenue, etiqueta, valor y variación con signo.
- **Métrica destacada:** etiqueta, valor grande, chip de variación y sparkline opcional.
- **Estado vacío:** icono sencillo, frase de una línea, botón de crear.
- **Badge de estado:** texto + color tenue (`bg-*-50 text-*-700`), nunca solo color.
- **Foco:** `focus-visible:ring-2 focus-visible:ring-brand-600 focus-visible:ring-offset-2`.
