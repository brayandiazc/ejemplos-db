# 🏪 Proyecto: Tienda Relámpago

Este mini-proyecto te guía paso a paso para crear una base de datos en PostgreSQL, limpiar datos desordenados y aplicar reglas básicas para mantener su integridad.
El objetivo es practicar comandos SQL esenciales: creación, inserción, limpieza, alteración y consulta de datos.

## **Paso 1 — Crear la base y conectarte**

**Qué harás:** crear una base de datos nueva y conectarte a ella.

```sql
CREATE DATABASE tienda_relampago;

\c tienda_relampago
```

**Qué significa:**

- `CREATE DATABASE` crea una nueva base llamada **tienda_relampago**.
- `\c` te conecta a esa base (en PostgreSQL).

**Qué deberías ver:**
Un mensaje como `You are now connected to database "tienda_relampago"`.

## **Paso 2 — Crear tablas “rápidas y sucias”**

**Qué harás:** crear tres tablas sin validaciones, todo tipo `TEXT` (sirve para cargar datos imperfectos).

```sql
CREATE TABLE productos (codigo TEXT, nombre TEXT, precio TEXT, stock TEXT, categoria TEXT);

CREATE TABLE ventas (fecha TEXT, producto_codigo TEXT, cantidad TEXT, total TEXT);

CREATE TABLE clientes (doc TEXT, nombre TEXT, telefono TEXT, ciudad TEXT );

\dt
```

**Qué significa:**

- `CREATE TABLE` crea cada tabla con sus columnas.
- `\dt` lista todas las tablas creadas.

## **Paso 3 — Insertar algunos datos “no perfectos”**

**Qué harás:** ingresar filas con errores o formatos distintos.

```sql
INSERT INTO productos VALUES ('A-01','Café molido 500g','12,5 USD','50u','alimentos');
INSERT INTO productos VALUES ('B-99','Taza cerámica','18.000','','hogar');
INSERT INTO productos VALUES ('X-7','Filtro papel #2','3.5','100',NULL);

INSERT INTO ventas VALUES ('2025/10/01','A-01','2','??');
INSERT INTO ventas VALUES ('ayer','B-99','-1','0');
INSERT INTO ventas VALUES ('01-10-25','X-7','10','35');

INSERT INTO clientes VALUES ('CC123','Ana Pérez','(57) 300-abc-9999','Bogota');
INSERT INTO clientes VALUES ('correo@ejemplo','Luis Mora','3001234567','');
```

**Qué pasa:**
PostgreSQL te responderá `INSERT 0 1` por cada fila.
Ya hay datos, aunque mezclados y con errores (como precios con texto, fechas raras, etc.).

## **Paso 4 — Ver lo que hay**

**Qué harás:** mirar el contenido de cada tabla.

```sql
SELECT * FROM productos;
SELECT * FROM ventas;
SELECT * FROM clientes;
```

**Qué notarás:**

- Precios con comas o “USD”.
- Fechas con distintos formatos.
- Cantidades negativas o vacías.

## **Paso 5 — Limpiar “suave” los productos**

**Qué harás:** quitar texto innecesario, reemplazar comas y rellenar valores vacíos.

```sql
UPDATE productos SET precio = REPLACE(precio,',','.');
UPDATE productos SET precio = REPLACE(precio,' USD','');
UPDATE productos SET stock = REPLACE(stock,'u','');
UPDATE productos SET stock = NULLIF(stock,'');
UPDATE productos SET categoria = COALESCE(NULLIF(categoria,''),'sin_categoria');
```

**Explicación:**

- `REPLACE` cambia partes del texto (por ejemplo, quita “USD”).
- `NULLIF` convierte valores vacíos en `NULL`.
- `COALESCE` pone un valor por defecto si algo está en `NULL`.

## **Paso 6 — Convertir tipos en productos**

**Qué harás:** pasar los precios y el stock a valores numéricos reales.

```sql
ALTER TABLE productos ALTER COLUMN precio TYPE NUMERIC(12,2) USING NULLIF(precio,'')::NUMERIC;
ALTER TABLE productos ALTER COLUMN stock TYPE INTEGER USING NULLIF(stock,'')::INTEGER;
```

**Qué significa:**

- `ALTER TABLE` cambia la definición de la columna.
- `USING ...::NUMERIC` convierte los textos a números.

## **Paso 7 — Limpiar “suave” las ventas**

**Qué harás:** uniformar fechas, corregir cantidades negativas y limpiar totales.

```sql
UPDATE ventas SET fecha = TO_CHAR(CURRENT_DATE - INTERVAL '1 day','YYYY-MM-DD') WHERE LOWER(fecha)='ayer';
UPDATE ventas SET fecha = REPLACE(fecha,'/','-');
UPDATE ventas SET cantidad = '0' WHERE cantidad LIKE '-%';
UPDATE ventas SET total = REPLACE(total,'?','');
UPDATE ventas SET total = REPLACE(total,',','.');
```

**Qué significa:**

- Las fechas quedan todas con guiones (`-`).
- “ayer” se reemplaza por la fecha real de ayer.
- Totales sin símbolos raros.

## **Paso 8 — Convertir tipos en ventas**

**Qué harás:** cambiar las columnas a tipos correctos.

```sql
ALTER TABLE ventas ALTER COLUMN fecha TYPE DATE USING TO_DATE(fecha,'YYYY-MM-DD');
ALTER TABLE ventas ALTER COLUMN cantidad TYPE INTEGER USING NULLIF(cantidad,'')::INTEGER;
ALTER TABLE ventas ALTER COLUMN total TYPE NUMERIC(12,2) USING NULLIF(total,'')::NUMERIC;
```

## **Paso 9 — Reglas básicas (CHECKS y DEFAULTS)**

**Qué harás:** agregar validaciones.

```sql
ALTER TABLE productos ALTER COLUMN stock SET DEFAULT 0;
ALTER TABLE productos ADD CONSTRAINT chk_precio_positivo CHECK (precio > 0);
ALTER TABLE productos ADD CONSTRAINT chk_stock_no_negativo CHECK (stock >= 0);
ALTER TABLE ventas ADD CONSTRAINT chk_cantidad_no_negativa CHECK (cantidad >= 0);
```

**Qué hace:**
Ahora PostgreSQL no dejará insertar valores inválidos (por ejemplo, precios negativos).

## **Paso 10 — Llaves primaria y foránea**

**Qué harás:** conectar las tablas con relaciones.

```sql
ALTER TABLE productos ADD CONSTRAINT productos_pkey PRIMARY KEY (codigo);
ALTER TABLE ventas ADD COLUMN id BIGSERIAL;
ALTER TABLE ventas ADD CONSTRAINT ventas_pkey PRIMARY KEY (id);
ALTER TABLE ventas ADD CONSTRAINT ventas_producto_fk FOREIGN KEY (producto_codigo) REFERENCES productos(codigo) ON UPDATE CASCADE ON DELETE RESTRICT;
```

**Explicación:**

- Cada producto tiene un código único (PK).
- Cada venta tiene un id autoincremental.
- No se puede vender un producto que no exista (FK).

## **Paso 11 — Consultas útiles**

### Ver todas las ventas con nombre del producto

```sql
SELECT v.id, v.fecha, v.producto_codigo, p.nombre AS producto, v.cantidad, p.precio, (v.cantidad * p.precio)::NUMERIC(12,2) AS total_calculado FROM ventas v JOIN productos p ON p.codigo = v.producto_codigo ORDER BY v.fecha, v.id;
```

### Ver total vendido por producto

```sql
SELECT p.nombre, SUM(v.cantidad * p.precio)::NUMERIC(12,2) AS total_vendido FROM ventas v JOIN productos p ON p.codigo = v.producto_codigo GROUP BY p.nombre ORDER BY total_vendido DESC;
```

**Explicación:**

- `JOIN` combina datos de ambas tablas.
- `SUM()` suma los valores de ventas.
- `GROUP BY` agrupa por producto.

## **Paso 12 — Probar las reglas**

**Qué harás:** probar que las validaciones funcionan.

```sql
INSERT INTO ventas (fecha,producto_codigo,cantidad,total)
VALUES (CURRENT_DATE,'A-01',-5,10.0);

INSERT INTO ventas (fecha,producto_codigo,cantidad,total)
VALUES (CURRENT_DATE,'NO-EXISTE',1,10.0);
```

**Qué debería pasar:**
Ambas fallarán:

- La primera por cantidad negativa (`CHECK`).
- La segunda porque el producto no existe (`FOREIGN KEY`).

## **Resumen de conceptos clave**

| Comando           | Qué hace                     | Ejemplo                                       |
| ----------------- | ---------------------------- | --------------------------------------------- |
| `CREATE DATABASE` | Crea una nueva base de datos | `CREATE DATABASE tienda_relampago;`           |
| `CREATE TABLE`    | Crea una tabla               | `CREATE TABLE productos (...);`               |
| `INSERT INTO`     | Inserta filas                | `INSERT INTO productos VALUES (...);`         |
| `SELECT * FROM`   | Muestra todas las filas      | `SELECT * FROM productos;`                    |
| `UPDATE`          | Modifica valores existentes  | `UPDATE productos SET precio=...;`            |
| `ALTER TABLE`     | Cambia estructura de tabla   | `ALTER TABLE ventas ADD COLUMN id BIGSERIAL;` |
| `CHECK`           | Valida condiciones           | `CHECK (precio > 0)`                          |
| `FOREIGN KEY`     | Relaciona tablas             | `REFERENCES productos(codigo)`                |
