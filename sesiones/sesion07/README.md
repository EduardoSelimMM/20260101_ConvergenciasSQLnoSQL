# Levantar base de datos DynamoDB

## Paso 1: Crear la tabla 'Clientes`

1. Busca DynamoDB en la barra de búsqueda
2. Menú izquierdo → Tables → Create table.

Table name: Clientes
Partition key: clienteId, tipo String
Deja "Sort key" sin marcar, y en Table settings deja "Default settings".
Create table → espera a que diga Active.

## Paso 2: Agregar un par de ítems a mano
Con la tabla Clientes ya creada, haz clic sobre su nombre para entrar a sus detalles.
Pestaña Explore table items.
Botón Create item.
Puedes usar la vista Form (llenar campos con botones "Add new attribute") o cambiar a vista JSON (más rápido). En JSON, pega esto:

```
{
  "clienteId": {"S": "C002"},
  "nombre": {"S": "Carlos Ruiz"},
  "ciudad": {"S": "Monterrey"}
}
```

```
{
  "clienteId": {"S": "C001"},
  "nombre": {"S": "Ana Lopez"},
  "ciudad": {"S": "Puebla"}
}
```

```
pip install faker
```

```
nano generar-datos-para-dynamodb.py
```

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

```
python3 generar-datos-para_dynamodb.py
```
