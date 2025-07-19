# Constructores, Destructores y Operadores de Asignación

## Item 12: Copiar todas las partes de un objeto

Si defino un constructor por copia o el operador de asignación, debo asegurarme de que todos los miembros del objeto **y de su clase base** sean copiados.

Si la clase base tiene función de copiado, llamarla dentro de la clase hija.

**No llamar al constructor por copia base dentro de la asignación** aunque sea posible, ni viceversa.

En caso de tener un constructor y operador que hace prácticamente lo mismo, se estila refactorizar en un método privado `init`.

# Manejo de memoria

Siempre que termino de usar un recurso tengo que devolvérselo al sistema. Algunos ejemplos comunes de recursos son:

- Memoria
- File descriptors
- Mutex locks
- Fonts y Brushes en GUI
- Database connections
- Network sockets

## Item 13: Usar objetos para gestionar recursos

La idea básicamente es no confiar en que el _usuario_ libere correctamente y de forma manual el recurso. Para evitar esto, es mejor aplicar **RAII** (Resource Acquisition Is Initialization), es decir que warpeo el recurso en un objeto que se destruya automáticamente y en el proceso devuelva el recurso al sistema.

Muy importante en este sentido recordar el **Item 8** y no tirar excepciones en destructores.

## Item 14: Pensar bien el comportamiento de copiado en clases de gestión de recursos

Si lo que estoy usando es una clase custom para implementar RAII, debo optar por uno de estos comportamientos a la hora de copiar el objeto:

- **Prohibir el copiado**: lo uso cuando no tiene sentido el copiado (por ejemplo con mutex). Aplico lo aprendido en el **Item 6** y declaro privadas las operaciones de copiado.
- **Aplicar un conteo de referencias**: puedo incluso utilizar un `shared_ptr` como miembro de mi clas pero teniendo cuidado de especificarle el destructor que quiero para cuando el conteo de referencias llegue a 0.
- **Copiar el recurso**: si puedo tener copias y mi única preocupación es que se liberen, puedo realizar un _deep copy_ así como lo hace la clase _String_, copiando toda la cadena de caracteres a una nueva dirección en el heap.
- **Trasnferencia del ownership**: si queiro asegurarme que no existan copias de este recurso bajo ningún aspecto, tengo que definir mis propias operaciones de copia donde le cedo el recurso a la copia e invalido el original.

## Item 15: Proveer aceso a recursos _raw_ en clases de gestión de recursos

La idea es que usualmente APIs externas requieren que les pasemos los recursos crudos, por lo que deberíamos poder exponerlos de alguna manera en nuestra clase de gestión de recursos. A priori tenemos dos maneras distintas y elegimos una u otra dependiendo del contexto.

- **Conversión explícita**: como el `get()` de los smart pointers, más difícil de equivocarse pero más trabajoso para el usuario.
- **Conversión implícita**: un operador como `()` o `*` puede ser más intuitivo pero más confuso y propenso a errores ya que no está claro el tipo de la variable.

## Item 16: Ser consecuente en los usos de _new_ y _delete_ para un mismo recurso

La idea es tener cuidado al momento de llamar a _delete_ en caso de que el objeto a destruir haya sido creardo como array. Como la declaración de un puntero puede tomar como valor la direcicón del array, es posible tener un leak de memoria si al momento de liberar la memoria hacemos `delete` en vez de `delete []`.

## Item 17: Almacenar objetos en Smart Pointers en una sola línea aparte

Hay un error muy sutil que puede surgir a la hora de crear un smart pointer y es a la hora de llamar en una misma línea a la alocación de memoria, la construcción del smart pointer y capaz un llamado a otra función.

Por ejemplo una función que toma 2 parámetros, un smart pointer y un float. Si construyo el smart pointer en el argumento y el float es el resultado de un llamado a otra función, esta otra función puede tirar una excepción y el compilador **no me garantiza** que los argumentos se ejecuten en el orden en que los escribo, por lo que la memoria alocada podría no llegar a almacenarse en el smart pointer y por ende podría no liberarse.

# Diseños y Declaraciones

## Item 18: Hacer interfaces fáciles de usar correctamente y difíciles de usar incorrectamente

Por un lado debemos asumir que el usuario va a intentar utilizar nuestra interfaz de la manera correcta, por lo que de haber un fallo el error resdiría en la interfaz.

Por otro lado, si el código no hace lo que el usuario espera,no debería compilar directamente. Y si compila, debería hacer exactamente lo que el usuario espera.

Una primera estrategia es hacer uso de nuevos **tipos de variables** definidas por nosotros (idealmente clases pero structs valen también). Al definirlos, nosotros decidimos qué instancias son válidas y qué instancias no lo son.

Estos tipos de variables deberían tener las mismas reglas que los _built-in types_ en cuanto a operadores, modificadores const, etc. También deberían ser consistentes con otras interfaces como las de STL (ej: que el tamaño de un container se consiga con `size()`).

Muy clave poder reducir la responsabilidad del usuario de manejar memoria. Entran en juego los muy útiles smart pointers.
