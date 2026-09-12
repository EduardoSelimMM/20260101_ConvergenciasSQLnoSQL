+ Las 11 operaciones básicas create-table, list-tables, describe-table, delete-table, update-table, put-item, get-item, update-item, delete-item, query, scan son las que hay que aprenderse para trabajo normal del día a día

# "Convergencias" entre DynamoDB y SQL

+ PartiQL es un lenguaje de consulta compatible con SQL que facilita la consulta eficiente de datos en DynamoDB mediante las sentencias DML (Data Manipulation Language) `SELECT`, `INSERT`, `UPDATE` y `DELETE`, i.e. manipular datos dentro de una estructura ya existente

+ ¿Por qué? ¿Para qué?

+ Para poder utilizar un lenguaje conocido para comenzar a trabajar con DynamoDB. No necesitas conocer **completamente** el lenguaje de consultas propio de DynamoDB... Por supuesto, esto tiene sus "asegunes"

+ Puedes utilizar PartiQL si no quieres trabajar directamente con `FilterExpressions`

```
aws dynamodb get-item --table-name Clientes --key '{"clienteId": {"S": "C001"}}'
```

+ Esto ⬇️ es 100% PartiQL, PEEEEEERO no puedes meterle `filter-expression` de sintaxis nativa

```
aws dynamodb execute-statement --statement "SELECT * FROM \"Clientes\" WHERE clienteId = 'C001'"
```

+ PartiQL fue agregado en 2020, como una "capa de conveniencia" dado el capitalismo voraz... intentar darle "poderes" SQL a una base NoSQL

+ La forma "nativa" y original de DynamoDB desde el principio nunca fue un lenguaje de consultas tipo SQL, sino un conjunto de operaciones de API, cada una con su propio formato JSON

+ PartiQL no le agrega ninguna capacidad nueva a DynamoDB

+ Cuando Postgres agregó JSONB, sí ganó una capacidad real que no tenía antes: guardar y consultar estructuras flexibles con índices propios, funciones de extracción, y se sigue pudiendo usar `JOIN`, `GROUP BY`, agregaciones SQL completas incluso sobre esos datos JSONB

+ PartiQL, en cambio, no le da a DynamoDB ninguna capacidad que no tuviera ya

+ Por debajo, cada `SELECT` de PartiQL se traduce exactamente a uno de los mismos 3 movimientos que ya se tenían desde antes de que existiera PartiQL: `GetItem`, `Query`, o `Scan`

+ PartiQL es puro maquillaje de sintaxis, cambia cómo se ve lo que escribes, no lo que la base de datos puede hacer

+ Entonces la frase "forzar a DynamoDB a parecer SQL" es mucho más superficial que la frase "forzar a postgreSQL a parecer MongoDB"

+ No estamos en arenas movedizas acá, pues no hay una tensión filosófica profunda

+ PartiQL no está intentando que DynamoDB haga cosas para las que no fue diseñado. Es sólo vocabulario distinto para las mismas operaciones limitadas de siempre

## Las 4 sentencias DML (i.e. las operaciones principales)

| Sentencia | Qué hace | Ejemplo |
|---|---|---|
| `SELECT` | Leer items | `SELECT * FROM "Clientes" WHERE clienteId = 'C001'` |
| `INSERT` | Crear un item nuevo | `INSERT INTO "Clientes" VALUE {'clienteId': 'C999', 'nombre': 'X'}` |
| `UPDATE` | Modificar campos de un item que ya existe | `UPDATE "Productos" SET stock = 0 WHERE productoId = 'P001'` |
| `DELETE` | Borrar un item | `DELETE FROM "Clientes" WHERE clienteId = 'C999'` |

+ Junto con Los operadores de comparación (estos sí son como en SQL vainilla)

```
=   <>   <   <=   >   >=
BETWEEN ... AND ...
IN (...)
AND   OR   NOT
```

+ Además hay do operadores de navegación (**OJO:** No son funciones, son símbolos)

| Símbolo | Para qué sirve | Ejemplo |
|---|---|---|
| `.` | Entrar a un **Map** (objeto anidado) | `preferencias.direccion.ciudad` |
| `[n]` | Entrar a una **List** (arreglo), por posición | `items[0].productoId` |

+ Se usa la misma lógica que en JavaScript o Python: objeto.propiedad para diccionarios, lista[0] para arreglos por posición.
	+ `.` (punto) → para entrar a un mapa (un objeto anidado, como preferencias o metadata)
 	+ `[]` (corchetes cuadrados con un número) → para entrar a una lista (un arreglo, como items o tags)

+ 🚨🚨 **Importante:** 🚨🚨 No son operaciones "nuevas" que antes no se hayan podido hacer nativamente en DynamoDB

| Función | Qué responde | Ejemplo |
|---|---|---|
| `attribute_exists(ruta)` | ¿Existe este campo? | `WHERE attribute_exists(metadata.cupon.codigo)` |
| `attribute_not_exists(ruta)` | ¿NO existe este campo? | `WHERE attribute_not_exists(metadata.cupon)` |
| `contains(lista, valor)` | ¿El arreglo/texto contiene este valor? | `WHERE contains(atributos.tags, 'oferta')` |
| `begins_with(campo, texto)` | ¿El texto empieza así? (útil en sort keys) | `WHERE begins_with(pedidoId, 'O00')` |
| `size(ruta)` | Tamaño de un arreglo, mapa, o string | `WHERE size(atributos.tags) > 1` |

## Veamos unos ejemplitos

### Traer todos los clientes
```
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Clientes\""
```

### Traer un cliente exacto por su llave (GetItem por debajo)
```
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Clientes\" WHERE clienteId = 'C001'"
```

### Query eficiente usando el índice ClienteIndex

+ Por ejemplo, los pedidos de un cliente
```
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Pedidos\".\"ClienteIndex\" WHERE clienteId = 'C005'"
```

### Scan con filtro

+ Por ejemplo, productos de una categoría
```
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Productos\" WHERE categoria = 'Electrónica'"
```

### Scan

+ Por ejemplo, pedidos por estado
```
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Pedidos\" WHERE estado = 'pendiente'"
```

### Atributo anidado (mapa dentro de mapa)
```
aws dynamodb execute-statement \
  --statement "
    SELECT nombre, preferencias.direccion.ciudad
    FROM \"Clientes\"
    WHERE clienteId = 'C001'
  "
```

### Elemento de una lista

+ Por ejemplo, el primer producto dentro de un pedido
```
aws dynamodb execute-statement \
  --statement "
    SELECT items[0].productoId, items[0].cantidad
    FROM \"Pedidos\"
    WHERE pedidoId = 'O001'
  "
```

### Función contains()

+ Por ejemplo, productos con el tag "oferta"
```
aws dynamodb execute-statement \
  --statement "
    SELECT nombre
    FROM \"Productos\"
    WHERE contains(atributos.tags, 'oferta')
  "
```

### Función attribute_exists()

+ Por ejemplo, pedidos que sí tienen cupón
```
aws dynamodb execute-statement \
  --statement "
    SELECT pedidoId
    FROM \"Pedidos\"
    WHERE attribute_exists(metadata.cupon.codigo)
  "
```

### Insertar un cliente nuevo
```
aws dynamodb execute-statement \
  --statement "
    INSERT INTO \"Clientes\"
    VALUE {
      'clienteId': 'C999',
      'nombre': 'Cliente CLI',
      'ciudad': 'CDMX'
    }
  "
```

### Actualizar el stock de un producto
```
aws dynamodb execute-statement \
  --statement "
    UPDATE \"Productos\"
    SET stock = 0
    WHERE productoId = 'P001'
  "
```

### Borrar el cliente de prueba que insertamos en el paso 10
```
aws dynamodb execute-statement \
  --statement "
    DELETE FROM \"Clientes\"
    WHERE clienteId = 'C999'
  "
```

### ¿Cómo apuntar a un índice secundario?

```
aws dynamodb execute-statement \
  --statement "
	SELECT * FROM "Pedidos"."ClienteIndex" WHERE clienteId = 'C005'
  "
```

+ Acá hay dos nociones de "vacío"

| Expresión | Pregunta |
|---|---|
| `campo IS NULL` | ¿El campo existe pero su valor es nulo? |
| `campo IS MISSING` | ¿El campo ni siquiera existe en este ítem? |


+ 👀**OJO:**👀 Ver sintaxis de la forma `"Tabla"."NombreDelIndice"`.... hay dos nombres entre comillas dobles seguidos

+ 💔 No hay `JOIN`... cada consulta de PartiQL trabaja sobre una sola tabla, nunca combina varias.

+ 💔 No hay agregaciones `SUM`, `AVG`, `COUNT` de grupo, etc

	+ Es decir, PartiQL es un lenguaje de CRUD, no un lenguaje de análisis... a pesar de que su sintaxis (`SELECT`) sugiere lo contrario y nos tienta a esperar el "poder completo" de SQL

+ 💔 No hay `CREATE TABLE`, `ALTER TABLE`

+ 💔 No hay `GROUP BY` ni `HAVING`

+ ❌ Las cláusulas `LIMIT`, `GROUP BY` y `HAVING` no son compatibles con las sentencias `SELECT` de PartiQL

+ 💔 No hay `subqueries` i.e. no puedes anidar un `SELECT` dentro de otro... algo súper súper súper común en SQL (el relacional de todos los días)

+ 💔 `ORDER BY` sólo funciona en casos muy específicos. Generalmente necesitas tener ya un `WHERE` sobre la llave de partición

	+ Si intentas ordenar sin eso, se obtiene un error de validación que dice que debe existir una cláusula `WHERE` en la sentencia cuando se usa `ORDER BY`

	+ Básicamente porque al ser un `scan`, no hay forma posible de tener `ORDER BY` sin ese `WHERE`, ya que DynamoDB no ordena las filas al leerlas de varias particiones y PartiQL no hace ningún procesamiento adicional sobre el resultado

+ PartiQL no es lo suficientemente "inteligente" como para elegir un índice secundario cuando el `WHERE` filtra sobre la clave de ese índice

	+ Se tiene que especificar directamente el índice secundario en la cláusula `FROM` para poder lograrlo

+ Hay PartiQL para Amazon Redshift, AWS TwinMaker, AWS Quantum Ledger DB... pero me vuelve a meter en arenas movedizas y se sale de mi área de experiencia

+ Cada implementación/motor/producto decide qué subconjunto del lenguaje completo va a soportar, según lo que su implementación/motor/producto puede ejecutar eficientemente

+ Donde la cosa ya se vuelve un mounstro (en mi opinión)

+ Varias sentencias PartiQL en una sola llamada

```
aws dynamodb batch-execute-statement \
  --statements '[
    {"Statement": "SELECT * FROM \"Clientes\" WHERE clienteId = '"'"'C001'"'"'"},
    {"Statement": "SELECT * FROM \"Productos\" WHERE productoId = '"'"'P001'"'"'"}
  ]'
```
 + Varias sentencias PartiQL como transacción (todas o ninguna)

```
aws dynamodb execute-transaction \
  --transact-statements '[
    {"Statement": "UPDATE \"Productos\" SET stock = stock - 1 WHERE productoId = '"'"'P001'"'"'"}
  ]'
```
### PartiQL editor

+ Hay un editor para tirar código PartiQL

```
SELECT * FROM "Clientes";


SELECT * FROM "Clientes" WHERE clienteId = 'C001';


SELECT * FROM "Pedidos"."ClienteIndex" WHERE clienteId = 'C005';


SELECT * FROM "Productos" WHERE categoria = 'Electrónica';


SELECT * FROM "Pedidos" WHERE estado = 'pendiente';


SELECT nombre, preferencias.direccion.ciudad
FROM "Clientes"
WHERE clienteId = 'C001';


SELECT items[0].productoId, items[0].cantidad
FROM "Pedidos"
WHERE pedidoId = 'O001';


SELECT nombre
FROM "Productos"
WHERE contains(atributos.tags, 'oferta');


SELECT pedidoId
FROM "Pedidos"
WHERE attribute_exists(metadata.cupon.codigo);


INSERT INTO "Clientes"
VALUE {
    'clienteId': 'C999',
    'nombre': 'Cliente de Prueba',
    'ciudad': 'CDMX'
};
 

UPDATE "Productos"
SET stock = 0
WHERE productoId = 'P001';
 

DELETE FROM "Clientes"
WHERE clienteId = 'C999';
```
