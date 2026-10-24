# Spec Delta

## MODIFIED Requirements

### Requirement: Planning y timeline son de solo lectura
Ninguna pantalla agregada MUST crear, modificar ni eliminar registros de dominio al abrirse; las pantallas agregadas son planning, timeline y el calendario. Lo que cada una dibuja de dominio MUST autorizarse con la policy del proyecto, sin policy propia; la ruta del calendario no exige policy de dominio, porque también dibuja el calendario personal, cuyos eventos son siempre los del usuario de la sesión. Solo cuentan proyectos y epics activos; la papelera MUST quedar fuera.

#### Scenario: Un proyecto archivado con epics activos
- **WHEN** una pantalla agregada lee el conjunto activo
- **THEN** ni el proyecto archivado ni sus epics aparecen

### Requirement: Una sola fecha es un hito
Un registro con una única fecha MUST dibujarse como un día —en el tablero como una barra de un día y en el calendario como un evento de un día—, para que no desaparezca de ninguna pantalla agregada.

#### Scenario: Un epic con fecha de inicio y sin fecha de fin
- **WHEN** el tablero dibuja un epic con una sola fecha
- **THEN** su barra tiene el ancho de un hito y no desaparece

#### Scenario: Un epic con una sola fecha en el calendario
- **WHEN** el calendario dibuja un epic con una única fecha
- **THEN** aparece como evento de ese día y no se pierde
