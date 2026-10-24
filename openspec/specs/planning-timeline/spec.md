# Planning y timeline

## Purpose
Garantiza que las dos pantallas agregadas son lecturas puras: derivan proyectos activos y sus epics en
memoria, no escriben nada y no duplican la autorización que ya pertenece al proyecto.

## Requirements

### Requirement: Planning y timeline son de solo lectura
Ninguna pantalla agregada MUST crear, modificar ni eliminar registros de dominio al abrirse; las pantallas agregadas son planning, timeline y el calendario. Lo que cada una dibuja de dominio MUST autorizarse con la policy del proyecto, sin policy propia; la ruta del calendario no exige policy de dominio, porque también dibuja el calendario personal, cuyos eventos son siempre los del usuario de la sesión. Solo cuentan proyectos y epics activos; la papelera MUST quedar fuera.

#### Scenario: Un proyecto archivado con epics activos
- **WHEN** una pantalla agregada lee el conjunto activo
- **THEN** ni el proyecto archivado ni sus epics aparecen

### Requirement: Una fila sin fecha no está atrasada
Una fila sin fecha de fin MUST presentarse como no planificada y MUST quedar al final del orden. El orden
MUST traer al principio lo que vence antes.

#### Scenario: Un proyecto sin fecha de fin en el roadmap
- **WHEN** un proyecto activo no tiene fecha de fin
- **THEN** aparece sin barra, al final y etiquetado como no planificado

### Requirement: Una sola fecha es un hito
Un registro con una única fecha MUST dibujarse como un día —en el tablero como una barra de un día y en el calendario como un evento de un día—, para que no desaparezca de ninguna pantalla agregada.

#### Scenario: Un epic con fecha de inicio y sin fecha de fin
- **WHEN** el tablero dibuja un epic con una sola fecha
- **THEN** su barra tiene el ancho de un hito y no desaparece

#### Scenario: Un epic con una sola fecha en el calendario
- **WHEN** el calendario dibuja un epic con una única fecha
- **THEN** aparece como evento de ese día y no se pierde

### Requirement: La búsqueda en timeline estrecha los dos niveles
Un término que coincide con un proyecto MUST traer todos sus epics, y uno que solo coincide con un epic
MUST traer ese epic dejando fuera los demás. En planning MUST aplicarse solo al proyecto.

#### Scenario: Solo un epic coincide con el término
- **WHEN** el término coincide con un epic y no con su proyecto
- **THEN** el proyecto aparece como contexto y solo ese epic se dibuja

### Requirement: La ventana tiene límites y los recuentos no dependen de la búsqueda
La ventana MUST contener hoy, abrir y cerrar en límites de mes y MUST tener un ancho máximo. Los totales
MUST describir el conjunto activo completo.

#### Scenario: El conjunto de la página abarca varios años
- **WHEN** los registros se separan más que el ancho máximo
- **THEN** la ventana se acota y los totales siguen completos
