# 1. The Basics

## 1.6 Constants

- Evaluar el uso de ``consteval`` en vez de ``constexpr``.
- Como curiosidad, las funciones consteval/constexpr son las **funciones puras** de C++.

## 1.8 Tests

- Me acabo de dar cuenta que en un _switch_ puedo omitir el ``break`` para evaluar varios casos con la misma lógica.
- Uso muy poco la creación de variables dentro de ifs.

# 4. Error Handling

## 4.5 Assertions

- No tenía la idea de usar assertions como chequeos opcionales en runtime **para testear invariantes y precondiciones**. En mi cabeza lo tenía como prevenir _errores del programador_ en contraposición a los errores de usuario.

# 5. Classes

## 5.2 Concrete Types

- Estaría bueno estar atento a cuando un método no necesita acceso directo a miembros privados. En ese caso podría pasar a ser una función no miembro que use la interfaz pública de la clase.
- _"Try to use unchecked casts only for the lowest level of a system. They are error-prone."_ - Es una recomendación o un challenge? :p
- Tener en cuenta `bit_cast` como equivalente a castear a **void***.
