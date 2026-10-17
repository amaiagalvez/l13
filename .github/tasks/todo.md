[] user demo13-tik basics-era pasa
[] command bat eguneratzeko laravel13-tik basics13-ra
[] crear el CRUD de usuarios en basics13
crear un nuevo archivo de configuracion y configurar la ruta register, por defecto sera false y entonces no se podran registrar nuevos usuarios desde fuera
[] gestion de roles y permisos
[] crear estructura de la base de datos y en full-review añadir que se mantenga actualizada
[] añadir datatables
[x] en los select hay dos opciones: 
    1. si el formulario se abre en modo create, en el select o select2 solo se mostrarán los que tengan active=1
    2. si el formulario se abre en modo edición, en el select o select2 se mostraran el elemento sseleccionado (tenga o no active=1) y el resto de elementos seran solo los que tengan acitive=1 
[] cachear listados select public function selectOptions y Observer para limpiar la cache ante cualquier modificación en el original
[] composer.json require y require-dev pasar a basics los que se pueda (debugbar no)
[] automatizar sentry <== claude begiratu, hor daude config file batzuk

# REVIEWS
[ ] @full-review analiza demo13 basics13 customer13 projects13 customer-projects13 y testing13 busca incongruencias entre los modelos, que tienen de diferente, que se gestiona de forma diferente en un modelo que en otro, que está testeado en uno si y en otro no, que código tienen repetido, donde hay sobreingenieria. repasa la apliación en busca de fallos y propon todas las mejoras que se te ocurran. Repasar los errores que da el phpstan y SonarQube y buscar una posible solucion.  busca incongruencias entre lo que hay en el código y lo que hay dentro de las carpetas .github y openspec y las configuraciones de los Agentes. Analiza tambien el fichero how.md busca incongurencias. Repasa la configuración de los AGENTES y analiza si hay alguna manera de ahorrar tokens y de mejorar la forma de trabajar
[ ] @fix-review .github/reviews/<fecha>/CODE-REVIEW.md antes de modificar pide confirmación y al terminar marca la tarea como realizada.

# PERFORMANCE
[ ] prepara un plan o acualiza el que ya existe que cree seeders para llenar las tablas con cientos de registros y luego analice el rendimiento del listado