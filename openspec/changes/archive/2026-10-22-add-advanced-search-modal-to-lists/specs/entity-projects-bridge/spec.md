## ADDED Requirements

### Requirement: El filtro por cliente lo aporta el bridge
El campo filtrable de cliente del listado de proyectos MUST llegar por el contrato de integración, no
desde el paquete base. MUST ser múltiple, admitir proyectos sin cliente y MUST NOT ofrecerse cuando
la lista ya está acotada a un cliente.

#### Scenario: Se filtran proyectos sin el bridge instalado
- **WHEN** el listado de proyectos se usa sin el bridge
- **THEN** la búsqueda avanzada no ofrece el campo de cliente
