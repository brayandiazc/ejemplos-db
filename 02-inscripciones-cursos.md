# Proyecto 2 — Inscripciones a Cursos

## ¿Qué vamos a construir?

- **estudiantes**: personas que toman cursos.
- **cursos**: cada curso ofrecido (título, modalidad, fechas, etc.).
- **inscripciones**: “puente” que relaciona un estudiante con un curso y su estado (pagada, pendiente, cancelada).

**Conceptos clave (en lenguaje sencillo):**

- **PRIMARY KEY (PK)**: columna que identifica de forma única cada fila (ej. `id`).
- **FOREIGN KEY (FK)**: columna que “apunta” a la PK de otra tabla para asegurar que la relación exista.
- **UNIQUE**: obliga a que no se repitan valores (ej. un mismo email).
- **CHECK**: regla que debe cumplirse (ej. edad ≥ 14).
- **DEFAULT**: valor por defecto si no lo especificas (ej. fecha actual).
- **Tipos de datos**:

  - `VARCHAR(n)`: texto hasta _n_ caracteres.
  - `INTEGER`: número entero.
  - `NUMERIC(p,s)`: número decimal con precisión, ideal para dinero.
  - `DATE`: solo fecha (año/mes/día).
  - `TIMESTAMP`: fecha y hora (sin zona horaria).
  - `BIGSERIAL`: entero grande auto-incremental (se usa mucho para `id`).

## Paso 1 Conectarte y crear la base

**¿Qué hace cada comando?**

- `psql -U postgres -h localhost`: abre la consola de PostgreSQL usando el usuario `postgres` en tu máquina.
- `CREATE DATABASE academia;`: crea una base de datos vacía llamada `academia`.
- `\c academia`: te “cambia”/conecta a esa base.

```bash
psql -U postgres -h localhost
```

```sql
CREATE DATABASE academia;
\c academia
```

**Cómo verifico que funcionó:** el prompt de `psql` debe mostrar `academia=#`.

## Paso 2 Crear la tabla `estudiantes`

**Qué representa cada columna y restricción:**

- `id BIGSERIAL PRIMARY KEY`: número autoincremental único.
- `nombre VARCHAR(100) NOT NULL`: texto obligatorio (si no lo mandas, falla).
- `email VARCHAR(150) NOT NULL UNIQUE`: texto obligatorio y sin repetidos.
- `edad INTEGER CHECK (edad >= 14)`: si pones 13 o menos, da error.
- `ciudad VARCHAR(80)`: opcional (puede quedar vacío).
- `creado_en TIMESTAMP DEFAULT NOW()`: si no mandas nada, se guarda la fecha y hora actual.

```sql
CREATE TABLE estudiantes (
  id        BIGSERIAL    PRIMARY KEY,
  nombre    VARCHAR(100) NOT NULL,
  email     VARCHAR(150) NOT NULL UNIQUE,
  edad      INTEGER      CHECK (edad >= 14),
  ciudad    VARCHAR(80),
  creado_en TIMESTAMP    DEFAULT NOW()
);
```

**Qué pasa si me equivoco:**

- Email repetido → error por `UNIQUE`.
- Edad menor a 14 → error por `CHECK`.
- Falta `nombre` o `email` → error por `NOT NULL`.

**Cómo veo la estructura después de crearla:** `\d estudiantes`

## Paso 3 Crear la tabla `cursos`

**Puntos clave:**

- `titulo UNIQUE NOT NULL`: no se permiten cursos con el mismo título.
- `modalidad` limitada a dos opciones (regla `CHECK`).
- `cupo` y `precio` no pueden ser negativos.
- `fecha_fin` no puede ser anterior a `fecha_inicio`.

```sql
CREATE TABLE cursos (
  id            BIGSERIAL     PRIMARY KEY,
  titulo        VARCHAR(120)  NOT NULL UNIQUE,
  modalidad     VARCHAR(12)   NOT NULL CHECK (modalidad IN ('presencial','virtual')),
  cupo          INTEGER       NOT NULL DEFAULT 0 CHECK (cupo >= 0),
  precio        NUMERIC(10,2) NOT NULL CHECK (precio >= 0),
  fecha_inicio  DATE          NOT NULL,
  fecha_fin     DATE          NOT NULL,
  CHECK (fecha_fin >= fecha_inicio)
);
```

**Errores típicos:**

- `modalidad='híbrido'` → falla por `CHECK`.
- `fecha_fin < fecha_inicio` → falla por `CHECK`.

## Paso 4 Crear la tabla `inscripciones`

**Por qué existe esta tabla:** un estudiante puede tomar muchos cursos y un curso puede tener muchos estudiantes. `inscripciones` “conecta” ambos lados.

**Columnas y reglas:**

- `estudiante_id` y `curso_id` son **FOREIGN KEY**: garantizan que la inscripción solo apunte a estudiantes y cursos que existen.
- `estado` tiene valores permitidos (`CHECK`) y por defecto es `'pendiente'`.
- `UNIQUE (estudiante_id, curso_id)` evita que la misma persona se inscriba dos veces al mismo curso.
- `fecha_inscripcion` toma la hora actual si no mandas nada.

```sql
CREATE TABLE inscripciones (
  id                BIGSERIAL   PRIMARY KEY,
  estudiante_id     BIGINT      NOT NULL REFERENCES estudiantes(id),
  curso_id          BIGINT      NOT NULL REFERENCES cursos(id),
  estado            VARCHAR(12) NOT NULL DEFAULT 'pendiente'
                    CHECK (estado IN ('pendiente','pagada','cancelada')),
  fecha_inscripcion TIMESTAMP   DEFAULT NOW(),
  UNIQUE (estudiante_id, curso_id)
);
```

**Errores típicos:**

- Usar un `estudiante_id` que no existe → falla por FK.
- Repetir la misma pareja `(estudiante_id, curso_id)` → falla por `UNIQUE`.

## Paso 5 Insertar datos de ejemplo

**¿Por qué insertar así?** Para practicar:

- Inserciones múltiples (ahorran tiempo).
- Reglas `NOT NULL`, `UNIQUE`, `CHECK` y FKs en acción.

```sql
-- Estudiantes
INSERT INTO estudiantes (nombre,email,edad,ciudad) VALUES
('Ana Pérez','ana@example.com',19,'Bogotá'),
('Luis Mora','luis@example.com',22,'Medellín'),
('Sara Gil','sara@example.com',17,'Cali');

-- Cursos
INSERT INTO cursos (titulo,modalidad,cupo,precio,fecha_inicio,fecha_fin) VALUES
('SQL desde cero','virtual',30,250000,'2025-11-05','2025-11-20'),
('Modelado de datos','presencial',20,300000,'2025-11-10','2025-11-25'),
('Consultas avanzadas','virtual',25,280000,'2025-11-15','2025-11-30');

-- Inscripciones (revisa los IDs reales en tu base con SELECT)
INSERT INTO inscripciones (estudiante_id,curso_id,estado) VALUES
(1,1,'pagada'),
(2,1,'pendiente'),
(3,2,'pagada');
```

**¿Cómo confirmo los IDs?**

- `SELECT id, nombre FROM estudiantes;`
- `SELECT id, titulo FROM cursos;`

## Paso 6 Consultas básicas (leer y combinar datos)

**Listar estudiantes por nombre (A-Z):**

```sql
SELECT id, nombre, email, ciudad
FROM estudiantes
ORDER BY nombre;
```

**Ver cursos virtuales del más caro al más barato:**

```sql
SELECT id, titulo, modalidad, precio
FROM cursos
WHERE modalidad = 'virtual'
ORDER BY precio DESC;
```

**¿Quién se inscribió a qué curso? (JOIN = unir tablas por su relación):**

```sql
SELECT
  i.id,
  e.nombre AS estudiante,
  c.titulo AS curso,
  i.estado,
  i.fecha_inscripcion
FROM inscripciones i
JOIN estudiantes e ON e.id = i.estudiante_id
JOIN cursos c      ON c.id = i.curso_id
ORDER BY i.id;
```

**¿Cuántos inscritos tiene cada curso? (agregación con COUNT y GROUP BY):**

```sql
SELECT
  c.titulo,
  COUNT(*) AS total_inscritos
FROM inscripciones i
JOIN cursos c ON c.id = i.curso_id
GROUP BY c.titulo
ORDER BY total_inscritos DESC;
```

## Paso 7 Probar las restricciones (errores que deben ocurrir)

**Para entender que la base “se cuida sola”:**

```sql
-- 1) Email duplicado → UNIQUE
INSERT INTO estudiantes (nombre,email,edad)
VALUES ('Prueba','ana@example.com',20);

-- 2) Fechas incoherentes → CHECK
INSERT INTO cursos (titulo,modalidad,cupo,precio,fecha_inicio,fecha_fin)
VALUES ('Prueba fechas','virtual',10,100000,'2025-11-20','2025-11-10');

-- 3) Curso inexistente → FOREIGN KEY
INSERT INTO inscripciones (estudiante_id,curso_id,estado)
VALUES (1,999,'pagada');
```

**Qué mensaje esperar (en general):**

- “duplicate key value violates unique constraint …” (por `UNIQUE`)
- “new row for relation … violates check constraint …” (por `CHECK`)
- “insert or update on table … violates foreign key constraint …” (por `FK`)

## Paso 8 Actualizar información existente (UPDATE)

**Cambiar datos puntuales de una fila:**

```sql
-- Cambiar el email de una persona
UPDATE estudiantes
SET email = 'ana.perez@ejemplo.com'
WHERE id = 1;

-- Subir el precio de un curso
UPDATE cursos
SET precio = 320000
WHERE id = 2;  -- "Modelado de datos"

-- Cambiar el estado de una inscripción
UPDATE inscripciones
SET estado = 'pagada'
WHERE id = 2;
```

**Actualizar con condiciones más elaboradas:**

```sql
-- Aumentar 10% el precio de todos los cursos virtuales
UPDATE cursos
SET precio = precio * 1.10
WHERE modalidad = 'virtual';

-- Marcar como 'cancelada' cualquier inscripción pendiente con más de 30 días
UPDATE inscripciones
SET estado = 'cancelada'
WHERE estado = 'pendiente'
  AND fecha_inscripcion < NOW() - INTERVAL '30 days';
```

**Ver qué cambiaste (útil para auditar):**

```sql
UPDATE cursos
SET precio = precio * 0.95
WHERE modalidad = 'presencial'
RETURNING id, titulo, precio;
```

## Paso 9 Insertar datos en gran volumen

### 9.1 Inserciones múltiples “normales”

```sql
INSERT INTO estudiantes (nombre,email,edad,ciudad) VALUES
('Carlos Ruiz','carlos@example.com',28,'Bogotá'),
('Marta León','marta@example.com',24,'Barranquilla'),
('Diego Patiño','diego@example.com',31,'Cali');
```

### 9.2 Generar datos masivos con `generate_series` (rápido para practicar)

```sql
-- 100 estudiantes de prueba: est001, est002, ..., est100
INSERT INTO estudiantes (nombre, email, edad, ciudad)
SELECT
  'Estudiante ' || to_char(gs, 'FM000') AS nombre,
  'est' || to_char(gs, 'FM000') || '@example.com' AS email,
  (14 + (gs % 30))::int AS edad,             -- entre 14 y 43
  CASE (gs % 5)
    WHEN 0 THEN 'Bogotá'
    WHEN 1 THEN 'Medellín'
    WHEN 2 THEN 'Cali'
    WHEN 3 THEN 'Barranquilla'
    ELSE 'Bucaramanga'
  END AS ciudad
FROM generate_series(1,100) AS gs;
```

```sql
-- 20 cursos virtuales y 20 presenciales de prueba
INSERT INTO cursos (titulo, modalidad, cupo, precio, fecha_inicio, fecha_fin)
SELECT
  'Curso Virtual ' || gs,
  'virtual',
  30 + (gs % 10),                      -- cupo variable
  200000 + (gs * 1000),                -- precios distintos
  DATE '2025-12-01' + (gs || ' days')::interval,
  DATE '2025-12-10' + (gs || ' days')::interval
FROM generate_series(1,20) AS gs;

INSERT INTO cursos (titulo, modalidad, cupo, precio, fecha_inicio, fecha_fin)
SELECT
  'Curso Presencial ' || gs,
  'presencial',
  20 + (gs % 10),
  250000 + (gs * 1000),
  DATE '2026-01-05' + (gs || ' days')::interval,
  DATE '2026-01-20' + (gs || ' days')::interval
FROM generate_series(1,20) AS gs;
```

```sql
-- Inscripciones masivas: tomar 200 combinaciones estudiante-curso válidas
INSERT INTO inscripciones (estudiante_id, curso_id, estado)
SELECT e.id, c.id,
       CASE (e.id + c.id) % 3
         WHEN 0 THEN 'pagada'
         WHEN 1 THEN 'pendiente'
         ELSE 'cancelada'
       END
FROM (SELECT id FROM estudiantes ORDER BY id LIMIT 100) e
JOIN (SELECT id FROM cursos ORDER BY id LIMIT 20) c
  ON (e.id + c.id) % 5 = 0         -- simple regla para no romper UNIQUE
LIMIT 200;
```

> Nota: si alguna inserción masiva falla, revisa restricciones `UNIQUE`/`CHECK` y ajusta la lógica (por ejemplo, cambios en la condición del JOIN para evitar duplicados).

## Paso 10 Limpiar campos dentro de una fila (sin borrar la fila)

A veces **no quieres borrar la fila**, solo “vaciar”/normalizar algunos campos.

```sql
-- Quitar la ciudad (dejarla en NULL) a quienes no la informaron correctamente
UPDATE estudiantes
SET ciudad = NULL
WHERE ciudad IN ('', 'N/A', 'Desconocido');

-- Regresar el estado a su valor por defecto (usando DEFAULT)
UPDATE inscripciones
SET estado = DEFAULT
WHERE estado NOT IN ('pendiente','pagada','cancelada');
```

## Paso 11 Eliminar información específica con WHERE (DELETE selectivo)

**Eliminar registros “que no sean necesarios” tras haber sido insertados:**

```sql
-- Borrar inscripciones canceladas anteriores a 2025-11-01
DELETE FROM inscripciones
WHERE estado = 'cancelada'
  AND fecha_inscripcion < '2025-11-01'::timestamp
RETURNING id, estudiante_id, curso_id;

-- Borrar estudiantes de prueba por patrón de email
DELETE FROM estudiantes
WHERE email LIKE 'est___@example.com'  -- est001, est002, etc.
RETURNING id, email;
```

**Eliminar usando datos de otra tabla (DELETE ... USING):**

```sql
-- Borrar inscripciones de cursos cuyo cupo es 0
DELETE FROM inscripciones i
USING cursos c
WHERE i.curso_id = c.id
  AND c.cupo = 0
RETURNING i.id, i.estudiante_id, i.curso_id;
```

**Patrón seguro (recomendado):** primero inspecciona, luego borra con la misma condición.

```sql
-- Ver qué se borraría:
SELECT i.*
FROM inscripciones i
JOIN cursos c ON c.id = i.curso_id
WHERE c.cupo = 0;

-- Si está bien, ejecuta el DELETE con la misma condición
DELETE FROM inscripciones i
USING cursos c
WHERE i.curso_id = c.id
  AND c.cupo = 0;
```

## Paso 12 Eliminar una fila completa (DELETE por PK)

**Importante:** en nuestro diseño, `inscripciones` depende de `estudiantes` y `cursos`.
Para eliminar un estudiante o un curso, **primero elimina sus inscripciones** (o usa `TRUNCATE ... CASCADE` si vas a vaciar).

```sql
-- 12.1 Eliminar todas las inscripciones del estudiante 1
DELETE FROM inscripciones
WHERE estudiante_id = 1;

-- 12.2 Ahora sí, eliminar la fila del estudiante 1
DELETE FROM estudiantes
WHERE id = 1;
```

**Opción alternativa (cambiar la FK a ON DELETE CASCADE):**

```sql
-- Si prefieres que al borrar un estudiante se borren sus inscripciones automáticamente:
ALTER TABLE inscripciones
  DROP CONSTRAINT inscripciones_estudiante_id_fkey,
  ADD CONSTRAINT inscripciones_estudiante_id_fkey
  FOREIGN KEY (estudiante_id) REFERENCES estudiantes(id) ON DELETE CASCADE;

-- Análogo para curso_id:
ALTER TABLE inscripciones
  DROP CONSTRAINT inscripciones_curso_id_fkey,
  ADD CONSTRAINT inscripciones_curso_id_fkey
  FOREIGN KEY (curso_id) REFERENCES cursos(id) ON DELETE CASCADE;
```

> Con `ON DELETE CASCADE`, bastaría:

```sql
DELETE FROM estudiantes WHERE id = 1;
```

## Paso 13 Vaciar/limpiar una tabla completa (TRUNCATE vs DELETE)

**Vaciar rápido (sin borrar la tabla):**

```sql
-- 13.1 Vaciar inscripciones (más rápido que DELETE)
TRUNCATE TABLE inscripciones;

-- 13.2 Si hay dependencias, usa CASCADE
TRUNCATE TABLE inscripciones CASCADE;
```

**Vaciar selectivamente con WHERE (límpiezas parciales):**

```sql
-- Borra solo inscripciones 'pendiente'
DELETE FROM inscripciones
WHERE estado = 'pendiente';

-- Borra inscripciones de cursos que terminaron antes de cierta fecha
DELETE FROM inscripciones i
USING cursos c
WHERE i.curso_id = c.id
  AND c.fecha_fin < DATE '2025-11-15';
```

> Regla práctica: **TRUNCATE** para vaciar todo **muy rápido**; **DELETE ... WHERE** para limpiezas selectivas.

## Paso 14 Eliminar una columna (ALTER TABLE ... DROP COLUMN)

Si una columna ya no es necesaria:

```sql
-- Quitar la columna 'ciudad' de estudiantes
ALTER TABLE estudiantes
DROP COLUMN ciudad;

-- Quitar 'cupo' de cursos (ejemplo). Ojo: asegúrate de no usarla en CHECKs/consultas
ALTER TABLE cursos
DROP COLUMN cupo;
```

> Si existe un `CHECK` o `INDEX` que la usa, elimínalos o modifícalos antes.

## Paso 15 Eliminar una tabla completa (DROP TABLE)

**Orden recomendado con FKs:** primero tablas “hijas” (inscripciones), luego “padres” (cursos/estudiantes).

```sql
-- 15.1 Si solo vas a borrar la tabla hija:
DROP TABLE inscripciones;

-- 15.2 Si quieres borrar todas aunque tengan dependencias:
-- (Cuidado: CASCADE elimina también objetos dependientes)
DROP TABLE inscripciones CASCADE;
DROP TABLE cursos CASCADE;
DROP TABLE estudiantes CASCADE;
```

> Alternativa segura: si solo vas a dejar de usar la tabla, considera **TRUNCATE** o archivarla en otra base.

## Paso 16 Eliminar la base de datos completa (DROP DATABASE)

**No puedes borrar una base si estás dentro de ella o hay conexiones activas.**

```sql
-- 16.1 Sal del contexto de 'academia' y vuelve a 'postgres'
\c postgres

-- 16.2 (Opcional) Forzar desconexión de clientes conectados a 'academia'
-- Requiere permisos suficientes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'academia'
  AND pid <> pg_backend_pid();

-- 16.3 Ahora sí, borrar la base
DROP DATABASE academia;
```

## Paso 17 Consejos de seguridad al modificar/borrar

1. **Usa transacciones** cuando hagas cambios grandes:

   ```sql
   BEGIN;

   -- tus UPDATE/DELETE/TRUNCATE aquí

   -- Revisa: si todo bien
   COMMIT;
   -- Si no:
   -- ROLLBACK;
   ```

2. **Vista previa antes de borrar:** ejecuta un `SELECT` con el mismo `WHERE` que usarás en el `DELETE`.
3. **Respeta el orden por FKs:** primero `inscripciones`, luego `cursos`/`estudiantes` (a menos que uses `ON DELETE CASCADE` o `TRUNCATE ... CASCADE`).
4. **Evita borrar columnas “calientes”:** confirma que no participen en `CHECK`, `INDEX`, vistas o consultas críticas.
5. **Respaldos:** antes de limpiezas masivas, haz un `pg_dump` rápido.
