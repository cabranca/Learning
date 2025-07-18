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