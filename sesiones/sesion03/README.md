# Sesión 3: 1 de septiembre de 2026

Trabajaremos en https://onecompiler.com/

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
-- CUSTOMERS (10)
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
-- PRODUCTS (10)
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
-- ORDERS (12)
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
-- ORDER_ITEMS (19)
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

