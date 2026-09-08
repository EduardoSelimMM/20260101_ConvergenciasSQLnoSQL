# Levantar base de datos postgreSQL

Paso 0: Lanzar el Learner Las

Paso 1: Creación de la base de datos

1. En la barra de búsqueda "RDS"
2. En la barra lateral izquierda "Create database"
3. Seleccionar "PostgreSQL"
4. Seleccionar la opción gratuita
5. Ponerle nombre: `mi-postgres`
6. Usuario: `dbadmin`, Contraseña: Poner contraseña (recuérdala/escríbela)
7. Conectividad
VPC: default
Public access: No
VPC security group: deja el que venga preseleccionado (default)
8. Additional configuration → Initial database name: `mi_base_postgres`
9. Create database
10. Una vez que esté lista, ir a ésta y buscar el endpoint, copiarlo y pegarlo para tenerlo a la mano.

Paso 2: Creación de la instancia de EC2

1. En la barra de búsqueda "EC2"
2. Barra lateral: Instancias → Lanzar instancia → `mi_instancia`
3. Seleccionar  `Amazon Linux 2023`, `t3.micro`
4. Key pair → Proceed without a key pair
5. Conectividad
VPC: default, cualquier subred, security group: default
Pestaña Inbound rules , Add rule: Type PostgreSQL (trae el 5432 automático), Source: el default explícitamente

6. Advanced details → IAM instance profile: `LabInstanceProfile`
7. Lanzar instancia

Paso 3: Verificación de la conexión
1. EC2 → selecciona bastion → Connect → Session Manager → Connect.
2. Instala el cliente de PostgreSQL:

```
sudo dnf install -y postgresql15
```

3. Confirma que psql sí se instaló bien

```
psql --version
```

4. Prueba la conexión

psql -h TU_ENDPOINT_AQUI -U dbadmin -d mi_base_postgres


Si te pide la contraseña y te deja entrar (prompt `mi_base_postgres=>`), la conectividad ya funciona

5. Salir del entorno de la base `\q` en la terminal

Paso 4: Poblar la base de datos

1. Instalar pip

```
sudo dnf install -y python3-pip
```

2. Instalar librerías de Python necesarias

```
pip3 install faker psycopg2-binary --user
```

3. Verificar en donde nos localizamos

```
pwd
```

4. Movernos al lugar adecuado

```
cd ~
```

Debería mostrar algo como `/home/ssm-user`

5. Escribir el archivo de Python que poblará la base de datos

Copia el contenido de este chunk

```
import random
import json
import psycopg2
from psycopg2.extras import Json
from faker import Faker

# ============================================================
# CONFIGURACIÓN — edita estos valores con los tuyos
# ============================================================
CONFIG = {
    "host": "mi-postgres.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com",  # tu endpoint de RDS
    "port": 5432,
    "dbname": "mi_base_postgres",
    "user": "dbadmin",
    "password": "TU_CONTRASEÑA_AQUI",
    "sslmode": "require",  # RDS PostgreSQL exige SSL por defecto
}

NUM_CLIENTES = 30
NUM_PRODUCTOS = 25
NUM_PEDIDOS = 80

SEMILLA = 42  # cualquier número entero fijo; cámbialo si quieres OTRO dataset reproducible distinto

random.seed(SEMILLA)   # controla random.choice, random.sample, random.randint, random.uniform
Faker.seed(SEMILLA)    # controla todo lo que genera Faker (nombres, emails, direcciones, etc.)

fake = Faker("es_MX")  # nombres, direcciones y ciudades en español latinoamericano

CATEGORIAS = ["Electrónica", "Muebles", "Papelería", "Accesorios", "Hogar"]
METODOS_PAGO = ["tarjeta", "paypal", "transferencia"]
ESTADOS_PEDIDO = ["entregado", "pendiente", "cancelado"]
CANALES = ["web", "app"]
CUPONES = [None, {"codigo": "DESC10", "descuento_pct": 10}, {"codigo": "BIENVENIDA", "descuento_pct": 15}]

def crear_tablas(cur):
    """Crea las tablas si no existen (mismo esquema que poblar-datos-ejemplo.sql)."""
    cur.execute("""
        DROP TABLE IF EXISTS detalle_pedidos;
        DROP TABLE IF EXISTS pedidos;
        DROP TABLE IF EXISTS productos;
        DROP TABLE IF EXISTS clientes;

        CREATE TABLE clientes (
            id SERIAL PRIMARY KEY,
            nombre VARCHAR(150) NOT NULL,
            email VARCHAR(150),
            ciudad VARCHAR(100),
            fecha_registro DATE,
            preferencias JSONB
        );

        CREATE TABLE productos (
            id SERIAL PRIMARY KEY,
            nombre VARCHAR(150) NOT NULL,
            categoria VARCHAR(50),
            precio NUMERIC(10,2),
            stock INT,
            atributos JSONB
        );

        CREATE TABLE pedidos (
            id SERIAL PRIMARY KEY,
            cliente_id INT REFERENCES clientes(id),
            fecha DATE,
            estado VARCHAR(20),
            total NUMERIC(10,2),
            metadata JSONB
        );

        CREATE TABLE detalle_pedidos (
            id SERIAL PRIMARY KEY,
            pedido_id INT REFERENCES pedidos(id),
            producto_id INT REFERENCES productos(id),
            cantidad INT,
            precio_unitario NUMERIC(10,2)
        );
    """)

def generar_clientes(cur, n):
    ids = []
    for _ in range(n):
        nombre = fake.name()
        email = fake.unique.email()
        ciudad = fake.city()
        fecha_registro = fake.date_between(start_date="-2y", end_date="today")
        preferencias = {
            "newsletter": random.choice([True, False]),
            "categorias_favoritas": random.sample(CATEGORIAS, k=random.randint(1, 3)),
            "direccion": {
                "calle": fake.street_address(),
                "ciudad": ciudad,
                "cp": fake.postcode(),
            },
        }
        cur.execute(
            """INSERT INTO clientes (nombre, email, ciudad, fecha_registro, preferencias)
               VALUES (%s, %s, %s, %s, %s) RETURNING id""",
            (nombre, email, ciudad, fecha_registro, Json(preferencias)),
        )
        ids.append(cur.fetchone()[0])
    return ids

def generar_productos(cur, n):
    ids = []
    for _ in range(n):
        categoria = random.choice(CATEGORIAS)
        nombre = f"{fake.word().capitalize()} {categoria}"
        precio = round(random.uniform(5, 900), 2)
        stock = random.randint(0, 300)
        atributos = {
            "color": fake.color_name(),
            "garantia_meses": random.choice([0, 6, 12, 24, 36]),
            "tags": random.sample(["oferta", "nuevo", "popular", "importado"], k=random.randint(0, 2)),
        }
        cur.execute(
            """INSERT INTO productos (nombre, categoria, precio, stock, atributos)
               VALUES (%s, %s, %s, %s, %s) RETURNING id""",
            (nombre, categoria, precio, stock, Json(atributos)),
        )
        ids.append(cur.fetchone()[0])
    return ids

def generar_pedidos_y_detalles(cur, n, cliente_ids, producto_ids):
    for _ in range(n):
        cliente_id = random.choice(cliente_ids)
        fecha = fake.date_between(start_date="-1y", end_date="today")
        estado = random.choice(ESTADOS_PEDIDO)
        metadata = {
            "metodo_pago": random.choice(METODOS_PAGO),
            "cupon": random.choice(CUPONES),
            "canal": random.choice(CANALES),
        }

        # Insertamos el pedido primero con total en 0, lo actualizamos después
        cur.execute(
            """INSERT INTO pedidos (cliente_id, fecha, estado, total, metadata)
               VALUES (%s, %s, %s, 0, %s) RETURNING id""",
            (cliente_id, fecha, estado, Json(metadata)),
        )
        pedido_id = cur.fetchone()[0]

        # Entre 1 y 4 productos distintos por pedido
        num_items = random.randint(1, 4)
        productos_pedido = random.sample(producto_ids, k=min(num_items, len(producto_ids)))
        total = 0
        for producto_id in productos_pedido:
            cur.execute("SELECT precio FROM productos WHERE id = %s", (producto_id,))
            precio_unitario = cur.fetchone()[0]
            cantidad = random.randint(1, 5)
            total += float(precio_unitario) * cantidad
            cur.execute(
                """INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
                   VALUES (%s, %s, %s, %s)""",
                (pedido_id, producto_id, cantidad, precio_unitario),
            )

        cur.execute("UPDATE pedidos SET total = %s WHERE id = %s", (round(total, 2), pedido_id))

def main():
    print("Conectando a PostgreSQL...")
    conn = psycopg2.connect(**CONFIG)
    conn.autocommit = False
    cur = conn.cursor()

    try:
        print("Creando tablas...")
        crear_tablas(cur)

        print(f"Generando {NUM_CLIENTES} clientes...")
        cliente_ids = generar_clientes(cur, NUM_CLIENTES)

        print(f"Generando {NUM_PRODUCTOS} productos...")
        producto_ids = generar_productos(cur, NUM_PRODUCTOS)

        print(f"Generando {NUM_PEDIDOS} pedidos con sus detalles...")
        generar_pedidos_y_detalles(cur, NUM_PEDIDOS, cliente_ids, producto_ids)

        conn.commit()
        print("¡Listo! Datos insertados correctamente.")

        cur.execute("SELECT COUNT(*) FROM clientes")
        print(f"  clientes: {cur.fetchone()[0]}")
        cur.execute("SELECT COUNT(*) FROM productos")
        print(f"  productos: {cur.fetchone()[0]}")
        cur.execute("SELECT COUNT(*) FROM pedidos")
        print(f"  pedidos: {cur.fetchone()[0]}")
        cur.execute("SELECT COUNT(*) FROM detalle_pedidos")
        print(f"  detalle_pedidos: {cur.fetchone()[0]}")

    except Exception as e:
        conn.rollback()
        print(f"Error, se revirtieron los cambios: {e}")
        raise
    finally:
        cur.close()
        conn.close()

if __name__ == "__main__":
    main()
```

De nuevo en el bash,

```
nano generar-datos.py
```

Te abrirá un editor de texto. Pega lo que copiaste antes

Ubica el endpoint que guardaste antes
Recuerda la contraseña que antes le pusiste al dbadmin

Modifica el diccionario de configuración, en "host" debe ir el endpoint que guardaste antes

```
CONFIG = {
    "host": "mi-postgres.XXXX.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "mi_base_postgres",
    "user": "dbadmin",
    "password": "TU_CONTRASEÑA_REAL_AQUI",
    "sslmode": "require",
}
```

Ctrl+O (guardar), Enter (confirmar) y Ctrl+X (salir)


Ejecutar

```
python3 generar-datos.py
```

Verifica que funcionó

```
psql -h mi-postgres.XXXX.us-east-1.rds.amazonaws.com -U dbadmin -d mi_base_postgres
```

Pon tu contraseña

Ejecuta

```
SELECT COUNT(*) FROM clientes;
```

Paso 5: Practicar nuestras queries

```
\dt
```
```
\d clientes
```

```
\d productos
```

```
\d pedidos
```

```
\d detalle_pedidos
```

```
\! nano mi_query.sql
```

```
\i mi_query.sql
```
### Ejemplo 0:

```
SELECT id, nombre, preferencias FROM clientes LIMIT 5;
```

```
SELECT id, nombre, atributos FROM productos LIMIT 5;
```

```
SELECT id, estado, metadata FROM pedidos LIMIT 5;
```

### Ejemplo 1:

Ver qué colores existen realmente en tus datos

```
SELECT DISTINCT atributos->>'color' AS color
FROM productos
ORDER BY color;
```

### Ejemplo 2:

Extraer y castear un número que vive dentro del JSON como texto

```
SELECT nombre, (atributos->>'garantia_meses')::int AS garantia
FROM productos
ORDER BY garantia DESC;
```

OJO: `->>'garantia_meses'` devuelve texto, hay que convertirlo para comparar/ordenar numéricamente

### Ejemplo 3:

Filtrar por contención exacta de un campo con `@>`

```
SELECT nombre, atributos
FROM productos
WHERE atributos @> '{"garantia_meses": 24}';
```

### Ejemplo 4:

Productos con más de un tag usando `jsonb_array_length`

```
SELECT nombre, atributos->'tags' AS tags
FROM productos
WHERE jsonb_array_length(atributos->'tags') > 1;
```

### Ejemplo 5:

Clientes que sí quieren newsletter

```
SELECT nombre, preferencias->>'newsletter' AS newsletter
FROM clientes
WHERE preferencias @> '{"newsletter": true}';
```

### Ejemplo 6:

Clientes cuya categoría favorita incluye "Electrónica" (operador ? sobre un arreglo)

```
SELECT nombre, preferencias->'categorias_favoritas' AS favoritas
FROM clientes
WHERE preferencias->'categorias_favoritas' ? 'Electrónica';
```

### Ejemplo 7:

Extraer un campo de 2 niveles de profundidad con `#>>`


```
SELECT nombre, preferencias #>> '{direccion,ciudad}' AS ciudad_envio
FROM clientes;
```

### Ejemplo 8:

Este es un error común: null de JSON vs NULL de SQL

Si quisiéramos contar TODOS los pedidos

```
SELECT COUNT(*) FROM pedidos WHERE metadata->'cupon' IS NOT NULL;
```

Esto NO filtra los pedidos sin cupón, porque el "null" quedó guardado como valor JSON (jsonb 'null'), no como ausencia de valor SQL


La forma correcta es navegar un nivel más adentro. Si el padre es JSON null, el siguiente `->>` sí devuelve NULL de SQL de verdad


```
SELECT COUNT(*) FROM pedidos WHERE metadata->'cupon'->>'codigo' IS NOT NULL;  -- cuenta solo los que sí tienen cupón
```

Otra forma usando jsonb_typeof

```
SELECT COUNT(*) FROM pedidos WHERE jsonb_typeof(metadata->'cupon') <> 'null';
```

### Ejemplo 9:

Pedidos con cupón real

```
SELECT p.id, c.nombre, p.total,
       p.metadata->'cupon'->>'codigo' AS codigo_cupon,
       (p.metadata->'cupon'->>'descuento_pct')::int AS descuento_pct
FROM pedidos p
JOIN clientes c ON p.cliente_id = c.id
WHERE p.metadata->'cupon'->>'codigo' IS NOT NULL;
```

### Ejemplo 10:

Descuento promedio otorgado (sólo entre pedidos que sí tuvieron cupón)

```
SELECT ROUND(AVG((metadata->'cupon'->>'descuento_pct')::numeric), 2) AS descuento_promedio
FROM pedidos
WHERE metadata->'cupon'->>'codigo' IS NOT NULL;
```

### Ejemplo 11:

Número de pedidos que hay por canal (web/app), extrayendo directo del JSONB

```
SELECT metadata->>'canal' AS canal, COUNT(*) AS total_pedidos
FROM pedidos
GROUP BY metadata->>'canal'
ORDER BY total_pedidos DESC;
```

### Ejemplo 12:

Frecuencia de tags entre todos los productos

```
SELECT tag, COUNT(*) AS cantidad
FROM productos, jsonb_array_elements_text(atributos->'tags') AS tag
GROUP BY tag
ORDER BY cantidad DESC;
```

OJO: Vean que acá expandimos arreglo y agrupamos

### Ejemplo 13:

Ranking de ciudades de envío con más clientes, extraído del JSONB anidado

```
SELECT preferencias #>> '{direccion,ciudad}' AS ciudad, COUNT(*) AS num_clientes
FROM clientes
GROUP BY ciudad
ORDER BY num_clientes DESC;
```

### Ejemplo 14:

Gasto total de clientes que aceptan newsletter

```
SELECT c.nombre, SUM(p.total) AS gasto_total
FROM clientes c
JOIN pedidos p ON p.cliente_id = c.id
WHERE c.preferencias @> '{"newsletter": true}'
GROUP BY c.nombre
ORDER BY gasto_total DESC;
```

OJO: Acá combinamos JSONB, JOIN y GROUP BY

### Ejemplo 15:

Modificar un valor dentro del JSON sin tocar el resto de la estructura

```
UPDATE productos
SET atributos = jsonb_set(atributos, '{garantia_meses}', '36')
WHERE (atributos->>'garantia_meses')::int < 12;
```

Verificamos cuántos se actualizaron:

```
SELECT COUNT(*) FROM productos WHERE atributos @> '{"garantia_meses": 36}';
```

### Ejemplo 16:

En postgreSQL se puede crear un índice para acelerar filtros JSONB en tablas grandes

``` 
CREATE INDEX IF NOT EXISTS idx_pedidos_metadata ON pedidos USING GIN (metadata);
```

### Ejemplo 17:

```
SELECT
    tc.table_name AS tabla_origen,
    kcu.column_name AS columna_fk,
    ccu.table_name AS tabla_referenciada,
    ccu.column_name AS columna_referenciada
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';
```

OJO: WHERE tc.constraint_type = 'FOREIGN KEY' dice "tráeme todas las llaves foráneas que existan en la base, sean cuales sean sus tablas"


# Levantar base de datos DynamoDB

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
