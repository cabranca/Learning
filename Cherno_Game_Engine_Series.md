# [Window Abstraction and GLFW](https://youtu.be/88dmtleVywk?list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT)

- Usa GLFW para la abstracción. La idea es usarlo porque soporta OpenGL que es por ahora el renderizado que va a usar. Crea una carpeta por plataforma, próximamente una carpeta por renderizador.
- Agrega GLFW como submódulo de su repo y lo buildea con un Premake. Luego tiene que agregar el premake de GLFW al premake de Hazel. No es un workspace ni un solution. Solo es un proyecto que se compila de manera estática.
- Crea una clase abstracta Window que funciona de interfaz para luego implementar en cada plataforma. Tiene un eventCallbackFn a ver en el próximo video.
- Agregar Asserts a su file de macros. Por ahora lo usa para chequear errores de librerías como glfw. No se pierde performance porque ni se buildea en las config que no lo incluyamos.

# [Window Events](https://youtu.be/r74WxFMIEdU?list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT)

- Llamo al método de Window::SetEventCallback() y le paso uno de los callbacks que tengo para los eventos.
- Setea el callback de GLFW y el User Pointer para en cada callback recuperar esa info y forwardear al callback de Window.
- Setea también un error callback.
- Macro para el bind del callback. -> Se puede cambiar por una lambda más expresiva. La contra sería que puede ser muy verboso si la fn tiene muchos parámetros.

# [Layers](https://youtu.be/_Kj6BSfM6P4?list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT)

- Aunque podemos partir de la idea de **layer** como en Photoshop, es decir capas superpuestas y ordenadas que definen una manera de mostrar objetos en pantallas, también podemos aplicar esta lógica de layers a eventos o al update de los objetos.
- Para enventos, es común recorrer desde la capa más externa a la más interna siempre y cuando el evento no se marque como _handled_.
- Para render y update la idea es ir de la capa más interna a la más externa. En el renderizado la decisión es bastante obvia, para asegurarme que algo se vea en pantalla tengo que renderizarlo en último lugar.
- Una utilidad de pensar en layers es tener una para debug que solo se utilice en las compilaciones correspondientes y que siempre se van a dibujar a lo último.
- Agrega una clase ``Layer``  como interfaz. Tiene un nombre para debug y callbacks como `onAttach` para ser agregado al LayerStack o `onEvent` para darle la oportunidad de manejar el evento. En el futuro podría tener un bool para activar/descativar cada layer para que no actualice ni renderice.
- Agregar una clase `LayerStack` que funcione como un wrapper de un vector de layers. Aunque se llame Stack, se pueden insertar en cualquier lugar por lo que se guarda un iterador como miembro de la clase. Funciona como owner de Layer y por ende maneja la memoria. Por lo pronto solo borra layers en la destrucción del stack, por lo que realizar un pop no lo elimina. Hace una separación entre layers y overlays pero los mantiene en una misma estructura, garantiza que siempre las layers se pusheen por debajo de los overlays.
- Agrega este LayerStack a la clase Application. En ``onEvent`` forwardea este evento al layer stack y en `run` llama al `Layer::update()`.
- Realiza un cambio en premake porque terminaba con distintos heaps entre la dll y Sandbox, por lo que tiene que linkear contra `Multi-threaded Debug`.

# [Modern OpenGL (Glad)](https://youtu.be/HFyHIc89z1g?list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT)

- Antes de introducir ImGUI explica un poco del funcionamiento de Glad. Básicamente es un poco más moderno que Glew pero se pueden usar de manera indistinta.

# [Intro to Profiling](https://youtu.be/YbYV8rRo9_A?list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT)

- Referencia a video de benchmarking anterior en el canal
- La idea es poder sacar métricas y poder visualizarlas, que el profiler sea independiente de VS. El profiler de VS no está mal pero la idea es analizar el código de manera agnóstica, sin tener en cuenta el entorno de trabajo en que se ejecuta.
- Uso una clase Timer que usando `std::chrono` que arranca en la construcción y se para en la destrucción. En la destrucción calcula el tiempo que pasó y lo loggea por consola (por qué no con el logger?).
- Usa `const char*` para el nombre y prefiere que no sea `std::string` porque probablemente sea estático y no quiere todo el overhead de usar un string.
- Le pasa al timer el nombre de la función.
- Puedo medir el tiempo de un bloque específico dentro de una función creando la variable en el scope del bloque.
- Como el log en consola es mucho para mirar, la idea es usar ImGUI para representarlo en pantalla.
- Por ahora está agregando una lambda al constructor de Timer para luego llamar a una función que devuelva los resultados para que los use ImGUI.
- Define una macro para hacer el llamado al timer y no escribir toda la lambda. No entiendo por qué llama a la lambda desde Timer con 2 parámetros si solo le definió uno.