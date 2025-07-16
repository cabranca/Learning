# Window Abstraction and GLFW

- Usa GLFW para la abstracción. La idea es usarlo porque soporta OpenGL que es por ahora el renderizado que va a usar. Crea una carpeta por plataforma, próximamente una carpeta por renderizador.
- Agrega GLFW como submódulo de su repo y lo buildea con un Premake. Luego tiene que agregar el premake de GLFW al premake de Hazel. No es un workspace ni un solution. Solo es un proyecto que se compila de manera estática.
- Crea una clase abstracta Window que funciona de interfaz para luego implementar en cada plataforma. Tiene un eventCallbackFn a ver en el próximo video.
- Agregar Asserts a su file de macros. Por ahora lo usa para chequear errores de librerías como glfw. No se pierde performance porque ni se buildea en las config que no lo incluyamos.