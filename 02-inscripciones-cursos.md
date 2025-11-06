# Proyecto 2 — Inscripciones a Cursos (PostgreSQL)

## 0) ¿Qué vamos a construir?

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

## 1) Conectarte y crear la base

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

## 2) Crear la tabla `estudiantes`

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

## 3) Crear la tabla `cursos`

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

## 4) Crear la tabla `inscripciones`

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

## 5) Insertar datos de ejemplo

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

## 6) Consultas básicas (leer y combinar datos)

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

## 7) Probar las restricciones (errores que deben ocurrir)

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

## 8) Trucos básicos de consola

- Listar tablas: `\dt`
- Ver estructura de una tabla: `\d nombre_tabla` (ej. `\d cursos`)
- Salir de `psql`: `\q`

## 9) Resumen de diseño (para tu mapa mental)

- **Relación:** `estudiantes (1) — (N) inscripciones (N) — (1) cursos`
- **Reglas clave:**

  - Email único, edad mínima.
  - Modalidad válida, cupo y precio no negativos, fechas coherentes.
  - Una inscripción por estudiante y curso.
  - FKs mantienen la integridad entre tablas.

## 10) Buenas prácticas mínimas (sin complicar)

- Usa `BIGSERIAL` para IDs (simple y funciona).
- Define `NOT NULL`, `UNIQUE` y `CHECK` donde tenga sentido.
- Usa `FOREIGN KEY` para relaciones reales entre tablas.
- Para dinero, usa `NUMERIC(10,2)`.
- Para fechas de cursos, `DATE`. Para “momento exacto” (inscripción), `TIMESTAMP`.
