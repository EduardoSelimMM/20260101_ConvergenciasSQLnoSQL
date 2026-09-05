# Sesión 5

La sesión de hoy sábado 5 de septiembre será asíncrona. Consistirá en revisar el video 🎥

📹📹 https://youtu.be/yrxCZLAN63E?si=kSiHEKQc5F5nCrUg 📹📹

y el sitio web

💻💻 https://www.tigerdata.com/learn/how-to-query-jsonb-in-postgresql 💻💻

Posteriormente responder las suguientes preguntas:

1. ¿Cómo se realiza el casting de una cadena de texto a un objeto JSONB en SQL?

2. ¿Qué función cumple el operador de concatenación (`||`) al trabajar con documentos JSONB?

3. Si ejecutas una sentencia `UPDATE` asignando directamente una nueva cadena `::jsonb` a una columna sin usar operadores como `||`, ¿qué le sucede al documento guardado anteriormente?

4. ¿Cómo se utiliza el operador de resta (`-`) para modificar la estructura de un objeto JSONB?

5. ¿Para qué sirve la función `jsonb_set` y qué parámetros requiere para modificar un valor específico sin reemplazar todo el objeto?

6. ¿Por qué falla una consulta que intenta encadenar un operador flecha simple inmediatamente después de haber usado un operador flecha doble? Por ejemplo: `data ->> 'reading' -> 'unit'`

7. ¿Cómo se extrae un elemento específico dentro de un arreglo JSON utilizando los operadores flecha (`->` o `->>`)?

8. ¿Qué diferencia existe entre el operador de ruta `#>` y el operador `#>>` al navegar por estructuras anidadas complejas?

9. Dado `'{device, manufacturer, company_name}'`, ¿cómo extraería la información el operador `#>`?

10. ¿Para qué se utiliza el operador de contención `@>` y qué tipo de datos o estructura requiere a su derecha?

11. ¿Cómo funciona el operador `?` y cuál es su principal limitación con respecto a las llaves que se encuentran anidadas?

12. ¿Qué diferencia existe entre el comportamiento del operador `?|` y el del operador `?&` cuando se les pasa un arreglo de texto como argumento?

13. Si quieres verificar la existencia de una llave anidada (por ejemplo, `company_name` dentro de `manufacturer`), ¿qué combinación de operadores usarías?

14. ¿Por qué el operador `?` devuelve `false` si buscas un valor en lugar de una llave en el nivel superior de un objeto?

15. ¿Qué resultado devuelve la función `jsonb_each()` y cómo puedes utilizar los alias `key` y `value` para consultar sus resultados?

16. ¿Cuál es la utilidad principal de la función `jsonb_object_keys()`?

17. ¿Para qué sirve la función `jsonb_pretty()` y en qué casos lo consideras útil?

18. ¿Qué tipo de resultado devuelve la función `jsonb_typeof()` y qué posibles valores puede regresar?

19. En cuanto al rendimiento y las búsquedas en grandes volúmenes de datos, ¿por qué el uso de filtros con el operador de contención (`@>`) tiene ventajas frente al uso de comparaciones con `->>` en consultas sobre columnas JSONB indexadas?

20. ¿Qué sentencia SQL usarías para crear una tabla `sensor_devices` con una columna `id` como llave primaria y una columna `data` de tipo JSONB?

21. Si quisieras guardar un JSON con las claves `device_name`, `active` y `temperature_readings` (un array de números) en la columna `data`, ¿qué sentencia `INSERT` escribirías, incluyendo el cast necesario para convertir el texto a JSONB?

22. Supón que un documento JSONB tiene la clave `"reading": {"temperature": 22.5, "unit": "C"}`. ¿Qué expresión usarías para extraer el objeto completo asociado a `reading` PERO sin convertirlo a texto?

23. Sobre ese mismo documento, ¿qué expresión, encadenando operadores, extraería el valor de `unit` directamente como texto plano?

24. Explica con tus propias palabras, y mostrando ambas expresiones. ¿Por qué `data ->> 'reading' -> 'unit'` devuelve un error? ¿Por qué que `data -> 'reading' ->> 'unit'` sí funciona?

25. Si quisieras agregar la clave `"location": "Room 2"` a un documento JSONB que ya existe, sin borrar las claves que ya tiene, ¿qué operador usarías y cómo se vería la sentencia `UPDATE`?

26. ¿Qué sentencia `UPDATE` usarías para eliminar la clave `location` de un documento?

27. Si un array `temperature_readings` dentro de un documento JSONB es `[22.5, 23.0, 21.8]`, ¿qué expresión devolvería el tercer elemento como texto plano?

28. ¿Qué expresión usarías con el operador `@>` para comprobar si un documento JSONB contiene la pareja `"active": false`?

29. ¿Cómo escribirías una condición equivalente a la anterior, pero usando `->>` y una comparación de igualdad en lugar de `@>`?

30. ¿Qué expresión usarías con el operador `?` para comprobar si la clave `manufacturer` existe en el nivel superior de un documento JSONB?

31. Si `manufacturer` es un objeto anidado que contiene la clave `company_name`, ¿qué combinación de operadores (-> junto con ?) usarías para comprobar si company_name existe dentro de manufacturer? ¿Por qué no basta con usar ? directamente sobre el documento completo?

32. ¿Qué expresión usarías con el operador `?|` para comprobar si un array `device_tags` contiene al menos uno de los valores `'wifi'` o `'bluetooth'`?

33. Dado un documento de la forma `{"device": {"manufacturer": {"company_name": "TechDevices"}}}`, ¿qué expresión con el operador `#>` y una ruta de tres niveles extraería el valor de `company_name`?

34. ¿Cómo se vería la misma extracción del punto anterior, pero usando la función `jsonb_extract_path` en lugar del operador `#>`?
