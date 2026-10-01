---
tipo: estudio
area: cs
estado: completado
tags: [adbd, practica, postgresql, ull, sql, p1]
creado: 2026-10-01
actualizado: 2026-10-01
---

# Práctica 1. Conceptos Fundamentales de PostgreSQL

Entorno de ejecución: Terminal interactiva `psql` en macOS.

```
~
❯ psql postgres
psql (18.6 (Homebrew))
Type "help" for help.

postgres=#
```

---

## 1. Creación de la base de datos

### 1.a. Crear una base de datos llamada `biblioteca`

```
postgres=# CREATE DATABASE biblioteca;
CREATE DATABASE
```

Conexión a la base de datos recién creada:

```
postgres=# \c biblioteca
You are now connected to database "biblioteca" as user "marcos".
```

---

## 2. Creación de usuarios

### 2.a. Crear dos usuarios: `admin_biblio` y `usuario_biblio`

Creación del usuario `admin_biblio` con permisos de administrador sobre la base de datos:

```
biblioteca=# CREATE USER admin_biblio WITH PASSWORD 'admin123';
CREATE ROLE
biblioteca=# GRANT ALL PRIVILEGES ON DATABASE biblioteca TO admin_biblio;
GRANT
```

Creación del usuario `usuario_biblio` con permisos iniciales de conexión (lectura):

```
biblioteca=# CREATE USER usuario_biblio WITH PASSWORD 'user123';
CREATE ROLE
biblioteca=# GRANT CONNECT ON DATABASE biblioteca TO usuario_biblio;
GRANT
```

### 2.b. Crear un rol llamado `lectores` con permisos únicamente de consulta sobre todas las tablas

Creamos el rol `lectores`, asignamos uso del esquema `public` y permisos de `SELECT` para las tablas actuales y futuras:

```
biblioteca=# CREATE ROLE lectores;
CREATE ROLE
biblioteca=# GRANT USAGE ON SCHEMA public TO lectores;
GRANT
biblioteca=# GRANT SELECT ON ALL TABLES IN SCHEMA public TO lectores;
GRANT
biblioteca=# ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO lectores;
ALTER DEFAULT PRIVILEGES
```

### 2.c. Asignar el usuario `usuario_biblio` a este rol

```
biblioteca=# GRANT lectores TO usuario_biblio;
GRANT ROLE
```

### 2.d. Consultar las tablas del sistema para listar todos los usuarios creados (`pg_roles`)

```
biblioteca=# SELECT rolname, rolcanlogin, rolsuper
    FROM pg_roles
    WHERE rolname IN ('admin_biblio', 'usuario_biblio', 'lectores');
    rolname     | rolcanlogin | rolsuper
----------------+-------------+----------
 admin_biblio   | t           | f
 usuario_biblio | t           | f
 lectores       | f           | f
(3 rows)
```

### 2.e. Cambiar la contraseña del usuario `usuario_biblio`

```
biblioteca=# ALTER USER usuario_biblio WITH PASSWORD 'nueva_clave123';
ALTER ROLE
```

### 2.f. Configurar permisos para que el usuario `usuario_biblio` no pueda eliminar registros en ninguna tabla

Revocamos explícitamente el privilegio `DELETE` tanto al usuario como al rol:

```
biblioteca=# REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM usuario_biblio;
REVOKE
biblioteca=# REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM lectores;
REVOKE
```

---

## 3. Creación de tablas

### 3.a. Crear las tablas con sus respectivas claves primarias (usando `TEXT`)

```
biblioteca=# CREATE TABLE autores (
        id_autor SERIAL PRIMARY KEY,
        nombre TEXT,
        nacionalidad TEXT
    );
CREATE TABLE

biblioteca=# CREATE TABLE libros (
        id_libro SERIAL PRIMARY KEY,
        titulo TEXT,
        año_publicacion INT,
        id_autor INT
    );
CREATE TABLE

biblioteca=# CREATE TABLE prestamos (
        id_prestamo SERIAL PRIMARY KEY,
        id_libro INT,
        fecha_prestamo DATE,
        fecha_devolucion DATE,
        usuario_prestatario TEXT
    );
CREATE TABLE
```

### 3.b. Establecer las claves foráneas correspondientes

En `prestamos` se define `ON DELETE CASCADE` para propagar el borrado de libros a sus préstamos:

```
biblioteca=# ALTER TABLE libros
    ADD FOREIGN KEY (id_autor) REFERENCES autores(id_autor);
ALTER TABLE

biblioteca=# ALTER TABLE prestamos
    ADD FOREIGN KEY (id_libro) REFERENCES libros(id_libro) ON DELETE
  CASCADE;
ALTER TABLE
```

---

## 4. Inserción de datos

### 4.a. Insertar al menos 5 autores, 8 libros y 5 préstamos de ejemplo

```
biblioteca=# INSERT INTO autores (nombre, nacionalidad) VALUES
    ('Gabriel García Márquez', 'Colombiana'),
    ('George Orwell', 'Británica'),
    ('Miguel de Cervantes', 'Española'),
    ('Jane Austen', 'Británica'),
    ('Isaac Asimov', 'Rusa');
INSERT 0 5

biblioteca=# INSERT INTO libros (titulo, año_publicacion, id_autor) VALUES
    ('Cien años de soledad', 1967, 1),
    ('El amor en los tiempos del cólera', 1985, 1),
    ('1984', 1949, 2),
    ('Rebelión en la granja', 1945, 2),
    ('Don Quijote de la Mancha', 1605, 3),
    ('Novelas ejemplares', 1613, 3),
    ('Orgullo y prejuicio', 1813, 4),
    ('Fundación', 1951, 5);
INSERT 0 8

biblioteca=# INSERT INTO prestamos (id_libro, fecha_prestamo, fecha_devolucion,
  usuario_prestatario) VALUES
    (3, '2026-09-01', '2026-09-15', 'Carlos Gómez'),
    (1, '2026-09-05', NULL, 'Ana Pérez'),
    (3, '2026-09-10', NULL, 'Carlos Gómez'),
    (5, '2026-09-12', '2026-09-20', 'Lucía Ramos'),
    (8, '2026-09-15', NULL, 'David Ruiz');
INSERT 0 5
```

---

## 5. Consultas básicas

### 5.a. Listar todos los libros con su autor correspondiente

```
biblioteca=# SELECT libros.titulo, autores.nombre AS autor
    FROM libros
    JOIN autores ON libros.id_autor = autores.id_autor;
              titulo               |         autor
-----------------------------------+------------------------
 Cien años de soledad              | Gabriel García Márquez
 El amor en los tiempos del cólera | Gabriel García Márquez
 1984                              | George Orwell
 Rebelión en la granja             | George Orwell
 Don Quijote de la Mancha          | Miguel de Cervantes
 Novelas ejemplares                | Miguel de Cervantes
 Orgullo y prejuicio               | Jane Austen
 Fundación                         | Isaac Asimov
(8 rows)
```

### 5.b. Mostrar los préstamos que aún no tienen fecha de devolución

```
biblioteca=# SELECT id_prestamo, id_libro, fecha_prestamo, usuario_prestatario
    FROM prestamos
    WHERE fecha_devolucion IS NULL;
 id_prestamo | id_libro | fecha_prestamo | usuario_prestatario
-------------+----------+----------------+---------------------
           2 |        1 | 2026-09-05     | Ana Pérez
           3 |        3 | 2026-09-10     | Carlos Gómez
           5 |        8 | 2026-09-15     | David Ruiz
(3 rows)
```

### 5.c. Obtener los autores que tienen más de un libro registrado

```
biblioteca=# SELECT autores.nombre, COUNT(libros.id_libro) AS total_libros
    FROM autores
    JOIN libros ON autores.id_autor = libros.id_autor
    GROUP BY autores.nombre
    HAVING COUNT(libros.id_libro) > 1;
         nombre         | total_libros
------------------------+--------------
 Miguel de Cervantes    |            2
 George Orwell          |            2
 Gabriel García Márquez |            2
(3 rows)
```

---

## 6. Consultas con agregación

### 6.a. Calcular el número total de préstamos realizados

```
biblioteca=# SELECT COUNT(*) AS total_prestamos FROM prestamos;
 total_prestamos
-----------------
               5
(1 row)
```

### 6.b. Obtener el número de libros prestados por cada usuario

```
biblioteca=# SELECT usuario_prestatario, COUNT(*) AS libros_prestados
    FROM prestamos
    GROUP BY usuario_prestatario;
 usuario_prestatario | libros_prestados
---------------------+------------------
 Ana Pérez           |                1
 Lucía Ramos         |                1
 Carlos Gómez        |                2
 David Ruiz          |                1
(4 rows)
```

---

## 7. Modificación de datos

### 7.a. Actualizar la fecha de devolución de un préstamo pendiente

```
biblioteca=# UPDATE prestamos
    SET fecha_devolucion = '2026-09-25'
    WHERE id_prestamo = 2;
UPDATE 1

biblioteca=# SELECT * FROM prestamos WHERE id_prestamo = 2;
 id_prestamo | id_libro | fecha_prestamo | fecha_devolucion | usuario_prestatario
-------------+----------+----------------+------------------+---------------------
           2 |        1 | 2026-09-05     | 2026-09-25       | Ana Pérez
(1 row)
```

### 7.b. Eliminar un libro y comprobar el efecto en la tabla de préstamos

Al haber definido la clave foránea con `ON DELETE CASCADE`, la eliminación del libro propaga el borrado a sus registros asociados en `prestamos`:

```
biblioteca=# DELETE FROM libros WHERE id_libro = 5;
DELETE 1

biblioteca=# SELECT * FROM prestamos WHERE id_libro = 5;
 id_prestamo | id_libro | fecha_prestamo | fecha_devolucion | usuario_prestatario
-------------+----------+----------------+------------------+---------------------
(0 rows)
```

---

## 8. Creación de vistas

### 8.a. Crear una vista llamada `vista_libros_prestados` que muestre: título del libro, autor y nombre del prestatario

```
biblioteca=# CREATE VIEW vista_libros_prestados AS
    SELECT libros.titulo, autores.nombre AS autor, prestamos.
  usuario_prestatario
    FROM prestamos
    JOIN libros ON prestamos.id_libro = libros.id_libro
    JOIN autores ON libros.id_autor = autores.id_autor;
CREATE VIEW

biblioteca=# SELECT * FROM vista_libros_prestados;
        titulo        |         autor          | usuario_prestatario
----------------------+------------------------+---------------------
 1984                 | George Orwell          | Carlos Gómez
 1984                 | George Orwell          | Carlos Gómez
 Fundación            | Isaac Asimov           | David Ruiz
 Cien años de soledad | Gabriel García Márquez | Ana Pérez
(4 rows)
```

### 8.b. Conceder permisos de consulta sobre esta vista únicamente a `usuario_biblio`

```
biblioteca=# REVOKE ALL ON vista_libros_prestados FROM lectores;
REVOKE
biblioteca=# GRANT SELECT ON vista_libros_prestados TO usuario_biblio;
GRANT
```

---

## 9. Funciones y consultas avanzadas

### 9.a. Crear una función que reciba el nombre de un autor y devuelva todos los libros escritos por él

```
biblioteca=# CREATE OR REPLACE FUNCTION libros_por_autor(nombre_autor TEXT)
    RETURNS TABLE(titulo TEXT, año_publicacion INT) AS $$
        SELECT libros.titulo, libros.año_publicacion
        FROM libros
        JOIN autores ON libros.id_autor = autores.id_autor
        WHERE autores.nombre = nombre_autor;
    $$ LANGUAGE sql;
CREATE FUNCTION
biblioteca=# SELECT * FROM libros_por_autor('George Orwell');
        titulo         | año_publicacion
-----------------------+-----------------
 1984                  |            1949
 Rebelión en la granja |            1945
(2 rows)
```

### 9.b. Crear una consulta que devuelva los tres libros más prestados

```
biblioteca=# SELECT libros.titulo, COUNT(prestamos.id_prestamo) AS total_prestamos
    FROM libros
    JOIN prestamos ON libros.id_libro = prestamos.id_libro
    GROUP BY libros.id_libro, libros.titulo
    ORDER BY total_prestamos DESC
    LIMIT 3;
        titulo        | total_prestamos
----------------------+-----------------
 1984                 |               2
 Fundación            |               1
 Cien años de soledad |               1
(3 rows)
```

---

## 10. Exportación e importación de datos

### 10.a. Exportar el contenido de la tabla `libros` a un archivo CSV

```
biblioteca=# \copy libros TO 'libros_exportados.csv' WITH (FORMAT csv, HEADER)
COPY 7
```

### 10.b. Importar datos adicionales de autores desde un archivo CSV externo

```
biblioteca=# \copy autores(nombre, nacionalidad) FROM 'autores_adicionales.csv'
  WITH (FORMAT csv, HEADER)
COPY 2
biblioteca=# SELECT * FROM autores;
 id_autor |         nombre         | nacionalidad
----------+------------------------+--------------
        1 | Gabriel García Márquez | Colombiana
        2 | George Orwell          | Británica
        3 | Miguel de Cervantes    | Española
        4 | Jane Austen            | Británica
        5 | Isaac Asimov           | Rusa
        6 | Virginia Woolf         | Británica
        7 | Franz Kafka            | Checa
(7 rows)
```
