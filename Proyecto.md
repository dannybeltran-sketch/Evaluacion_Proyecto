 
<div align="center">

# SISTEMA DE GESTIÓN PARA UNA TIENDA DE ABARROTES

**Proyecto de Base de Datos**

---

</div>

## 1. TÍTULO

*Sistema de gestión para una tienda de abarrotes.*

---

## 2. CONTEXTO DEL PROBLEMA

En la ciudad de Loja, Ecuador, las tiendas de abarrotes se dedican a la venta de productos de consumo diario, como granos, aceites, lácteos, bebidas, artículos de aseo y productos andinos. Para realizar sus actividades correctamente, necesitan llevar un control de los productos que ofrecen, las ventas que realizan, sus clientes y los empleados que trabajan en el establecimiento.

Sin embargo, cuando la información no se encuentra organizada en una base de datos, pueden presentarse dificultades al momento de registrar las ventas, consultar las facturas, verificar las cantidades disponibles de los productos o conocer cuánto dinero se ha vendido en una fecha determinada. Esto puede ocasionar errores en los registros, pérdida de información y problemas para mantener actualizado el inventario.

Esta situación afecta principalmente al propietario de la tienda y a los empleados, ya que necesitan consultar y registrar información constantemente para realizar sus actividades. También puede afectar a los clientes cuando existen inconvenientes con los datos de sus facturas o con la disponibilidad de los productos.

Por esta razón, se considera necesario diseñar una base de datos que permita almacenar y organizar la información de los empleados, clientes, productos, categorías, facturas y detalles de cada venta. De esta manera, se podrá llevar un mejor control de las operaciones del establecimiento, consultar las ventas realizadas y mantener actualizado el stock de los productos cada vez que se registre una venta [1].

---

## 3. OBJETIVOS

### 3.1. Objetivo General

**Tienda de Abarrotes**

Diseñar e implementar una base de datos para la gestión de una tienda de abarrotes en la ciudad de Loja, mediante el modelado de entidades, relaciones y restricciones, así como la utilización de sentencias SQL, con la finalidad de organizar la información de los clientes, empleados, productos, inventario y ventas del establecimiento.

### 3.2. Objetivos Específicos

1. Elaborar el modelo conceptual y lógico de la base de datos mediante un diagrama Entidad-Relación y su correspondiente modelo relacional, identificando las entidades, atributos, relaciones y restricciones necesarias para representar el funcionamiento de la tienda de abarrotes.

2. Implementar la estructura de la base de datos mediante sentencias SQL, creando las tablas, claves primarias, claves foráneas y restricciones correspondientes, además de realizar consultas que permitan registrar y obtener información sobre los productos, clientes, empleados y ventas.

---

## 4. MODELOS CONCEPTUALES DE BASES DE DATOS
### 4.1. Modelo Entidad-Relación (E-R)

[**Enlace al Modelo Entidad-Relación (E-R)**](https://drive.google.com/file/d/1j272YgXVbnev2120fo7G8wrQDtMmAu6F/view?usp=sharing)

### 4.2. Modelo Relacional

[**Enlace al Modelo Relacional**](https://drive.google.com/file/d/1mvquf_8ErXrfep5RsL8J2iRFjaYvM3KZ/view?usp=sharing)


# 5. CÓDIGO SQL

## 5.1. Creación de Tablas (DDL)

```sql
CREATE TABLE Empleado (
    id_empleado INT AUTO_INCREMENT PRIMARY KEY,
    cedula VARCHAR(10) NOT NULL UNIQUE,
    nombres VARCHAR(60) NOT NULL,
    apellidos VARCHAR(60) NOT NULL,
    cargo VARCHAR(30) NOT NULL,
    turno VARCHAR(12) NOT NULL,
    telefono VARCHAR(15),
    CONSTRAINT ck_empleado_turno CHECK (turno IN ('MATUTINO','VESPERTINO','COMPLETO'))
);

CREATE TABLE Cliente (
    id_cliente INT AUTO_INCREMENT PRIMARY KEY,
    tipo_identificacion VARCHAR(20) NOT NULL,
    identificacion VARCHAR(13) NOT NULL UNIQUE,
    nombres VARCHAR(100) NOT NULL,
    apellidos VARCHAR(60),
    celular VARCHAR(15),
    ciudad VARCHAR(60) NOT NULL DEFAULT 'Loja',
    calle VARCHAR(120),
    CONSTRAINT ck_cliente_tipo CHECK (tipo_identificacion IN
    ('CEDULA','RUC','PASAPORTE','CONSUMIDOR_FINAL'))
);

CREATE TABLE Categoria (
    id_categoria INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(60) NOT NULL UNIQUE
);

CREATE TABLE Producto (
    id_producto INT AUTO_INCREMENT PRIMARY KEY,
    codigo_barras VARCHAR(20) UNIQUE,
    nombre VARCHAR(100) NOT NULL,
    id_categoria INT NOT NULL,
    precio_venta DECIMAL(8,2) NOT NULL,
    tarifa_iva DECIMAL(4,2) NOT NULL DEFAULT 15.00,
    stock_actual DECIMAL(10,3) NOT NULL DEFAULT 0,
    stock_minimo DECIMAL(10,3) NOT NULL DEFAULT 0,
    unidad_medida VARCHAR(10) NOT NULL DEFAULT 'UNIDAD',
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    CONSTRAINT fk_producto_categoria FOREIGN KEY (id_categoria)
        REFERENCES Categoria(id_categoria),
    CONSTRAINT ck_producto_precio CHECK (precio_venta >= 0),
    CONSTRAINT ck_producto_iva CHECK (tarifa_iva BETWEEN 0 AND 100),
    CONSTRAINT ck_producto_stock CHECK (stock_actual >= 0 AND stock_minimo >= 0)
);

CREATE TABLE Factura (
    id_factura INT AUTO_INCREMENT PRIMARY KEY,
    numero_factura VARCHAR(17) NOT NULL UNIQUE,
    fecha_emision DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    id_cliente INT NOT NULL,
    id_empleado INT NOT NULL,
    subtotal_iva0 DECIMAL(12,2) NOT NULL DEFAULT 0,
    subtotal_gravado DECIMAL(12,2) NOT NULL DEFAULT 0,
    iva DECIMAL(12,2) NOT NULL DEFAULT 0,
    total DECIMAL(12,2) NOT NULL DEFAULT 0,
    estado VARCHAR(10) NOT NULL DEFAULT 'EMITIDA',
    CONSTRAINT fk_factura_cliente FOREIGN KEY (id_cliente) REFERENCES Cliente(id_cliente),
    CONSTRAINT fk_factura_empleado FOREIGN KEY (id_empleado) REFERENCES Empleado(id_empleado),
    CONSTRAINT ck_factura_estado CHECK (estado IN ('EMITIDA','ANULADA'))
);

CREATE TABLE Detalle_Factura (
    id_factura INT NOT NULL,
    num_linea SMALLINT NOT NULL,
    id_producto INT NOT NULL,
    cantidad DECIMAL(10,3) NOT NULL,
    precio_unitario DECIMAL(8,2) NOT NULL,
    tarifa_iva DECIMAL(4,2) NOT NULL,
    subtotal DECIMAL(12,2) AS (ROUND(cantidad * precio_unitario, 2)) STORED,
    PRIMARY KEY (id_factura, num_linea),
    CONSTRAINT fk_detalle_factura FOREIGN KEY (id_factura)
        REFERENCES Factura(id_factura) ON DELETE CASCADE,
    CONSTRAINT fk_detalle_producto FOREIGN KEY (id_producto)
        REFERENCES Producto(id_producto),
    CONSTRAINT ck_detalle_cantidad CHECK (cantidad > 0),
    CONSTRAINT ck_detalle_precio CHECK (precio_unitario >= 0)
);
```

### Trigger: descuento automático del stock

```sql
DELIMITER $$

CREATE TRIGGER trg_detalle_descuenta_stock
AFTER INSERT ON Detalle_Factura
FOR EACH ROW
BEGIN
    UPDATE Producto
    SET stock_actual = stock_actual - NEW.cantidad
    WHERE id_producto = NEW.id_producto;
END$$

DELIMITER ;
```

---

## 5.2. Inserción de Datos (DML)

### Categorías de productos

```sql
INSERT INTO Categoria (nombre)
VALUES
    ('Lácteos'),
    ('Bebidas'),
    ('Abarrotes'),
    ('Limpieza'),
    ('Frutas y Verduras');
```

### Empleados

```sql
INSERT INTO Empleado (cedula, nombres, apellidos, cargo, turno, telefono)
VALUES
    ('1104567891', 'María Fernanda', 'Paredes Castillo', 'Cajera', 'MATUTINO', '0987654321'),
    ('1103456782', 'Luis Alberto', 'Ordóñez Vega', 'Cajero', 'VESPERTINO', '0991234567'),
    ('1105678903', 'Carmen Lucía', 'Jaramillo Torres', 'Supervisora', 'COMPLETO', '0984561237'),
    ('1102345674', 'Jorge Andrés', 'Salinas Ruiz', 'Bodeguero', 'MATUTINO', NULL),
    ('1106789015', 'Paola Estefanía', 'Cueva Ríos', 'Cajera', 'VESPERTINO', '0978123456');
```

### Clientes

La ciudad toma el valor por defecto 'Loja' cuando no se indica.

```sql
INSERT INTO Cliente (tipo_identificacion, identificacion, nombres, apellidos, celular, ciudad, calle)
VALUES
    ('CEDULA', '1100123456', 'Ana Gabriela', 'Armijos León', '0981112233', 'Loja', 'Av. Universitaria y Lourdes'),
    ('CEDULA', '1101987654', 'Pedro Xavier', 'Samaniego Ochoa', '0992223344', 'Loja', 'Calle Bolívar 10-25'),
    ('RUC', '1190012345001', 'Distribuidora El Sol S.A.', NULL, '0993334455', 'Loja', 'Av. Orillas del Zamora'),
    ('PASAPORTE', 'X1234567', 'John Michael', 'Smith', '0974445566', 'Cuenca', 'Calle Larga 5-12'),
    ('CONSUMIDOR_FINAL', '9999999999999', 'CONSUMIDOR FINAL', NULL, NULL, DEFAULT, NULL),
    ('CEDULA', '1103345678', 'Sofía Valentina', 'Castillo Mora', '0965556677', 'Catamayo', 'Calle Eloy Alfaro 4-30');
```

### Productos

```sql
INSERT INTO Producto (codigo_barras, nombre, id_categoria, precio_venta, tarifa_iva, stock_actual, stock_minimo, unidad_medida, activo)
VALUES
    ('7861001000011', 'Leche entera 1L', 1, 1.10, 0.00, 120, 20, 'UNIDAD', TRUE),
    ('7861001000028', 'Queso fresco 500g', 1, 3.50, 0.00, 40, 10, 'UNIDAD', TRUE),
    ('7861002000013', 'Gaseosa cola 2L', 2, 2.25, 15.00, 80, 24, 'UNIDAD', TRUE),
    ('7861002000020', 'Agua sin gas 600ml', 2, 0.60, 0.00, 200, 48, 'UNIDAD', TRUE),
    ('7861003000014', 'Arroz grano largo', 3, 1.20, 0.00, 150, 30, 'KG', TRUE),
    ('7861003000021', 'Aceite vegetal 1L', 3, 3.40, 15.00, 60, 15, 'UNIDAD', TRUE),
    ('7861003000038', 'Atún en lata 170g', 3, 1.85, 15.00, 8, 12, 'UNIDAD', TRUE),
    ('7861004000015', 'Detergente en polvo 1kg', 4, 4.20, 15.00, 45, 10, 'UNIDAD', TRUE),
    ('7861004000022', 'Cloro 1L', 4, 1.50, 15.00, 30, 15, 'UNIDAD', FALSE),
    (NULL, 'Tomate riñón', 5, 1.00, 0.00, 35.50, 10, 'KG', TRUE),
    ('7861005000019', 'Banano', 5, 0.80, 0.00, 6, 10, 'KG', TRUE);
```

### Facturas

Los totales corresponden a la suma de sus detalles; IVA 15 %.

```sql
INSERT INTO Factura (numero_factura, fecha_emision, id_cliente, id_empleado, subtotal_iva0, subtotal_gravado, iva, total, estado)
VALUES
    ('001-001-000000001', '2026-10-05 08:15:00', 1, 1, 7.40, 4.50, 0.67, 12.57, 'EMITIDA'),
    ('001-001-000000002', '2026-10-05 10:40:00', 2, 1, 7.10, 3.40, 0.51, 11.01, 'EMITIDA'),
    ('001-001-000000003', '2026-10-06 17:05:00', 3, 2, 24.00, 68.40, 10.26, 102.66, 'EMITIDA'),
    ('001-001-000000004', '2026-10-07 09:30:00', 5, 1, 2.70, 0.00, 0.00, 2.70, 'EMITIDA'),
    ('001-001-000000005', '2026-10-08 18:20:00', 4, 5, 7.00, 10.05, 1.51, 18.56, 'EMITIDA'),
    ('001-001-000000006', '2026-10-09 11:10:00', 1, 3, 6.60, 12.10, 1.81, 20.51, 'EMITIDA'),
    ('001-001-000000007', '2026-10-10 16:45:00', 2, 2, 0.00, 9.05, 1.36, 10.41, 'ANULADA'),
    ('001-001-000000008', '2026-10-11 12:00:00', 3, 5, 52.80, 10.20, 1.53, 64.53, 'EMITIDA');
```

### Detalle de facturas

El subtotal es una columna generada; el trigger descuenta el stock.

```sql
INSERT INTO Detalle_Factura (id_factura, num_linea, id_producto, cantidad, precio_unitario, tarifa_iva)
VALUES
    (1, 1, 1, 4, 1.10, 0.00),
    (1, 2, 5, 2.5, 1.20, 0.00),
    (1, 3, 3, 2, 2.25, 15.00),
    (2, 1, 2, 1, 3.50, 0.00),
    (2, 2, 6, 1, 3.40, 15.00),
    (2, 3, 4, 6, 0.60, 0.00),
    (3, 1, 5, 20, 1.20, 0.00),
    (3, 2, 6, 6, 3.40, 15.00),
    (3, 3, 8, 5, 4.20, 15.00),
    (3, 4, 3, 12, 2.25, 15.00),
    (4, 1, 4, 2, 0.60, 0.00),
    (4, 2, 10, 1.5, 1.00, 0.00),
    (5, 1, 7, 3, 1.85, 15.00),
    (5, 2, 3, 2, 2.25, 15.00),
    (5, 3, 2, 2, 3.50, 0.00),
    (6, 1, 8, 2, 4.20, 15.00),
    (6, 2, 1, 6, 1.10, 0.00),
    (6, 3, 7, 2, 1.85, 15.00),
    (7, 1, 6, 2, 3.40, 15.00),
    (7, 2, 3, 1, 2.25, 15.00),
    (8, 1, 1, 24, 1.10, 0.00),
    (8, 2, 4, 24, 0.60, 0.00),
    (8, 3, 5, 10, 1.20, 0.00),
    (8, 4, 6, 3, 3.40, 15.00);
```

---

# 6. CONSULTAS DE ÁLGEBRA RELACIONAL Y SU TRADUCCIÓN A SQL

### Consulta 1 — Selección (σ) sobre una reunión de dos tablas

**Descripción del objetivo de la consulta:**

Se desea conocer qué productos activos necesitan reposición, es decir, aquellos cuyo stock_actual es menor que su stock_minimo, mostrando además el nombre de la categoría a la que pertenecen (requiere relacionar Producto con Categoria).

**Álgebra Relacional:**

```text
π id_producto, Producto.nombre, Categoria.nombre, stock_actual, stock_minimo (
    σ activo = TRUE ∧ stock_actual < stock_minimo (
        Producto ⋈ Producto.id_categoria = Categoria.id_categoria Categoria
    )
)
```

**SQL:**

```sql
SELECT p.id_producto, p.nombre, c.nombre AS categoria,
       p.stock_actual, p.stock_minimo
FROM Producto p
JOIN Categoria c ON p.id_categoria = c.id_categoria
WHERE p.activo = TRUE
  AND p.stock_actual < p.stock_minimo;
```

**Resultado esperado:**

Se obtienen 2 productos: Atún en lata 170g (categoría Abarrotes, stock 3 de mínimo 12) y Banano (categoría Frutas y Verduras, stock 6 de mínimo 10). El Cloro no aparece porque está inactivo [2].

---

### Consulta 2 — Proyección (π) con Reunión natural (⋈) de tres tablas

**Descripción:**

Obtener únicamente las columnas necesarias para un listado de facturas: número, fecha, nombre del cliente, apellido del empleado que atendió y total. Para ello se relacionan Factura, Cliente y Empleado mediante sus claves foráneas.

**Álgebra Relacional:**

```text
π numero_factura, fecha_emision, Cliente.nombres, Cliente.apellidos, Empleado.apellidos, total (
    (Factura ⋈ Factura.id_cliente = Cliente.id_cliente Cliente)
    ⋈ Factura.id_empleado = Empleado.id_empleado Empleado
)
```

*Nota: ORDER BY no es un operador del álgebra relacional básica; solo ordena la presentación del resultado.*

**SQL:**

```sql
SELECT f.numero_factura, f.fecha_emision,
       c.nombres, c.apellidos,
       e.apellidos AS atendido_por, f.total
FROM Factura f
JOIN Cliente c ON f.id_cliente = c.id_cliente
JOIN Empleado e ON f.id_empleado = e.id_empleado
ORDER BY f.fecha_emision;
```

**Resultado esperado:**

Se muestran las 8 facturas con solo 6 columnas, ordenadas por fecha. Por ejemplo, la factura 001-001-000000003 aparece a nombre de Distribuidora El Sol S.A., atendida por Ordóñez Vega, con un total de 102.66. Los apellidos del cliente salen NULL para la empresa y para Consumidor Final.

---

### Consulta 3 — Selección con Proyección (σ + π combinados)

**Descripción:**

Combina varias condiciones de filtro con la selección de columnas: listar las facturas emitidas (no anuladas) de clientes de la ciudad de Loja cuyo total sea de al menos 20 dólares, mostrando número de factura, nombre del cliente y total.

**Álgebra Relacional:**

```text
π numero_factura, nombres, apellidos, total (
    σ estado = 'EMITIDA' ∧ ciudad = 'Loja' ∧ total ≥ 20 (
        Factura ⋈ Factura.id_cliente = Cliente.id_cliente Cliente
    )
)
```

**SQL:**

```sql
SELECT f.numero_factura, c.nombres, c.apellidos, f.total
FROM Factura f
JOIN Cliente c ON f.id_cliente = c.id_cliente
WHERE f.estado = 'EMITIDA'
  AND c.ciudad = 'Loja'
  AND f.total >= 20
ORDER BY f.total DESC;
```

**Resultado esperado:**

Se obtienen 3 facturas: 001-001-000000003 (Distribuidora El Sol S.A., 102.66), 001-001-000000008 (Distribuidora El Sol S.A., 64.53) y 001-001-000000006 (Ana Gabriela Armijos León, 20.51). Se excluyen la factura anulada, la del cliente de Cuenca y las de menor valor.

---

### Consulta 4 — Reunión natural / JOIN (operador: ⋈)

**Descripción:**

Combina registros de cinco tablas relacionadas mediante claves foráneas para mostrar el detalle de ventas de los productos de la categoría Abarrotes en facturas emitidas: número de factura, cliente, producto, cantidad, precio unitario y subtotal de la línea.

**Álgebra Relacional:**

```text
π numero_factura, Cliente.nombres, Producto.nombre, cantidad, precio_unitario, subtotal (
    σ Categoria.nombre = 'Abarrotes' ∧ estado = 'EMITIDA' (
        Factura
        ⋈ Factura.id_factura = Detalle_Factura.id_factura Detalle_Factura
        ⋈ Detalle_Factura.id_producto = Producto.id_producto Producto
        ⋈ Producto.id_categoria = Categoria.id_categoria Categoria
        ⋈ Factura.id_cliente = Cliente.id_cliente Cliente
    )
)
```

**SQL:**

```sql
SELECT f.numero_factura, c.nombres, p.nombre AS producto,
       d.cantidad, d.precio_unitario, d.subtotal
FROM Factura f
JOIN Detalle_Factura d ON f.id_factura = d.id_factura
JOIN Producto p ON d.id_producto = p.id_producto
JOIN Categoria ca ON p.id_categoria = ca.id_categoria
JOIN Cliente c ON f.id_cliente = c.id_cliente
WHERE ca.nombre = 'Abarrotes'
  AND f.estado = 'EMITIDA'
ORDER BY f.numero_factura, d.num_linea;
```

**Resultado esperado:**

Se obtienen 8 líneas de venta de Arroz grano largo, Aceite vegetal 1L y Atún en lata en las facturas 1, 2, 3, 5, 6 y 8. Por ejemplo, la factura 003 incluye 20 kg de arroz (24.00) y 6 unidades de aceite (20.40) para Distribuidora El Sol S.A. La línea de aceite de la factura anulada (007) no aparece.

---

### Consulta 5 — Consulta con agrupamiento y función de agregación

**Descripción:**

Usa agrupamiento y funciones de agregación (COUNT, SUM) para obtener un resumen de ventas por categoría de producto en facturas emitidas, mostrando solo las categorías cuyo total vendido sea de al menos 10 dólares. Relaciona Detalle_Factura, Factura, Producto y Categoria.

**Álgebra Relacional:**

```text
σ total_vendido ≥ 10 (
    γ Categoria.nombre; COUNT(DISTINCT id_factura) → num_facturas, SUM(subtotal) → total_vendido (
        σ estado = 'EMITIDA' (
            Detalle_Factura
            ⋈ Detalle_Factura.id_factura = Factura.id_factura Factura
            ⋈ Detalle_Factura.id_producto = Producto.id_producto Producto
            ⋈ Producto.id_categoria = Categoria.id_categoria Categoria
        )
    )
)
```

*Nota: γ representa la operación de agrupamiento y agregación en álgebra relacional extendida, y la σ exterior cumple la función de la cláusula HAVING.*

**SQL:**

```sql
SELECT ca.nombre AS categoria,
       COUNT(DISTINCT f.id_factura) AS num_facturas,
       SUM(d.subtotal) AS total_vendido
FROM Detalle_Factura d
JOIN Factura f ON d.id_factura = f.id_factura
JOIN Producto p ON d.id_producto = p.id_producto
JOIN Categoria ca ON p.id_categoria = ca.id_categoria
WHERE f.estado = 'EMITIDA'
GROUP BY ca.nombre
HAVING SUM(d.subtotal) >= 10
ORDER BY total_vendido DESC;
```

**Resultado esperado:**

Se obtienen 4 categorías: Abarrotes (6 facturas, 82.25), Bebidas (6 facturas, 55.20), Lácteos (5 facturas, 47.90) y Limpieza (2 facturas, 29.40). Frutas y Verduras queda fuera porque solo acumula 2.70 en ventas emitidas [3].

---

# 7. CONCLUSIONES

1. Con este proyecto pude comprender la importancia de realizar primero el modelado de una base de datos antes de comenzar con su implementación. Al elaborar el diagrama Entidad-Relación y transformarlo al modelo relacional, aprendí a identificar las entidades, sus atributos y las relaciones que existen entre ellas. También pude entender mejor cómo organizar la información de las facturas y sus detalles, y por qué es necesario establecer correctamente las claves primarias y foráneas para mantener los datos relacionados.

2. Mediante la implementación de las tablas y las sentencias SQL pude poner en práctica los conocimientos adquiridos durante la unidad. Aprendí cómo funcionan las claves primarias, claves foráneas y restricciones para evitar errores en los registros y mantener la integridad de la información. Además, comprendí la utilidad de los triggers, especialmente para automatizar el descuento del stock de los productos cada vez que se registra una venta, lo que permite llevar un mejor control del inventario.

3. Al trabajar con el álgebra relacional y las consultas SQL pude mejorar mi comprensión sobre cómo obtener información de una o varias tablas. Aprendí a utilizar operaciones de selección, proyección y combinación para organizar los datos y posteriormente aplicar consultas con SELECT, JOIN, WHERE y GROUP BY. Gracias a esto, comprendí que una base de datos no solamente permite almacenar información, sino también realizar consultas que facilitan el control de las ventas, los productos y las actividades de la tienda.

<div align="center">

**PROYECTO DE BASE DE DATOS**

*Loja - Ecuador*

</div>


