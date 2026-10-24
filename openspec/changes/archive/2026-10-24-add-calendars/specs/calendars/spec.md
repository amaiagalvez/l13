# Spec Delta

## Purpose

La pantalla de calendario muestra en unas mismas vistas las fechas de proyectos, epics y los eventos personales del usuario, y aparta la creación de eventos al calendario personal.

## ADDED Requirements

### Requirement: El calendario deriva eventos de las fechas de dominio
Los eventos de proyecto y epic MUST derivarse de sus fechas de inicio y fin: un rango MUST dibujarse como evento de los días que abarca. Un registro sin ninguna fecha MUST NOT generar evento. La lectura pura y el conjunto activo son los invariantes de planning-timeline, que este calendario comparte como pantalla agregada.

#### Scenario: Un proyecto activo sin ninguna fecha
- **WHEN** el calendario compone sus eventos
- **THEN** ese proyecto no genera evento alguno

### Requirement: Solo el calendario personal admite eventos
Crear, editar y eliminar eventos MUST estar disponible solo sobre los eventos personales del usuario autenticado, y el dueño MUST tomarse del usuario autenticado, nunca del payload. Acciones de creación o edición sobre proyectos y epics desde el calendario MUST NOT ofrecerse.

#### Scenario: Un evento personal se crea desde el calendario
- **WHEN** un usuario envía el formulario de evento con datos válidos
- **THEN** se crea con él como dueño y su fecha aparece en la vista del periodo que la contiene

#### Scenario: Un user_id de otro usuario en el payload
- **WHEN** el formulario envía como dueño un usuario que no es el autenticado
- **THEN** el evento se crea con el autenticado, sin usar el valor enviado

#### Scenario: Un evento personal de otro usuario
- **WHEN** un usuario intenta editar o eliminar un evento ajeno
- **THEN** recibe 403 y el evento queda intacto

#### Scenario: Listar el calendario personal
- **WHEN** un usuario consulta su calendario
- **THEN** los eventos personales ajenos no aparecen

### Requirement: Un evento personal es CRUD plano con fechas acotadas
Un evento personal MUST NOT tener papelera ni archivados, y al eliminar su dueño MUST NOT quedar huérfanos. Un evento personal MUST llevar título y fecha de inicio obligatorios, y fecha de fin opcional; la fecha de fin, si existe, MUST ser ≥ la fecha de inicio, y una fecha con formato inválido o una fin anterior a la inicio MUST producir el error junto a su campo.

#### Scenario: Un evento sin fecha de fin
- **WHEN** se envía el formulario con fecha de inicio y sin fecha de fin
- **THEN** se crea como evento de un solo día

#### Scenario: Fechas inválidas
- **WHEN** se envía una fecha con formato inválido o una fin anterior a la inicio
- **THEN** el error aparece junto a su campo y no se guarda

#### Scenario: Se elimina el dueño de un evento personal
- **WHEN** el usuario dueño de eventos personales se elimina
- **THEN** sus eventos no quedan huérfanos

### Requirement: Cada calendario se puede ocultar de forma individual
Los tres calendarios (proyectos, epics y personal) MUST poder ocultarse y mostrarse de forma independiente desde la propia pantalla; ocultar uno MUST NOT afectar a los demás. El estado de visibilidad vive solo en la URL y, sin parámetros, los tres MUST verse.

#### Scenario: Ocultar el calendario de epics
- **WHEN** un usuario desactiva el calendario de epics
- **THEN** sus eventos desaparecen y los de proyectos y personales siguen

#### Scenario: Volver a mostrar un calendario ocultado
- **WHEN** el usuario reactiva el calendario de epics
- **THEN** sus eventos vuelven y los demás calendarios no cambian

#### Scenario: Abrir el enlace compartido
- **WHEN** otro usuario abre el enlace que codifica el estado
- **THEN** ve el mismo periodo y los mismos calendarios ocultos, y los eventos personales que aparecen son los suyos

### Requirement: Las cuatro vistas muestran el mismo conjunto
Mes, semana, día y año MUST dibujar el mismo conjunto de eventos de los calendarios visibles, cada vista acotada al periodo que nombra y contando la fecha de fin del evento como parte del periodo. La navegación entre periodos y vistas MUST mantener el estado en la URL y MUST tener un atajo para volver a hoy.

#### Scenario: Un evento fuera del periodo
- **WHEN** la vista de una semana concreta se dibuja
- **THEN** un evento de otra semana no aparece en ella

#### Scenario: Un evento que cruza el límite de dos semanas
- **WHEN** un evento va del fin de una semana al inicio de la siguiente
- **THEN** aparece en las dos vistas

### Requirement: Una celda muestra un número acotado de eventos
Cada celda de día (mes y semana) y cada mes (año) MUST dibujar como máximo un número fijo de eventos y MUST indicar cuántos quedan con un enlace "+N más" a la vista que los muestra todos (día para una celda de día, mes para un mes del año). La vista de día MUST mostrar todos sus eventos.

#### Scenario: Un día con más eventos de los que caben
- **WHEN** un día del mes tiene más eventos que el máximo de la celda
- **THEN** se ven los primeros, un enlace "+N más" con el número restante lleva a la vista de ese día, y allí aparecen todos
