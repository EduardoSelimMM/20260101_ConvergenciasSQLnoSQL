# Levantar base de datos DynamoDB

+ Antes de levantar la base, demos algunos detalles generales sobre DynamoDB

+ Entre las instrucciones se irán dando algunas otras características

+ **Amazon DynamoDB** es un servicio de base de datos **NoSQL** con modelo **clave-valor** totalmente administrado por AWS

+ Promete tener latencias de un solo dígito de milisegundo a cualquier escala

+ Escala horizontalmente de forma automática para manejar muchísimas peticiones por segundo sin degradación en el tiempo de respuesta

+ Es lo que se conoce como **serverless**... ¿a qué les suena?

+ Me gusta más decir que es **totalmente administrada**... por AWS, no por tí

+ Esto quiere decir que no requiere aprovisionar ni administrar servidores. AWS coordina el escalado, los parches de seguridad, las copias de seguridad, la replicación, etc.

+ Replica automáticamente los datos en al menos tres zonas de disponibilidad dentro de una región de AWS

+ Aunque inicialmente estaba pensada para estructuras de datos de clave-valor, dicha estructura es tan flexible que se dice que también recibe documentos JSON

## Modelo de datos básico

+ Items (elementos): Un registro dentro de la tabla (equivalente a una "fila" en SQL o un documento en MongoDB/DocumentDB

+ Tablas (tables): Colección de ítems

+ Atributos: Datos individuales asociados a un elemento (equivalente a un campo/columna).

+ Claves primarias: Es la característica definitoria (en mi opinión)

	+ Existe un **partition key** (en formato HASH) que determina la partición física donde se almacena el elemento

	+ Existe una **sort key** que, aunque es opcional, permite ordenar elementos dentro de la misma partición para realizar consultas por rangos

+ Vamos a empezar a levantar la base...

## Paso 0: Entrar al Learner Lab

## Paso 1: Crear una tabla

1. Busca DynamoDB en la barra de búsqueda
2. Menú izquierdo → Tables → Create table

+ Table name: `Clientes`
+ Partition key: `clienteId`
+ Tipo: `String`
+ Deja "sort key" sin marcar, y en "Table settings" deja "Default settings"
+ Create table → espera a que diga Active.

## Paso 2: Agregar un par de items "a mano"

1. Con la tabla `Clientes` ya creada, haz click sobre su nombre para entrar a sus detalles
2. Pestaña: Explore table items.
3. Botón Create item.
4. Puedes usar la vista "form" (llenar campos con botones "Add new attribute") o cambiar a vista JSON (más rápido)
5. Selecciona JSON y pega esto:

```
{
  "clienteId": {"S": "C001"},
  "nombre": {"S": "Carlos Ruiz"},
  "ciudad": {"S": "Monterrey"}
}
```
Repite el proceso con el siguiente item

```
{
  "clienteId": {"S": "C002"},
  "nombre": {"S": "Ana Lopez"},
  "ciudad": {"S": "Puebla"}
}
```

+ A nivel de la API interna de AWS, DynamoDB requiere que especifiques explícitamente el tipo de dato para cada atributo mediante descriptores

+ `S` para string, `N` para número, `BOOL` para booleano, `M` para map, `L` para list, `SS`/ `NS` para string set / number set (que básicamente so conjuntos de valores únicos), etc.

+  De entrada aparecen tipos de datos "no tradicionales": map, list, string set, number set ...

```
{
  "UserId": { "S": "USR-10294" },
  "Email": { "S": "alex@example.com" },
  "Age": { "N": "29" },
  "IsActive": { "BOOL": true },
  "Roles": { "L": [ { "S": "admin" }, { "S": "developer" }, { "N": "189" } ] },
  "Category": {"SS": ["A", "B", "C"]},
  "Address": {
    "M": { "City": { "S": "CDMX" }, "ZipCode": { "N": "01000" } }
  }
}
```

+ **OJO:** También se pueden definir los items en formato "unmarshalled JSON", pero de eso hablaremos más adelante...

+ Hay un límite de tamaño. El tamaño máximo de un solo item es de 400 KB (incluyendo la longitud del nombre del atributo y el valor)

+ Salvo por los atributos que forman la clave primaria (partition key y opcionalmente sort key), dos items en la misma tabla pueden tener atributos completamente diferentes

## Paso 3: Poblar la base de datos

Como antes, poblaremos la base con un script de Python que usa la librería {faker} para generar datos artificiales con una estructura deseada

1. Click en el ícono de terminal (`>_`) en la parte superior (a.k.a. abrir el CloudShell de AWS)
2. Instalar la librería de Python, faker

```
pip install faker
```

3. Copia el contenido del siguiente chunk

```
import random
import time
import boto3
from decimal import Decimal
from faker import Faker

REGION = "us-east-1"
NUM_CLIENTES = 30
NUM_PRODUCTOS = 25
NUM_PEDIDOS = 80
SEMILLA = 42

random.seed(SEMILLA)
Faker.seed(SEMILLA)
fake = Faker("es_MX")

CATEGORIAS = ["Electrónica", "Muebles", "Papelería", "Accesorios", "Hogar"]
METODOS_PAGO = ["tarjeta", "paypal", "transferencia"]
ESTADOS_PEDIDO = ["entregado", "pendiente", "cancelado"]
CANALES = ["web", "app"]
CUPONES = [None, {"codigo": "DESC10", "descuentoPct": 10}, {"codigo": "BIENVENIDA", "descuentoPct": 15}]

dynamodb = boto3.resource("dynamodb", region_name=REGION)
client = boto3.client("dynamodb", region_name=REGION)


def crear_tabla(nombre, partition_key, gsi=None):
    """Crea una tabla en modo On-Demand (sin preocuparse por throughput)."""
    existentes = client.list_tables()["TableNames"]
    if nombre in existentes:
        print(f"  Tabla '{nombre}' ya existe, se borra para empezar limpio...")
        client.delete_table(TableName=nombre)
        client.get_waiter("table_not_exists").wait(TableName=nombre)

    attr_defs = [{"AttributeName": partition_key, "AttributeType": "S"}]
    key_schema = [{"AttributeName": partition_key, "KeyType": "HASH"}]

    kwargs = {
        "TableName": nombre,
        "AttributeDefinitions": attr_defs,
        "KeySchema": key_schema,
        "BillingMode": "PAY_PER_REQUEST",  # On-demand: no gestionas capacidad
    }

    if gsi:
        gsi_attr, gsi_name = gsi
        kwargs["AttributeDefinitions"].append({"AttributeName": gsi_attr, "AttributeType": "S"})
        kwargs["GlobalSecondaryIndexes"] = [{
            "IndexName": gsi_name,
            "KeySchema": [{"AttributeName": gsi_attr, "KeyType": "HASH"}],
            "Projection": {"ProjectionType": "ALL"},
        }]

    print(f"  Creando tabla '{nombre}'...")
    client.create_table(**kwargs)
    client.get_waiter("table_exists").wait(TableName=nombre)
    print(f"  ✔ Tabla '{nombre}' lista")


def generar_clientes(tabla):
    ids = []
    for i in range(1, NUM_CLIENTES + 1):
        cliente_id = f"C{i:03d}"
        item = {
            "clienteId": cliente_id,
            "nombre": fake.name(),
            "email": fake.unique.email(),
            "ciudad": fake.city(),
            "fechaRegistro": fake.date_between(start_date="-2y", end_date="today").isoformat(),
            "preferencias": {
                "newsletter": random.choice([True, False]),
                "categoriasFavoritas": random.sample(CATEGORIAS, k=random.randint(1, 3)),
                "direccion": {
                    "calle": fake.street_address(),
                    "ciudad": fake.city(),
                    "cp": fake.postcode(),
                },
            },
        }
        tabla.put_item(Item=item)
        ids.append(cliente_id)
    return ids


def generar_productos(tabla):
    ids = []
    for i in range(1, NUM_PRODUCTOS + 1):
        producto_id = f"P{i:03d}"
        categoria = random.choice(CATEGORIAS)
        item = {
            "productoId": producto_id,
            "nombre": f"{fake.word().capitalize()} {categoria}",
            "categoria": categoria,
            "precio": Decimal(str(round(random.uniform(5, 900), 2))),
            "stock": random.randint(0, 300),
            "atributos": {
                "color": fake.color_name(),
                "garantiaMeses": random.choice([0, 6, 12, 24, 36]),
                "tags": random.sample(["oferta", "nuevo", "popular", "importado"], k=random.randint(0, 2)),
            },
        }
        tabla.put_item(Item=item)
        ids.append(producto_id)
    return ids


def generar_pedidos(tabla, cliente_ids, productos_info):
    for i in range(1, NUM_PEDIDOS + 1):
        pedido_id = f"O{i:03d}"
        cliente_id = random.choice(cliente_ids)

        num_items = random.randint(1, 4)
        productos_pedido = random.sample(list(productos_info.keys()), k=min(num_items, len(productos_info)))

        items = []
        total = Decimal("0")
        for producto_id in productos_pedido:
            precio_unitario = productos_info[producto_id]
            cantidad = random.randint(1, 5)
            total += precio_unitario * cantidad
            items.append({
                "productoId": producto_id,
                "cantidad": cantidad,
                "precioUnitario": precio_unitario,
            })

        item = {
            "pedidoId": pedido_id,
            "clienteId": cliente_id,
            "fecha": fake.date_between(start_date="-1y", end_date="today").isoformat(),
            "estado": random.choice(ESTADOS_PEDIDO),
            "total": round(total, 2),
            "metadata": {
                "metodoPago": random.choice(METODOS_PAGO),
                "cupon": random.choice(CUPONES),
                "canal": random.choice(CANALES),
            },
            "items": items,
        }
        tabla.put_item(Item=item)


def main():
    print("Creando tablas...")
    crear_tabla("Clientes", "clienteId")
    crear_tabla("Productos", "productoId")
    crear_tabla("Pedidos", "pedidoId", gsi=("clienteId", "ClienteIndex"))

    print("\nGenerando y cargando datos...")
    tabla_clientes = dynamodb.Table("Clientes")
    tabla_productos = dynamodb.Table("Productos")
    tabla_pedidos = dynamodb.Table("Pedidos")

    print(f"  {NUM_CLIENTES} clientes...")
    cliente_ids = generar_clientes(tabla_clientes)

    print(f"  {NUM_PRODUCTOS} productos...")
    ids_productos = generar_productos(tabla_productos)

    # Necesitamos el precio de cada producto para calcular totales de pedidos
    productos_info = {}
    for producto_id in ids_productos:
        resp = tabla_productos.get_item(Key={"productoId": producto_id})
        productos_info[producto_id] = resp["Item"]["precio"]

    print(f"  {NUM_PEDIDOS} pedidos...")
    generar_pedidos(tabla_pedidos, cliente_ids, productos_info)

    print("\n¡Listo! Resumen:")
    for nombre in ["Clientes", "Productos", "Pedidos"]:
        resp = client.describe_table(TableName=nombre)
        print(f"  {nombre}: tabla activa (ItemCount puede tardar en actualizarse)")


if __name__ == "__main__":
    main()
```

4. Regresa a la terminal y pega crear el archivo .py a ejecutar

```
nano generar-datos-para-dynamodb.py
```
y pega el contenido (Ctrl + O, Enter, Ctrl + X)

5. En la línea de comando ejecuta el script python

```
python3 generar-datos-para-dynamodb.py
```

6. Verificar en AWS, DynamoDB → Tables → Clientes → Explore table items... y deberías ver 30 registros

+ En comparación con la sesión pasada que hicimos postgreSQL, ¿notan algo diferente?

+ A diferencia del script que usamos para postgreSQL, aquí no hay ningún bloque CONFIG con host/usuario/contraseña que editar

+ Sólo revisa que REGION = "us-east-1" coincida con tu región (ya viene así en el chunk que copiaste)

+ Aquí no necesitas ninguna contraseña de base de datos, porque DynamoDB se autentica con los mismos permisos de tu cuenta de AWS, no con usuario/contraseña propios

+ **No creamos una instancia para la base de datos ni una instancia EC2**.... Esto significa ser serverless

+ Pareciera que estamos creando una tabla así en el "vacío", sin que pertenezca a una "base de datos"

+ En DynamoDB no existe un paso separado de "crear la base de datos"

+ En PostgreSQL y MongoDB/DocumentDB, primero creabamos un servidor/instancia completo (RDS → Create database), y después, dentro de ese servidor, creabamos tablas. Eran dos pasos separados

+ En DynamoDB no hay ese primer paso

+ No existe un "servidor" que aprovisionar... vas directo a crear tablas, y cada tabla ya es, por sí misma, una base de datos independiente y funcional (administrada 100% por AWS, sin servidor visible)

+ En DynamoDB, "crear la tabla" es "crear la base de datos"... es el mismo paso!!!

+ La característica de servicio serverless de DynamoDB significa que no hay un "servidor" que administrar ni configurar

+ Por tanto, tampoco existe el concepto de "una base de datos que agrupa varias tablas"

+ Cada tabla es su propia unidad independiente, completa y autosuficiente

+ La tabla misma es la unidad equivalente a lo que antes llamábamos "una base de datos"

+ La pregunta que quizás te estás haciendo...

+ "¿Y si tengo 2 aplicaciones distintas y no quiero que se mezclen sus tablas?"

+ En DynamoDB, la forma de separarlos no es con una "base de datos" distinta, sino con nombres de tabla distintos o usando cuentas de AWS separadas

+ En DynamoDB, el diseño de tablas empieza preguntando "¿cómo voy a buscar esto?" antes de crear la tabla. A esto se le llama "diseño orientado a patrones de acceso"

# Hagamos algunas consultas

+ Click en el ícono de terminal (`>_`) en la parte superior (a.k.a. abrir el CloudShell de AWS)

## 1. Operaciones sobre tablas (estructura, no datos)

### `create-table`: Crea una tabla nueva
```
aws dynamodb create-table \
  --table-name Prueba \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### `list-tables`: Muestra todas las tablas
```
aws dynamodb list-tables
```

### `describe-table`: Para ver la estructura de una tabla (claves, índices, estado)
```
aws dynamodb describe-table --table-name Clientes
```

### `delete-table`: Borra una tabla completa (OJO: es irreversible)
```
aws dynamodb delete-table --table-name Prueba
```

### `update-table`: Modifica la configuración de una tabla

+ Por ejemplo, para agregar un índice

```
aws dynamodb update-table \
  --table-name Productos \
  --attribute-definitions AttributeName=categoria,AttributeType=S \
  --global-secondary-index-updates \
  '[{"Create":{"IndexName":"CategoriaIndex","KeySchema":[{"AttributeName":"categoria","KeyType":"HASH"}],"Projection":{"ProjectionType":"ALL"}}}]'
```

---

## 2. Operaciones de escritura un item a la vez

### `put-item`: Inserta o reemplaza un item completo
```
aws dynamodb put-item \
  --table-name Clientes \
  --item '{"clienteId": {"S": "C999"}, "nombre": {"S": "Prueba"}}'
```

### `update-item`: Modifica sólo algunos campos de un item existente
```
aws dynamodb update-item \
  --table-name Productos \
  --key '{"productoId": {"S": "P001"}}' \
  --update-expression "SET stock = :s" \
  --expression-attribute-values '{":s": {"N": "10"}}'
```

### `delete-item`: Borra un ítem específico
```
aws dynamodb delete-item \
  --table-name Clientes \
  --key '{"clienteId": {"S": "C999"}}'
```

---

## 3. Operaciones de lectura de uno o varios items

### `get-item`: Trae un item por su clave exacta (la más rápida posible)
```
aws dynamodb get-item \
  --table-name Clientes \
  --key '{"clienteId": {"S": "C001"}}'
```

### `query`: Trae varios items que comparten partition key (eficiente, usa índice)

+ **IMPORTANTE:** El término "query" acá significa algo más específico, no un término genérico que usamos para referirnos a una consulta

```
aws dynamodb query \
  --table-name Pedidos \
  --index-name ClienteIndex \
  --key-condition-expression "clienteId = :c" \
  --expression-attribute-values '{":c": {"S": "C005"}}'
```

### `scan`: Revisa toda la tabla, con o sin filtro (aunque es menos eficiente)
```
aws dynamodb scan --table-name Productos
```

+ Con un filtro:
```
aws dynamodb scan \
  --table-name Productos \
  --filter-expression "categoria = :cat" \
  --expression-attribute-values '{":cat": {"S": "Electrónica"}}'
```

---

## 4. Operaciones en batch a.k.a varios items en una sola llamada

### `batch-get-item`: Traer varios items específicos de una vez (es más eficiente que varios GetItem)
```
aws dynamodb batch-get-item \
  --request-items '{
    "Clientes": {
      "Keys": [
        {"clienteId": {"S": "C001"}},
        {"clienteId": {"S": "C002"}}
      ]
    }
  }'
```

### `batch-write-item`: Inserta o borra varios items de una vez
```
aws dynamodb batch-write-item \
  --request-items '{
    "Clientes": [
      {"PutRequest": {"Item": {"clienteId": {"S": "C900"}, "nombre": {"S": "Lote 1"}}}},
      {"PutRequest": {"Item": {"clienteId": {"S": "C901"}, "nombre": {"S": "Lote 2"}}}}
    ]
  }'
```

+ **Nota**: `batch-write-item` soporta máximo 25 operaciones por llamada

---

## 5. Trasnacciones: varias operaciones que se ejecutan todas o ninguna

### `transact-write-items`: Como una transacción de SQL (BEGIN/COMMIT), pero para varias escrituras
```
aws dynamodb transact-write-items \
  --transact-items '[
    {"Put": {"TableName": "Clientes", "Item": {"clienteId": {"S": "C950"}, "nombre": {"S": "Transacción"}}}},
    {"Update": {"TableName": "Productos", "Key": {"productoId": {"S": "P001"}}, "UpdateExpression": "SET stock = stock - :n", "ExpressionAttributeValues": {":n": {"N": "1"}}}}
  ]'
```
+ Es útil para "crear un pedido y descontar el stock" de forma que, si algo falla, ninguna de las 2 cosas se aplique

### `transact-get-items`: Lee varios items de forma consistente en un solo instante
```
aws dynamodb transact-get-items \
  --transact-items '[
    {"Get": {"TableName": "Clientes", "Key": {"clienteId": {"S": "C001"}}}},
    {"Get": {"TableName": "Productos", "Key": {"productoId": {"S": "P001"}}}}
  ]'
```

## Traer un cliente exacto
```
aws dynamodb get-item \
  --table-name Clientes \
  --key '{"clienteId": {"S": "C001"}}'
```

## Pedidos de un cliente, usando el índice ClienteIndex

```
aws dynamodb query \
  --table-name Pedidos \
  --index-name ClienteIndex \
  --key-condition-expression "clienteId = :c" \
  --expression-attribute-values '{":c": {"S": "C005"}}'
```

## `scan` con filtro para productos de una categoría

```
aws dynamodb scan \
  --table-name Productos \
  --filter-expression "categoria = :cat" \
  --expression-attribute-values '{":cat": {"S": "Electrónica"}}'
```

## `scan` para pedidos por estado

```
aws dynamodb scan \
  --table-name Pedidos \
  --filter-expression "estado = :e" \
  --expression-attribute-values '{":e": {"S": "pendiente"}}'
```

## Filtro sobre un atributo anidado (mapa dentro de mapa)

```
aws dynamodb scan \
  --table-name Clientes \
  --filter-expression "preferencias.direccion.ciudad = :c" \
  --expression-attribute-values '{":c": {"S": "Guadalajara"}}'
```

## `contains()` para encontrar productos que tienen el tag "oferta"

```
aws dynamodb scan \
  --table-name Productos \
  --filter-expression "contains(atributos.tags, :t)" \
  --expression-attribute-values '{":t": {"S": "oferta"}}'
```

## `attribute_exists()` para pedidos que sí tienen cupón con código

```
aws dynamodb scan \
  --table-name Pedidos \
  --filter-expression "attribute_exists(metadata.cupon.codigo)"
```

## Ejemplo con `ExpressionAttributeNames`

+ Cuando el campo coincide con una palabra reservada de DynamoDB (aquí NO nos pasa con nuestros campos, pero es común con nombres como "status" o "size")

+ Si tuvieras un campo llamado, por ejemplo, "status" en vez de "estado"

```
aws dynamodb scan \
#   --table-name Pedidos \
#   --filter-expression "#s = :e" \
#   --expression-attribute-names '{"#s": "status"}' \
#   --expression-attribute-values '{":e": {"S": "pendiente"}}'
```

+ El "#s" es un alias que evita el choque con la palabra reservada

## `PutItem` para insertar un cliente nuevo

```
aws dynamodb put-item \
  --table-name Clientes \
  --item '{
    "clienteId": {"S": "C999"},
    "nombre": {"S": "Cliente de Prueba"},
    "ciudad": {"S": "CDMX"},
    "preferencias": {"M": {
      "newsletter": {"BOOL": true},
      "categoriasFavoritas": {"L": [{"S": "Electrónica"}]}
    }}
  }'
```

## `UpdateItem` para cambiar el stock de un producto

```
aws dynamodb update-item \
  --table-name Productos \
  --key '{"productoId": {"S": "P001"}}' \
  --update-expression "SET stock = :s" \
  --expression-attribute-values '{":s": {"N": "0"}}'
```

## `DeleteItem`para borrar el cliente de prueba

```
aws dynamodb delete-item \
  --table-name Clientes \
  --key '{"clienteId": {"S": "C999"}}'
```

## ¿Cuándo usar cada una?

| Operación | Úsala cuando... |
|---|---|
| `get-item` | Ya sabes exactamente la clave del item que quieres |
| `query` | Quieres varios items que comparten partition key |
| `scan` | Necesitas revisar/filtrar por un campo que no es la clave |
| `put-item` | Quieres crear un item nuevo (o reemplazar uno existente por completo) |
| `update-item` | Quieres cambiar sólo 1-2 campos sin tocar el resto del item |
| `delete-item` | Quieres borrar un item específico |
| `batch-get-item` / `batch-write-item` | Necesitas leer/escribir varios items y quieres ahorrar llamadas |
| `transact-write-items` | Varias escrituras deben tener éxito TODAS o NINGUNA |
| `create-table` / `delete-table` / `describe-table` | Trabajas con la estructura, no con los datos |
