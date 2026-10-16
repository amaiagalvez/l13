[] user demo13-tik basics-era pasa
[] command bat eguneratzeko laravel13-tik basics13-ra
[] crear el CRUD de usuarios en basics13
crear un nuevo archivo de configuracion y configurar la ruta register, por defecto sera false y entonces no se podran registrar nuevos usuarios desde fuera
[] gestion de roles y permisos
[] en el full-review añadir el repasar la configuracion de agentes, skills, openspec en busca de duplicados, incongruencias con el código y cómo evitar que se consuman demasiados tokens
[] crear estructura de la base de datos y en full-review añadir que se mantenga actualizada
[] añadir datatables
[x] en los select hay dos opciones: 
    1. si el formulario se abre en modo create, en el select o select2 solo se mostrarán los que tengan active=1
    2. si el formulario se abre en modo edición, en el select o select2 se mostraran el elemento sseleccionado (tenga o no active=1) y el resto de elementos seran solo los que tengan acitive=1 
[] cachear listados select public function selectOptions y Observer para limpiar la cache ante cualquier modificación en el original
[] composer.json require y require-dev pasar a basics los que se pueda (debugbar no)
[] automatizar sentry <== claude begiratu, hor daude config file batzuk

# REVIEWS
[ ] repasa la apliación en busca de fallos y propon todas las mejoras que se te ocurran. escribe un task en .github
[ ] @full-review analiza demo13 basics13 customer13 projects13 y testing13 busca incongruencias entre los modelos, que tienen de diferente, que se gestiona de forma diferente en un modelo que en otro, que está testeado en uno si y en otro no, que código tienen repetido, donde hay sobreingenieria. escribe un task en .github 
[ ] skill fix-review .github/reviews/<fecha>/CODE-REVIEW.md antes de modificar pide confirmación y al terminar marca la tarea como realizada
[ ] @full-review  eres una experta en laravel, testing, seguridad y buenas practicas de programación. analiza demo13 basics13 customer13 y projects13 busca incongruencias entre los modelos, que tienen de diferente, que se gestiona de forma diferente en un modelo que en otro, que está testeado en uno si y en otro no, que código tienen repetido, donde hay sobreingenieria. 
[ ] Repasar los errores que da el SonarQube y buscar una posible solucion. Preguntar antes de cambiar nada

# PERFORMANCE
[ ] prepara un plan o acualiza el que ya existe que cree seeders para llenar las tablas con cientos de registros y luego analice el rendimiento del listado

# AGENTS
[ ] busca incongruencias entre lo que hay en el código y lo que hay dentro de la carpeta .github. muestrame una lista y vete preguntandome una por una, dame una posible solución y pide confirmación atnes de hacer nada 
[ ] analiza tambien el fichero how.md y comparalo con lo que hay en la carpeta .github y busca incongurencias. muestrame una lista y vete preguntandome una por una, dame una posible solución y pide confirmación atnes de hacer nada 
[ ] repasa la configuración de los AGENTES y analiza si hay alguna manera de ahorrar tokens y de mejorar la forma de trabajar.