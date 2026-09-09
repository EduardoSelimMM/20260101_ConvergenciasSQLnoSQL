# Levantar base de datos DynamoDB

+ Antes de levantar la base, demos algunos detalles generales sobre DynamoDB

+ Entre las instrucciones se irán dando algunas otras características

+ **Amazon DynamoDB** es un servicio de base de datos **NoSQL** con modelo **clave-valor** totalmente administrado por AWS

+ Promete tener latencias de un solo dígito de milisegundo a cualquier escala

+ Escala horizontalmente de forma automática para manejar muchísimas peticiones por segundo sin degradación en el tiempo de respuesta

+ Es lo que se conoce como **serverless**... ¿a qué les suena?

+ Me gusta más decir que es **totalmente administrada**

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
  "Roles": { "L": [ { "S": "admin" }, { "S": "developer" } ] },
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

1. Click en el ícono de terminal (`>_`) en la parte superior
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

+ No creamos una instancia para la base de datos ni una instancia EC2

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

# "Convergencias" entre DynamoDB y SQL

+ PartiQL es un lenguaje de consulta compatible con SQL que facilita la consulta eficiente de datos en DynamoDB mediante las sentencias DML (Data Manipulation Language) SELECT, INSERT, UPDATE y DELETE, i.e. manipular datos dentro de una estructura ya existente

+ ¿Por qué? ¿Para qué?

+ Para poder utilizar un lenguaje conocido para comenzar a trabajar con DynamoDB. No necesitas conocer **completamente** el lenguaje de consultas propio de DynamoDB... Por supuesto, esto tiene sus "asegunes"

+ Puedes utilizar PartiQL si no quieres trabajar directamente con FilterExpressions

+ PartiQL fue agregado en 2020, como una "capa de conveniencia" dado el capitalismo voraz... intentar darle poderes SQL a una base NoSQL

+ La forma "nativa" y original de DynamoDB desde el principio nunca fue un lenguaje de consultas tipo SQL, sino un conjunto de operaciones de API, cada una con su propio formato JSON

+ PartiQL no le agrega ninguna capacidad nueva a DynamoDB

+ Cuando Postgres agregó JSONB, sí ganó una capacidad real que no tenía antes: guardar y consultar estructuras flexibles con índices propios, funciones de extracción, y sigue pudiendo usar JOIN, GROUP BY, agregaciones SQL completas incluso sobre esos datos JSONB

+ PartiQL, en cambio, no le da a DynamoDB ninguna capacidad que no tuviera ya

+ Por debajo, cada SELECT de PartiQL se traduce exactamente a uno de los mismos 3 movimientos que ya se tenían desde antes de que existiera PartiQL: GetItem, Query, o Scan

+ PartiQL es puro maquillaje de sintaxis, cambia cómo se ve lo que escribes, no lo que la base de datos puede hacer

+ Entonces "forzar a DynamoDB a parecer SQL" es mucho más superficial que "forzar a postgreSQL a parecer MongoDB"

+ No estamos en arenas movedizas acá, pues no hay una tensión filosófica profunda

+ PartiQL no está intentando que DynamoDB haga cosas para las que no fue diseñado. Es sólo vocabulario distinto para las mismas operaciones limitadas de siempre

+ No hay JOIN... cada consulta de PartiQL trabaja sobre una sola tabla, nunca combina varias.

+ No hay agregaciones SUM, AVG, COUNT de grupo, etc

+ No hay CREATE TABLE, ALTER TABLE

+ No hay GROUP BY ni HAVING

+ Las cláusulas LIMIT, GROUP BY y HAVING no son compatibles con las sentencias SELECT de PartiQL

+ No hay `subqueries` i.e. no puedes anidar un SELECT dentro de otro... algo súper súper súper común en SQL relacional

+ ORDER BY sólo funciona en casos muy específicos. Generalmente necesitas tener ya un WHERE sobre la llave de partición

	+ Si intentas ordenar sin eso, se obtiene un error de validación que dice que debe existir una cláusula WHERE en la sentencia cuando se usa ORDER BY

	+ Básicamente porque al ser un Scan, no hay forma posible de tener ORDER BY sin ese WHERE, ya que DynamoDB no ordena las filas al leerlas de varias particiones y PartiQL no hace ningún procesamiento adicional sobre el resultado

+ PartiQL no es lo suficientemente "inteligente" como para elegir un índice secundario cuando el WHERE filtra sobre la llave de ese índice

	+ Se tiene que especificar directamente el índice secundario en la cláusula FROM para poder lograrlo

+ Hay PartiQL para Amazon Redshift, AWS TwinMaker, AWS Quantum Ledger DB... pero me vuelve a meter en arenas movedizas y se sale de mi área de experiencia

+ Cada implementación/motor/producto decide qué subconjunto del lenguaje completo va a soportar, según lo que su implementación/motor/producto puede ejecutar eficientemente
