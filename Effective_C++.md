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

## Item 19: Pensar el diseño de una clase como el diseño de un tipo

Este punto es difícil en cuanto requiere tiempo de reflexión y aconstumbramiento a un análisis exhaustivo de las clases definidas por el usuario. Algunas preguntas pertinentes son:

- Cómo deberían crearse y destruirse los objetos de nuestro nuevo tipo?
- Cómo debería diferir la inicialización de nuestro objeto de su asignación?
- Qué significa para los objetos de tu nuevo tipo que sean pasados por valor?
- Cuáles son las restricciones para instancias válidas de tu nuevo tipo?
- Encaja tu nuevo tupo en un grafo de herencia?
- Qué tipo de conversiones están permitidas para tu nuevo tipo?

**Conversiones implícitas vs explícitas**

Cuando tengo operadores de conversión o constructores de un parámetro, el compilador puede utilizarlos siempre que yo haga un llamado a una función que toma como parámetro una clase A y esta clase A puede ser construida con este tipo B. Esto puede arreglarse con el modificador `explicit` que obliga a construir primero A a partir de B y luego sí pasarlo por parámetro a la función. Evidentemente con constructores de 0 parámetros o más de 2 parámetros esto no es un problema.

- Qué operadores y funciones tienen sentido para el nuevo tipo?
- Qué funciones estándar (generadas por el compilador) deberían deshabilitarse?
- Quién debería tener acceso a los miembros de tu nuevo tipo?
- Qué garantías brinda este nuevo tipo con respecto a performance, manejo de excepciones, manejo de memoria?
- Qué tan general es tu nuevo tipo? Estás definiendo uno o un template de tipos?
- Es un nuevo tipo realmente lo que necesitás? Podrías reemplazarlo por agregar funcionalidad a un tipo existente?

## Item 20: Preferir pasaje por referencia a constante antes que pasaje por valor

Este punto es bastante directo. El pasaje por valor implica una copia por lo que al pasar por valor objetos grandes es difícil medir el costo de esta copia. El único caso general en que se recomienda pasar por valor es para los _built-in types_ y algunos tipos relacionados a la STL, ya que están fuertemente optimizados (no confundir con que la razón sea el tamaño).

Por otra parte, un problema que puede surgir de pasar por valor tiene que ver con el pasaje de una clase a un parámetro que tiene como tipo una clase base. Al construirse un objeto en el mismo llamado de la función, ocurre un _slice_ del objeto y perdemos información del objeto orignal (específicamente lo relacionado a la clase hija).

Como último comentario, no termino de entender el razonamiento en el siguiente fragmento:

_Si miramos lo que realiza el compilador, veremos que que la const reference es en realidad un puntero. **Como resultado**, es más eficiente para los built-in types pasar por valor que pasar por referencia._

**Resuelto**: las dos razones son que un puntero serían 8 bytes mientras que el int típicamente es de 4 bytes. Además, al pasar un puntero existe una mínima indirección contra tener directamente el valor.

## Item 21:

Evitar a toda costa los siguientes retornos en funciones:

- Puntero o referencia a un objeto local en el stack.
- Referencia a un objeto alocado en el heap.
- Puntero o referencia a un objeto estático local (excepto que sepa que realmetne necesito uno solo).

**Comentario**: y puntero a un objeto del heap? Entiendo que como poder, ser puede. Pero creo recordar que el libro diga algo de que es un mal diseño si le dejo al usuario la responsabilidad de liberar la memoria.

## Item 22: Declarar _private_ a los data members

- Declarar a los data members privados. Acceso uniforme, control de acceso atomizado, refuerza invariantes y da flexibilidad en la implementación.
- Recordar que **_protected_ no es más encapsulado que _public_**.

**Chequear el campo Handled de la clase Event de Cabrankengine**

## Item 23: Preferir funciones _no-friend no-miembro_ a funciones _miembro_

Vamos con algunos casos concrectos donde esta preferencia se manifiesta:

- Operadores simétricos (multiplicación, suma, igualdad).
- Funciones que involucran dos tipos o más y no hay un owner claro, una predominancia de alguno de estos tipos.
- Algoritmos auxiliares que solo interactúan con la interfaz pública de la clase.
- Testing o funcionalidad extendida de la clase.

## Item 24: Declarar _no-miembro_ funciones donde se debería aplicar conversión de tipo a todos sus parámetros

Recordar que lo opuesto de una función miembro es una función _no-miembro_, no una función _friend_.

Lo que dice el título. Si tu función neestia conversiones de tipo en todos los parámetros (incluyendo el _this_), la función debe ser _no-miembro_.

## Item 25: Considerar soporte para un _non-throwing_ swap

Básicamente el swap es una manera de sortear asignaciones por copia para intercambiar variables. Antes de C++11 era bastante necsario ya que no estaba optimizado más allá de los _built-in types_.

Ahora no pareciera ser tan necesario crear custom swaps para las clases definidas por el usuario. De todos modos, si mi clase Entity tuviese miembros costosos de copiar o manejase objetos dinámicos podría implementar un swap que ahorre copias por un lado y asegure un _no-throw_ por el otro.

# Implementaciones

## Item 26: Posponer la definición de una variable todo lo que se pueda

No hay mucho que comentar. Evidentemente no hay ninguna razón para no hacerlo y sí hay varias para sí hacerlo. Principalmente el hecho de que puede haber leaks de memoria si declaro uyna variable y antes de inicializarla ocurre una excepción.

## Item 27: Minimizar el uso de castings

- Evitar el uso de casteos, especialmente de `dynamic_cast` porque puede ser muy poco performante. En general, si se realiza un casteo habría que investigar una alternativa _cast-free_.
- Si vas a castear, mejor hacerlo dentro de una función y no exponerlo al cliente.
- Bajo ningún concepto utilizar _C-Style_ casts. No solo por lo vetusto sino porque **no sabemos** que tipo de cast se está aplicando en este caso.

Hagamos un repaso de los casts disponibles y su uso más común:

|Tipo|Cuándo|Concepto|Uso principal|
|-|-|-|-|
|`static_cast`|Compilación|Conversión definida|Conversiones numéricas|
|`reinterpret_cast`|Compilación|Conversión no definida|Interpretación de buffers|
|`const_cast`|Compilación|Remueve el modificador `const`|Conexión con APIs que sé que no modifican. Reutilización de código cuando tengo verisón const y no const|
|`dynamic_cast`|Ejecución|Downcasting|Castear de una clase base a una clase derivada|

## Item 28: Evitar devolver _handles_ a objetos internos

La idea es tratar de no devolver nunca referencias ni punteros a miembros de una clase. Por un lado (y evidentemente), evitar devolver estos no const porque podrían modificar el estado de mi objeto desde fuera rompiendo invariantes. Por otro lado y más sutilmente, si devuelvo alguna de estas aunque sea constante y mi objeto se destruye, el _handle_ que devolví queda inválido.

Si bien es verdad que en principio devolvería una copia, por un lado es posible que se realize un `std::move` que ahorre esto o incluso que el compilador realice un **RVO (return value optimization)** que consiste en construir el objeto directamente en la variable de retorno (seguría habiendo una construcción contra un move).

## Item 29: Siempre buscar un código _exception-safe_

La idea general de este punto es que las funciones que arrojen excepciones no deberían dejar al objeto que las contiene en un estado inválido. Tengo 3 alternativas de garrantía que puedo ofrecer en una API:

- **No-throw**: nunca lanza excepciones, la operación siempre es exitosa.
- **Strong guarantee**: si la operación falla, el estado del programa no cambió y es como si nada hubiera pasado.
- **Basic guarantee**: si la operación falla, se garantiza que el programa quede en un estado válido pero indeterminado.

**Tenes especial cuidado con alocación de memoria y construcción de objetos**

## Item 30: Entender a fondo el _Inlining_

- A veces puede generar un .obj más pequeño pero **no siempre**.
- Todas las funciones definidas en un header (incluidos los templates) o dentro de una clase son inline.
- Ninguna función virual puede ser inline.

## Item 31: Minimizar las dependencias de compilación entre archivos

Por un lado, hay una optimización posible en el desacople que produce declarar punteros a objetos en vez de tener los objetos en sí. Por otro lado, se habla de dos maneras de desacoplar la declaración de la implementación al momento de incluir:

- Patrón **Pimpl** (Pointer to implementation) con _Handle classes_.
- Clases interfaz en polimorfismo.

**Averiguar más sobre pros y contras de tener punteros en vez de los objetos en sí. Con punteros crudos entiendo que es mejor a costa de manejar la memoria. Con smart pointers capaz eso mejora pero el unique_ptr tiene su overhead no solo de implementación sino de tener que definir el delter pero capaz es más sencillo de lo que creo".**

# Herencia y Diseño Orientado a Objetos

## Item 32: Asegurarse que una herencia pública modela un "is-a"

Básicamente no hay que olvidar jamás al modelo de que una herencia pública indica que una clase derivada **es todo lo que es su clase base**.

## Item 33: Evitar esconder nombres heredados

- Esconder nombres **solo es posible en herencia pública** 
- Los nombres se esconden incluso aunque haya overload con distintos parámetros!!
- Se puede aplicar `Using::método` para no esconder los nombres heredados.

## Item 34: Diferenciar entre heredar una interfaz y heredar una implementación

- Bajo herencia pública, las clases derivadas **siempre** heredan interfaz, específicamente métodos virtuales puros.
- Los métodos virtuales impuros agregan una herencia de implementación por defecto.
- Los métodos no virtuales marcan una herencia de interfaz más una herencia obligatoria de implementación.

## Item 35: Considerar alternativas a funciones virtuales

Es importante recordar que existen ciertas alternativas:

- **Non-virtual interface idiom (NVI idiom)**: wrappea una función virtual con una no virtual más accesible.
- **Function Pointer data members**: cambio el comportamiento en runtime pasándole al objeto la función que debe llamar para el estado actual.
- **Strategy Pattern**: la idea es cambiar funciones virtuales en una jerarquía por funciones virtuales en otra.

Desde ya que la desventaja de considerar la alternativa de FPointers es que no tendríamos acceso a los miembros no públicos de la clase.

Creo que los puntos 2 y 3 hablan medio de lo mismo en C++ moderno: el uso de cualquier _callable_.

## Item 36: Nunca redefinir una función no virtual heredada

Se escondería la implementación original y el objeto derivado rompería la relación **is_a**.

## Item 37: Nunca redefinir el valor default de un parámetro en una función heredada

Esto sería un error porque los parámetros default están vinculados estáticamente mientras que las funciones virtuales lo están dinámicamente.

## Item 38: La composición debe modelar _has-a_ o _is-implemented-in-terms-of_

- Separar bien la idea de composición y de herencia pública.
- En el dominio de aplicación (diseño), la composición significa _has-a_. En el dominio de implementación, significa _is-implemented-in-terms-of_.

## Item 39: Usar herencia privada con juicio

- A diferencia de la herencia pública, esta significa _is-implemented-in-terms-of_. La idea es que podría se útil si quiero esconder la herencia y necesito acceder a algunos miembros protegidos en la clase base (o redefinir funciones virtuales).
- A diferencia de la composición, puedo permitir la **empty base optimization** (especialmente útil para desarrolladores de librerías que buscan minimizar tamaños de objetos).

## Item 40: Usar herencia múltiple de manera juiciosa

- Es más compleja que la simple y puede dar lugar a ambigüedades y a herencia virtual.
- Esto es costoso pero se puede amortizar si la clase base no tiene data.
- Un uso común es para combinar herencia pública de una clase interfaz con una herencia privada de una clase que solo me interesa la implementación.

# Templates y Programación Genérica

## Item 41: Entender interfaces implícitas y polimorfismo en tiempo de compilación

- Tanto clases como templates soportan interfaces y polimorfismo.
- Para las clases, **las interfaces son explícitas y se centran en signaturas de funciones**. El polimorfismo ocurre en tiempo de ejecución mediante funciones virtuales.
- Para parámetros templates, **las interfaces son implícitas y se basan en expresiones válidas**. El polimorfismo ocurre en tiempo de compilación mediante instanciación de templates y overloading de funciones.

## Item 42: Entender los dos significados de `typename`

- Cuando declaramos parámetros template, `class` y `typename` son intercambiables.
- Para identificar nombre de tipos anidados deebe utilizarse `typename`, excepto en un identificador de clase base en una _initialization list_.

## Item 43: Saber cómo acceder a nombres en clases base templatizadas

En clases template derivadas, referirse a nombres de la clase base template usando el prefijo `this->`, utilizando la declaración con `using` o con una califiación de la clase base explícita.

## Item 44: Factorizar código independiente de parámetros fuera de los templates


