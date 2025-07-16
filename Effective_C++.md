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


