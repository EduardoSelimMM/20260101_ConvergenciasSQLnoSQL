# Sesión 3: 1 de septiembre de 2026 😃

Trabajaremos en https://onecompiler.com/

SQL (relacional, centrado en tablas y esquemas rígidos) y MongoDB (NoSQL, centrado en documentos JSON y esquemas flexibles) partieron de filosofías opuestas

Han ido evolucionando para incorporar características del otro

| Etapa MongoDB | Cláusula SQL Equivalente | Propósito |
|---|---|---|
| `$match` | `WHERE / HAVING` | Filtrar registros según condiciones |
| `$project` | `SELECT` | Seleccionar, renombrar o calcular campos |
| `$group` | `GROUP BY` | Agrupar datos y aplicar funciones (`$sum`, `$avg`) |
| `$sort` | `ORDER BY` | Ordenar resultados |
| `$limit` / `$skip` | `LIMIT / OFFSET` | Paginación de resultados |
| `$lookup` | `JOIN` | Combinar datos entre colecciones/tablas |


Vamos a insertar 4 tablas pequeñas para esta sesión práctica/recordatorio

```
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
 
CREATE TABLE customers (
    id          INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    city        VARCHAR(50),
    signup_date DATE
);
 
CREATE TABLE products (
    id       INTEGER PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    category VARCHAR(50),
    price    DECIMAL(10,2)
);
 
CREATE TABLE orders (
    id           INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers(id),
    order_date   DATE,
    status       VARCHAR(20)
);
 
CREATE TABLE order_items (
    id         INTEGER PRIMARY KEY,
    order_id   INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity   INTEGER NOT NULL
);
```

```
-- ---------------------------------------------------------
-- CUSTOMERS
-- ---------------------------------------------------------
INSERT INTO customers (id, name, city, signup_date) VALUES
(1,  'Ana García',      'CDMX',        '2023-01-15'),
(2,  'Luis Fernández',  'Guadalajara', '2023-02-20'),
(3,  'Marta López',     'Monterrey',   '2023-03-05'),
(4,  'Carlos Ruiz',     'CDMX',        '2023-03-18'),
(5,  'Sofía Torres',    'Puebla',      '2023-04-02'),
(6,  'Jorge Díaz',      'Guadalajara', '2023-05-10'),
(7,  'Elena Morales',   'CDMX',        '2023-06-01'),
(8,  'Pedro Sánchez',   'Querétaro',   '2023-06-15'),
(9,  'Lucía Romero',    'Monterrey',   '2023-07-22'),
(10, 'Diego Castro',    'CDMX',        '2023-08-30');
 
-- ---------------------------------------------------------
-- PRODUCTS
-- ---------------------------------------------------------
INSERT INTO products (id, name, category, price) VALUES
(1,  'Laptop Pro 15',        'Electrónica', 18500.00),
(2,  'Mouse Inalámbrico',    'Electrónica',   350.00),
(3,  'Teclado Mecánico',     'Electrónica',  1200.00),
(4,  'Silla Ergonómica',     'Oficina',      3200.00),
(5,  'Escritorio Ajustable', 'Oficina',      5400.00),
(6,  'Monitor 27"',          'Electrónica',  4800.00),
(7,  'Lámpara LED',          'Oficina',       450.00),
(8,  'Audífonos Bluetooth',  'Electrónica',   900.00),
(9,  'Organizador de Cables','Oficina',       150.00),
(10, 'Webcam HD',            'Electrónica',   700.00);
 
-- ---------------------------------------------------------
-- ORDERS
-- ---------------------------------------------------------
INSERT INTO orders (id, customer_id, order_date, status) VALUES
(1,  1, '2024-01-10', 'shipped'),
(2,  2, '2024-01-12', 'shipped'),
(3,  1, '2024-02-01', 'cancelled'),
(4,  3, '2024-02-14', 'shipped'),
(5,  4, '2024-02-20', 'pending'),
(6,  5, '2024-03-01', 'shipped'),
(7,  2, '2024-03-05', 'cancelled'),
(8,  6, '2024-03-10', 'shipped'),
(9,  7, '2024-03-15', 'pending'),
(10, 8, '2024-04-01', 'shipped'),
(11, 1, '2024-04-10', 'shipped'),
(12, 9, '2024-04-15', 'cancelled');
 
-- ---------------------------------------------------------
-- ORDER_ITEMS
-- ---------------------------------------------------------
INSERT INTO order_items (id, order_id, product_id, quantity) VALUES
(1,  1,  1, 1),
(2,  1,  2, 2),
(3,  2,  4, 1),
(4,  2,  5, 1),
(5,  3,  6, 1),
(6,  4,  3, 2),
(7,  4,  9, 3),
(8,  5,  1, 1),
(9,  6,  7, 4),
(10, 6,  8, 1),
(11, 7,  2, 1),
(12, 8,  6, 2),
(13, 8, 10, 1),
(14, 9,  5, 1),
(15, 10, 1, 1),
(16, 10, 6, 1),
(17, 10, 3, 1),
(18, 11, 8, 2),
(19, 12, 4, 1);
```

+ Recuérdese que en MongoDB no existe el concepto de "esquemas" (`CREATE TABLE`) ni se usan sentencias `INSERT INTO`

+ Acá se usan colecciones y documentos JSON/BSON

+ No necesitas crear la colección manualmente

+ MongoDB la crea automáticamente al insertar el primer documento

+ Acá se utiliza el comando `insertMany` sobre la colección `customers`

```
db.customers.insertMany([
  {
    _id: 1,
    name: "Ana García",
    city: "CDMX",
    signup_date: ISODate("2023-01-15T00:00:00Z")
  },
  {
    _id: 2,
    name: "Luis Fernández",
    city: "Guadalajara",
    signup_date: ISODate("2023-02-20T00:00:00Z")
  },
  {
    _id: 3,
    name: "Marta López",
    city: "Monterrey",
    signup_date: ISODate("2023-03-05T00:00:00Z")
  },
  {
    _id: 4,
    name: "Carlos Ruiz",
    city: "CDMX",
    signup_date: ISODate("2023-03-18T00:00:00Z")
  },
  {
    _id: 5,
    name: "Sofía Torres",
    city: "Puebla",
    signup_date: ISODate("2023-04-02T00:00:00Z")
  },
  {
    _id: 6,
    name: "Jorge Díaz",
    city: "Guadalajara",
    signup_date: ISODate("2023-05-10T00:00:00Z")
  },
  {
    _id: 7,
    name: "Elena Morales",
    city: "CDMX",
    signup_date: ISODate("2023-06-01T00:00:00Z")
  },
  {
    _id: 8,
    name: "Pedro Sánchez",
    city: "Querétaro",
    signup_date: ISODate("2023-06-15T00:00:00Z")
  },
  {
    _id: 9,
    name: "Lucía Romero",
    city: "Monterrey",
    signup_date: ISODate("2023-07-22T00:00:00Z")
  },
  {
    _id: 10,
    name: "Diego Castro",
    city: "CDMX",
    signup_date: ISODate("2023-08-30T00:00:00Z")
  }
]);
```

+ Acá tampoco existe el concepto de "llave primaria" (PRIMARY KEY), acá existen IDs

+ En MongoDB es `_id`. Si no lo defines, MongoDB genera un `ObjectId` automático de 12 bytes

+ ESPECIFICO para los datos de esta tabla: Se utiliza la función `ISODate("YYYY-MM-DD")` para almacenar la fecha como un tipo de dato nativo `BSON Date` en lugar de una simple cadena de texto

```
db.products.insertMany([
  {
    "_id": 1,
    "name": "Laptop Pro 15",
    "category": "Electrónica",
    "price": 18500.0
  },
  {
    "_id": 2,
    "name": "Mouse Inalámbrico",
    "category": "Electrónica",
    "price": 350.0
  },
  {
    "_id": 3,
    "name": "Teclado Mecánico",
    "category": "Electrónica",
    "price": 1200.0
  },
  {
    "_id": 4,
    "name": "Silla Ergonómica",
    "category": "Oficina",
    "price": 3200.0
  },
  {
    "_id": 5,
    "name": "Escritorio Ajustable",
    "category": "Oficina",
    "price": 5400.0
  },
  {
    "_id": 6,
    "name": "Monitor 27\"",
    "category": "Electrónica",
    "price": 4800.0
  },
  {
    "_id": 7,
    "name": "Lámpara LED",
    "category": "Oficina",
    "price": 450.0
  },
  {
    "_id": 8,
    "name": "Audífonos Bluetooth",
    "category": "Electrónica",
    "price": 900.0
  },
  {
    "_id": 9,
    "name": "Organizador de Cables",
    "category": "Oficina",
    "price": 150.0
  },
  {
    "_id": 10,
    "name": "Webcam HD",
    "category": "Electrónica",
    "price": 700.0
  }
])
```

```
db.orders.insertMany([
  {
    "_id": 1,
    "customer_id": 1,
    "order_date": "2024-01-10",
    "status": "shipped"
  },
  {
    "_id": 2,
    "customer_id": 2,
    "order_date": "2024-01-12",
    "status": "shipped"
  },
  {
    "_id": 3,
    "customer_id": 1,
    "order_date": "2024-02-01",
    "status": "cancelled"
  },
  {
    "_id": 4,
    "customer_id": 3,
    "order_date": "2024-02-14",
    "status": "shipped"
  },
  {
    "_id": 5,
    "customer_id": 4,
    "order_date": "2024-02-20",
    "status": "pending"
  },
  {
    "_id": 6,
    "customer_id": 5,
    "order_date": "2024-03-01",
    "status": "shipped"
  },
  {
    "_id": 7,
    "customer_id": 2,
    "order_date": "2024-03-05",
    "status": "cancelled"
  },
  {
    "_id": 8,
    "customer_id": 6,
    "order_date": "2024-03-10",
    "status": "shipped"
  },
  {
    "_id": 9,
    "customer_id": 7,
    "order_date": "2024-03-15",
    "status": "pending"
  },
  {
    "_id": 10,
    "customer_id": 8,
    "order_date": "2024-04-01",
    "status": "shipped"
  },
  {
    "_id": 11,
    "customer_id": 1,
    "order_date": "2024-04-10",
    "status": "shipped"
  },
  {
    "_id": 12,
    "customer_id": 9,
    "order_date": "2024-04-15",
    "status": "cancelled"
  }
])
```

```
db.order_items.insertMany([
  {
    "_id": 1,
    "order_id": 1,
    "product_id": 1,
    "quantity": 1
  },
  {
    "_id": 2,
    "order_id": 1,
    "product_id": 2,
    "quantity": 2
  },
  {
    "_id": 3,
    "order_id": 2,
    "product_id": 4,
    "quantity": 1
  },
  {
    "_id": 4,
    "order_id": 2,
    "product_id": 5,
    "quantity": 1
  },
  {
    "_id": 5,
    "order_id": 3,
    "product_id": 6,
    "quantity": 1
  },
  {
    "_id": 6,
    "order_id": 4,
    "product_id": 3,
    "quantity": 2
  },
  {
    "_id": 7,
    "order_id": 4,
    "product_id": 9,
    "quantity": 3
  },
  {
    "_id": 8,
    "order_id": 5,
    "product_id": 1,
    "quantity": 1
  },
  {
    "_id": 9,
    "order_id": 6,
    "product_id": 7,
    "quantity": 4
  },
  {
    "_id": 10,
    "order_id": 6,
    "product_id": 8,
    "quantity": 1
  },
  {
    "_id": 11,
    "order_id": 7,
    "product_id": 2,
    "quantity": 1
  },
  {
    "_id": 12,
    "order_id": 8,
    "product_id": 6,
    "quantity": 2
  },
  {
    "_id": 13,
    "order_id": 8,
    "product_id": 10,
    "quantity": 1
  },
  {
    "_id": 14,
    "order_id": 9,
    "product_id": 5,
    "quantity": 1
  },
  {
    "_id": 15,
    "order_id": 10,
    "product_id": 1,
    "quantity": 1
  },
  {
    "_id": 16,
    "order_id": 10,
    "product_id": 6,
    "quantity": 1
  },
  {
    "_id": 17,
    "order_id": 10,
    "product_id": 3,
    "quantity": 1
  },
  {
    "_id": 18,
    "order_id": 11,
    "product_id": 8,
    "quantity": 2
  },
  {
    "_id": 19,
    "order_id": 12,
    "product_id": 4,
    "quantity": 1
  }
])
```

## Operaciones básicas

+ En SQL: lectura (`SELECT`), filtrado (`WHERE`) y agrupación (`GROUP BY`)

+ En MongoDB: `.find()` y `.aggregate()`

```
SELECT * FROM customers;
```

```
db.customers.find();
```

Proyección (a.k.a elegir qué columnas devolver)

+ En SQL defines los campos en la lista del SELECT

```
SELECT name, city FROM customers;
```

+ En MongoDB pasas un segundo objeto al `.find()` donde 1 significa incluir el campo y 0 lo excluye

```
db.customers.find({}, { name: 1, city: 1, _id: 0 });
```

OJO: El `_id` se incluye por defecto. En este mini ejemplo lo excluimos explícitamente con 0

Filtros simples

+ En SQL con `WHERE`

```
SELECT * FROM customers WHERE city = 'CDMX';
```

+ En MongoDB las condiciones de comparación usan operadores como $eq (igual), $gt (mayor que), $gte (mayor o igual que), $lt (menor que) y $ne (no igual)

```
db.customers.find({ city: "CDMX" });
```

```
SELECT * FROM customers WHERE signup_date >= '2023-05-01';
```

```
db.customers.find({
  signup_date: { $gte: ISODate("2023-05-01T00:00:00Z") }
});
```

```
SELECT * FROM customers WHERE name LIKE 'A%';
```

```
db.customers.find({ name: { $regex: "^A" } })
```

Filtros con más de una condición

+ En SQL se usa AND y/o OR

```
SELECT * FROM customers WHERE city = 'CDMX' AND signup_date >= '2023-06-01';

```

+ En MongoDB en vez de usar `AND`, se logra incluyen varias condiciones dentro del mismo objeto de filtro

```
db.customers.find({
  city: "CDMX",
  signup_date: { $gte: ISODate("2023-06-01T00:00:00Z") }
});
```

```
SELECT * FROM products WHERE category = 'Electrónica' AND price < 1000;
```

```
db.products.find({ category: "Electrónica", price: { $lt: 1000 } })
```

```
SELECT * FROM customers WHERE city = 'CDMX' OR city = 'Monterrey';
```

+ Para el `OR`, se utiliza el operador $or con un arreglo de condiciones


```
db.customers.find({
  $or: [
    { city: "CDMX" },
    { city: "Monterrey" }
  ]
});
```

+ Nota: También puedes usar el operador $in

```
db.customers.find({ city: { $in: ["CDMX", "Monterrey"] } });
```

```
SELECT * FROM orders WHERE status IN ('shipped', 'pending');
```

```
db.orders.find({ status: { $in: ["shipped", "pending"] } })
```

Ordenamientos y límites

+ En SQL, con `ORDER BY` y `LIMIT`

```
SELECT * FROM customers ORDER BY name ASC LIMIT 3;
```
+ En MongoDB, con .sort() y .limit()

```
db.customers.find().sort({ name: 1 }).limit(3);
```

+ OJO: El .sort() se incluye el nombre del campo y 1 para ascendente o -1 para descendente

```
SELECT * FROM products ORDER BY price DESC LIMIT 3;
```

```
db.products.find().sort({ price: -1 }).limit(3)
```

Agrupación y funciones de Agregación

+ En SQL, con GROUP BY

```
SELECT city, COUNT(*) AS total_customers 
FROM customers 
GROUP BY city;
```

+ En MongoDB, las agrupaciones se procesan con lo que se conoce como Aggregation Framework con la función aggregate(), usando etapas como $match (equivalente a WHERE/HAVING) y $group (equivalente a GROUP BY).

```
db.customers.aggregate([
  {
    $group: {
      _id: "$city",                 // El campo con el que se agrupa
      total_customers: { $sum: 1 }  // Cuenta la cantidad de documentos
    }
  }
]);
```

```
SELECT city, COUNT(*) AS total_customers 
FROM customers 
WHERE signup_date >= '2023-03-01'
GROUP BY city
HAVING total_customers >= 2;
```

```
db.customers.aggregate([
  {
    $match: {
      signup_date: { $gte: ISODate("2023-03-01T00:00:00Z") }
    }
  },
  {
    $group: {
      _id: "$city",
      total_customers: { $sum: 1 }
    }
  },
  {
    $match: {
      total_customers: { $gte: 2 } // Filtra después de agrupar
    }
  }
]);
```

+ Aunque contar es muy importante

```
SELECT COUNT(*) FROM orders WHERE status = 'cancelled';
```

```
db.orders.countDocuments({ status: "cancelled" })
```

```
SELECT DISTINCT category FROM products;

```

```
db.products.distinct("category")
```

## JOINS

+ En SQL, JOIN

+ En Mongo, $lookup

+ En MongoDB, para establecer relaciones se utiliza la etapa $lookup dentro del Aggregation Framework.

+ Por ejemplo, si se quisieran los pedidos con el nombre del cliente

```sql
SELECT o.id, c.name, o.order_date, o.status
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customer_id",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" }, // $unwind convierte el arreglo 'customer' en documentos individuales
  {
    $project: {
      order_date: 1, status: 1,
      customer_name: "$customer.name"
    }
  }
])
```

+ Otro ejemplo: Obtener todos los clientes y sus órdenes si existen

```
SELECT c.name, c.city, o.order_date, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
```

```
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customer_id",
      as: "orders"
    }
  }
]);

```

+ Otro ejemplo: Se quiere sólo clientes que tengan al menos una orden

```
SELECT c.name, o.order_date, o.status
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

```

```
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customer_id",
      as: "order_details"
    }
  },
  {
    $unwind: "$order_details" // $unwind convierte el arreglo 'order_details' en documentos individuales
  },
  // Opcional: Proyección para aplanar y formatear la salida
  {
    $project: {
      _id: 0,
      customer_name: "$name",
      amount: "$order_details.amount",
      status: "$order_details.status"
    }
  }
]);

```

+ Otro ejemplo: Se quiere a los clientes sin pedidos

```
SELECT c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

```
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customer_id",
      as: "orders"
    }
  },
  { $match: { orders: { $size: 0 } } },
  { $project: { name: 1, _id: 0 } }
])
```

+ Otro ejemplo: Detalle completo de la orden con id = 10

```
SELECT o.id AS order_id, p.name, oi.quantity, p.price
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.id = 10;
```

```
db.orders.aggregate([
  { $match: { _id: 10 } },
  {
    $lookup: {
      from: "order_items",
      localField: "_id",
      foreignField: "order_id",
      as: "items"
    }
  },
  { $unwind: "$items" },
  {
    $lookup: {
      from: "products",
      localField: "items.product_id",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$product" },
  {
    $project: {
      order_id: "$_id",
      name: "$product.name",
      quantity: "$items.quantity",
      price: "$product.price"
    }
  }
])
```

## Más sobre GROUP BY y funciones de agregación

```
SELECT category, AVG(price) AS avg_price, COUNT(*) AS n
FROM products
GROUP BY category;
```

```
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      avg_price: { $avg: "$price" },
      n: { $sum: 1 }
    }
  }
])

```

## Más de GROUP BY con HAVING

```
SELECT customer_id, COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

```
db.orders.aggregate([
  { $group: { _id: "$customer_id", total_orders: { $sum: 1 } } },
  { $match: { total_orders: { $gt: 1 } } }
])
```

+ En Mongo HAVING no existe como palabra es simplemente un $match después del $group, porque el pipeline ya filtró/transformó los datos en ese punto

## Subconsultas

+ Ejemplo: Clientes que compraron el producto 1

```
SELECT name FROM customers
WHERE id IN (
  SELECT o.customer_id
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  WHERE oi.product_id = 1
);
```

```
const customerIds = db.orders.aggregate([
  {
    $lookup: {
      from: "order_items",
      localField: "_id",
      foreignField: "order_id",
      as: "items"
    }
  },
  { $unwind: "$items" },
  { $match: { "items.product_id": 1 } },
  { $group: { _id: "$customer_id" } }
]).toArray().map(d => d._id);

db.customers.find({ _id: { $in: customerIds } })
```

+ Mongo también permite subconsultas relacionadas dentro del mismo pipeline con $lookup y `pipeline`

## Unión

Ejemplo: Clientes sin pedidos UNION clientes con algún pedido cancelado

```
SELECT name FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders)
UNION
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'cancelled';
```

```
db.customers.aggregate([
  {
    $lookup: { from: "orders", localField: "_id", foreignField: "customer_id", as: "orders" }
  },
  {
    $match: {
      $or: [
        { orders: { $size: 0 } },
        { "orders.status": "cancelled" }
      ]
    }
  },
  { $project: { name: 1, _id: 0 } }
])
```

+ Nota: Mongo tiene un $unionWith real para combinar dos pipelines distintos, útil cuando las fuentes son colecciones diferentes. Aquí el $or es más directo porque es la misma colección.
