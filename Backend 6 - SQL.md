# SQL: la otra forma de guardar datos

Hoy nos tomamos un respiro del proyecto. Después de una semana cerrando un sprint completo, te mereces un día más tranquilo. Pero un día tranquilo en clase no es un día perdido: vamos a abrir una puerta importante.

Hasta ahora has guardado datos en dos sitios: en memoria (un array que se pierde al reiniciar) y en un fichero JSON. El lunes vas a aprender MongoDB, que es una **base de datos** de verdad. Pero antes de eso, te voy a enseñar el "otro mundo" de las bases de datos: el **modelo relacional** y su lenguaje, **SQL**.

¿Por qué en el proyecto vamos a usar Mongo y no SQL? Por tres razones:

1. **En cualquier entrevista técnica te van a preguntar por SQL.** Es el lenguaje de bases de datos más usado del mundo desde hace 50 años. No saberlo es un agujero en el CV.
2. **Para entender por qué existe Mongo, primero hay que ver qué hace SQL.** Mongo nació justamente como reacción a las limitaciones del modelo relacional. Entender el contraste te ayuda a entender ambos.
3. **El concepto de "lenguaje para hablar con la base de datos" lo vas a usar el lunes también.** Mongo tiene su propio lenguaje (los queries de Mongoose), pero la lógica de "le pides datos a la base, te los devuelve" es la misma.

Hoy no tocas el proyecto. SQL no entra en M2 como código. Esto es **cultura backend**: tienes que saber que existe, entender los conceptos básicos, saber leer una query sencilla.

---

## Warm-up: kata

```javascript
const posts = [
  { id: 1, titulo: "Hola mundo", autor: "Ana", likes: 23 },
  { id: 2, titulo: "Express mola", autor: "Luis", likes: 41 },
  { id: 3, titulo: "Async fácil", autor: "Ana", likes: 17 },
  { id: 4, titulo: "Refactor", autor: "Carmen", likes: 8 },
  { id: 5, titulo: "MVC explicado", autor: "Luis", likes: 35 },
];

// 1. Devuelve los títulos de los posts con más de 20 likes
// 2. Devuelve un array con los autores únicos
// 3. Devuelve un objeto: { Ana: 40, Luis: 76, Carmen: 8 }
//    (likes totales por autor)
```

> Apunta lo que ha costado el tercer ejercicio. Hoy lo verás resuelto en una línea de SQL.

---

## ¿Qué es una base de datos?

Empecemos por la palabra. Una **base de datos** es, conceptualmente, un sitio donde se guardan datos de forma organizada. Tu fichero `datos.json` es, en sentido amplio, una base de datos. Pero cuando alguien dice "base de datos" en el mundo profesional, normalmente se refiere a un **sistema de gestión de bases de datos** (SGBD o, en inglés, DBMS): un programa especializado que se dedica a guardar datos y servirlos eficientemente.

Algunos ejemplos que ya te sonarán o te van a sonar:

- **Relacionales (SQL):** PostgreSQL, MySQL, SQLite, SQL Server, Oracle.
- **De documentos (NoSQL):** MongoDB, CouchDB, Firestore (la que vimos en M1).
- **Clave-valor:** Redis, DynamoDB.
- **De grafos:** Neo4j.
- **Vectoriales (las "nuevas" para IA):** Pinecone, Weaviate.

Hoy vemos el primer grupo (relacionales) con SQLite, que es el más sencillo de probar porque no hace falta instalar nada: la base de datos entera vive en un solo fichero.

---

## Vamos a tocar SQL ya mismo

Para no instalar nada, vamos a usar **[sqliteonline.com](https://sqliteonline.com)**. Es un editor SQL en el navegador. Abre la web, asegúrate de que en la barra lateral izquierda está seleccionado **SQLite** (que es el dialecto que vamos a usar) y ya puedes empezar.

> No necesitas registro ni nada. Si quieres una alternativa offline, puedes instalar [DB Browser for SQLite](https://sqlitebrowser.org), que es una app de escritorio gratis. Pero hoy con la web sobra.

Borra cualquier cosa que haya en el editor y pega esto:

```sql
CREATE TABLE usuarios (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  nombre TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  edad INTEGER
);
```

Pulsa **Run** (el botón de play). Has creado tu primera tabla.

Vamos a desmenuzar esto:

- **`CREATE TABLE usuarios`**: crea una tabla llamada `usuarios`.
- **`id INTEGER PRIMARY KEY AUTOINCREMENT`**: una columna llamada `id`, de tipo número entero, que es la **clave primaria** (identificador único de cada fila) y se autoincrementa (1, 2, 3...) cada vez que insertas una fila.
- **`nombre TEXT NOT NULL`**: una columna `nombre` de tipo texto que **no puede estar vacía**.
- **`email TEXT NOT NULL UNIQUE`**: texto, no vacío, y **único** (no puede haber dos usuarios con el mismo email).
- **`edad INTEGER`**: número entero. Sin `NOT NULL`, así que puede estar vacío.

Una **tabla** es como una hoja de Excel: tiene columnas (los campos) con tipo fijo, y filas (los registros). Esto es lo más importante del modelo relacional: **el esquema es rígido**. Una vez creas la tabla, todas las filas tienen exactamente las mismas columnas.

### Insertar datos

```sql
INSERT INTO usuarios (nombre, email, edad) VALUES ('Ana', 'ana@example.com', 28);
INSERT INTO usuarios (nombre, email, edad) VALUES ('Luis', 'luis@example.com', 34);
INSERT INTO usuarios (nombre, email, edad) VALUES ('Carmen', 'carmen@example.com', 22);
INSERT INTO usuarios (nombre, email) VALUES ('Pedro', 'pedro@example.com');
```

Selecciónalas todas y dale a **Run**. Fíjate en que el último usuario no tiene edad: como la columna no es `NOT NULL`, se permite. No hace falta indicar el `id` porque se autogenera.

### Consultar datos: SELECT

La consulta más simple del universo:

```sql
SELECT * FROM usuarios;
```

`SELECT *` significa "trae todas las columnas". Vas a ver tus 4 usuarios.

Variantes que te van a sonar:

```sql
-- Solo algunas columnas
SELECT nombre, email FROM usuarios;

-- Filtrar con WHERE
SELECT * FROM usuarios WHERE edad > 25;

-- Filtrar por igualdad
SELECT * FROM usuarios WHERE nombre = 'Ana';

-- Ordenar
SELECT * FROM usuarios ORDER BY edad DESC;

-- Limitar resultados
SELECT * FROM usuarios LIMIT 2;

-- Combinar
SELECT nombre, edad FROM usuarios WHERE edad IS NOT NULL ORDER BY edad ASC LIMIT 3;
```

> Detalle: las cadenas en SQL van entre **comillas simples**, no dobles. `'Ana'`, no `"Ana"`. Las comillas dobles tienen otro uso (identificadores), así que no las uses para strings.

Pruébalas todas, cambia las condiciones, juega con `WHERE`.

### Actualizar: UPDATE

```sql
UPDATE usuarios SET edad = 35 WHERE nombre = 'Luis';
```

Y comprueba:

```sql
SELECT * FROM usuarios WHERE nombre = 'Luis';
```

⚠️ **Cuidado infinito con `UPDATE` sin `WHERE`:**

```sql
UPDATE usuarios SET edad = 99;
```

Esa query pone edad = 99 a **todos los usuarios**. No es una broma: es uno de los errores más caros que se pueden cometer en producción. Acostúmbrate a escribir el `WHERE` primero, y luego el `SET`. Mucha gente con experiencia escribe `SELECT * FROM ... WHERE ...` primero para confirmar qué filas va a tocar, y solo después cambia `SELECT` por `UPDATE`.

### Borrar: DELETE

```sql
DELETE FROM usuarios WHERE id = 4;
```

La misma advertencia: `DELETE FROM usuarios;` sin `WHERE` borra **todo**. Y como en SQL no hay papelera, no hay vuelta atrás.

---

## Tipos de datos en SQLite

SQLite es laxo con los tipos comparado con otros motores SQL, pero los más comunes que verás son:

| Tipo               | Para qué                                                  |
| ------------------ | --------------------------------------------------------- |
| `INTEGER`          | Números enteros                                           |
| `REAL`             | Números decimales                                         |
| `TEXT`             | Cadenas de texto                                          |
| `BLOB`             | Datos binarios (imágenes, etc.)                           |
| `BOOLEAN`          | En SQLite se guarda como `INTEGER` (0 o 1)                |
| `DATE`, `DATETIME` | Fechas (en SQLite se guardan como `TEXT` con formato ISO) |

En otros motores como PostgreSQL hay más tipos y son más estrictos (`VARCHAR(50)`, `TIMESTAMP WITH TIME ZONE`, etc.), pero el concepto es el mismo: **cada columna tiene un tipo, y no puedes meter texto donde dice número**.

Esto contrasta brutalmente con lo que haremos en Mongo, donde un documento puede tener una colección de campos distintos al siguiente. SQL es rígido, Mongo es flexible. Cada uno tiene su sitio.

---

## El concepto clave del modelo relacional: las relaciones

Hasta aquí podrías pensar que esto es como tu fichero JSON pero más feo. Lo que hace especial al modelo relacional son las **relaciones entre tablas**.

Imagina que quieres modelar un blog. Tienes:

- **Usuarios** (los que escriben)
- **Posts** (lo que publican)

Cada post pertenece a un usuario. ¿Cómo lo expresas? Con una **foreign key** (clave foránea): una columna en la tabla `posts` que apunta al `id` de un usuario.

```sql
CREATE TABLE posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  titulo TEXT NOT NULL,
  contenido TEXT,
  usuario_id INTEGER NOT NULL,
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);
```

La línea `FOREIGN KEY (usuario_id) REFERENCES usuarios(id)` le dice a la base de datos: "la columna `usuario_id` apunta al `id` de la tabla `usuarios`. No me dejes meter un `usuario_id` que no exista en `usuarios`".

Inserta posts:

```sql
INSERT INTO posts (titulo, contenido, usuario_id) VALUES ('Mi primer post', 'Hola blog', 1);
INSERT INTO posts (titulo, contenido, usuario_id) VALUES ('Otro post', 'Más contenido', 1);
INSERT INTO posts (titulo, contenido, usuario_id) VALUES ('Post de Luis', 'Hola yo soy Luis', 2);
INSERT INTO posts (titulo, contenido, usuario_id) VALUES ('Carmen escribe', 'Aquí estoy', 3);
```

Y ahora viene la magia: **cruzar las dos tablas con un JOIN**.

```sql
SELECT posts.titulo, usuarios.nombre AS autor
FROM posts
INNER JOIN usuarios ON posts.usuario_id = usuarios.id;
```

Resultado:

```
titulo               | autor
---------------------+--------
Mi primer post       | Ana
Otro post            | Ana
Post de Luis         | Luis
Carmen escribe       | Carmen
```

Esto es lo que la base de datos **te resuelve por ti**. No has tenido que recorrer dos arrays y emparejarlos a mano: SQL lo hace internamente y de forma muy eficiente.

El `INNER JOIN ... ON ...` se lee así: "une las filas de `posts` con las filas de `usuarios` donde `posts.usuario_id` coincida con `usuarios.id`".

> Hay otros tipos de JOIN (`LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN`) que tratan los casos donde no hay coincidencia. Hoy con `INNER JOIN` te sobra. Pero apúntate el nombre `LEFT JOIN`: te va a aparecer en entrevistas.

---

## Agregaciones: GROUP BY y COUNT

¿Recuerdas el tercer ejercicio de la kata? "Likes totales por autor". En JavaScript te ha costado un `reduce` con bastante cabeza. Mira en SQL:

```sql
-- Cuántos posts ha escrito cada usuario
SELECT usuarios.nombre, COUNT(posts.id) AS total_posts
FROM usuarios
LEFT JOIN posts ON posts.usuario_id = usuarios.id
GROUP BY usuarios.id;
```

Resultado:

```
nombre  | total_posts
--------+------------
Ana     | 2
Luis    | 1
Carmen  | 1
Pedro   | 0
```

`GROUP BY` agrupa las filas que comparten un valor, y luego puedes aplicar funciones de agregación: `COUNT` (contar), `SUM` (sumar), `AVG` (media), `MAX`, `MIN`...

Esto es lo que se llama el "superpoder" de SQL: hacer cálculos complejos sobre grandes volúmenes de datos directamente en la base de datos, sin tener que traerlos a tu aplicación. Si tuvieras 10 millones de posts, ese `COUNT` se ejecuta en milisegundos. Hacerlo en JavaScript implicaría traer los 10 millones a memoria, y reventarías el servidor.

---

## Ejercicio guiado: modelar un blog completo

Sin tocar tu proyecto, vamos a modelar un blog con tres tablas: usuarios, posts y comentarios. Esto es un ejercicio mental + práctica de queries. Sigue los pasos en sqliteonline.

```sql
-- Limpia lo anterior
DROP TABLE IF EXISTS posts;
DROP TABLE IF EXISTS usuarios;

-- Crea las tablas
CREATE TABLE usuarios (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  nombre TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE
);

CREATE TABLE posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  titulo TEXT NOT NULL,
  contenido TEXT,
  usuario_id INTEGER NOT NULL,
  fecha_creacion TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE TABLE comentarios (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  texto TEXT NOT NULL,
  post_id INTEGER NOT NULL,
  usuario_id INTEGER NOT NULL,
  FOREIGN KEY (post_id) REFERENCES posts(id),
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

-- Inserta datos
INSERT INTO usuarios (nombre, email) VALUES
  ('Ana', 'ana@blog.com'),
  ('Luis', 'luis@blog.com'),
  ('Carmen', 'carmen@blog.com');

INSERT INTO posts (titulo, contenido, usuario_id) VALUES
  ('Mi receta de tortilla', 'Patatas, huevos y cebolla', 1),
  ('Por qué Mongo mola', 'Es flexible y rápida', 2),
  ('Vacaciones en Galicia', 'Llovía pero comimos pulpo', 1);

INSERT INTO comentarios (texto, post_id, usuario_id) VALUES
  ('Sin cebolla, herejía', 2, 2),
  ('Con cebolla siempre', 1, 3),
  ('Yo prefiero SQL la verdad', 2, 3),
  ('A mí también me llovió', 3, 2);
```

Ahora prueba estas queries y entiéndelas (¡intenta predecir el resultado antes de ejecutarlas!):

```sql
-- 1. Todos los posts con el nombre de su autor
SELECT posts.titulo, usuarios.nombre AS autor
FROM posts
INNER JOIN usuarios ON posts.usuario_id = usuarios.id;

-- 2. Todos los comentarios de un post concreto, con el nombre del que comenta
SELECT comentarios.texto, usuarios.nombre AS autor_comentario
FROM comentarios
INNER JOIN usuarios ON comentarios.usuario_id = usuarios.id
WHERE comentarios.post_id = 2;

-- 3. Cuántos comentarios tiene cada post
SELECT posts.titulo, COUNT(comentarios.id) AS total_comentarios
FROM posts
LEFT JOIN comentarios ON comentarios.post_id = posts.id
GROUP BY posts.id;

-- 4. Posts que NO tienen ningún comentario
SELECT posts.titulo
FROM posts
LEFT JOIN comentarios ON comentarios.post_id = posts.id
WHERE comentarios.id IS NULL;
```

La cuarta es un truco clásico: con un `LEFT JOIN`, las filas de `posts` que no tengan comentarios aparecen con `NULL` en las columnas de `comentarios`. Filtrar por `IS NULL` da los posts huérfanos.

---

## SQL vs MongoDB: el contraste que verás el lunes

Para que llegues al lunes con la cabeza ordenada, este es el resumen de las dos formas de pensar los datos:

| Aspecto           | SQL (relacional)                                       | MongoDB (documentos)                                  |
| ----------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| **Unidad básica** | Fila en una tabla                                      | Documento (un JSON)                                   |
| **Estructura**    | Esquema fijo: todas las filas iguales                  | Flexible: cada documento puede tener campos distintos |
| **Relaciones**    | Foreign keys + JOINs                                   | Referencias o documentos anidados                     |
| **Lenguaje**      | SQL (estándar entre motores)                           | Queries en JSON (específicos de Mongo)                |
| **Fortaleza**     | Datos muy estructurados, queries complejas, integridad | Flexibilidad, rapidez de prototipo, JSON nativo       |
| **Cuándo elegir** | Banca, ERPs, sistemas con muchas relaciones            | Apps web modernas, datos heterogéneos, prototipos     |

El blog que has modelado con tres tablas, en Mongo se podría representar con **un solo documento por post** que tenga los comentarios anidados dentro:

```javascript
{
  _id: "abc123",
  titulo: "Mi receta de tortilla",
  contenido: "Patatas, huevos y cebolla",
  autor: { nombre: "Ana", email: "ana@blog.com" },
  comentarios: [
    { texto: "Sin cebolla, herejía", autor: "Luis" },
    { texto: "Con cebolla siempre", autor: "Carmen" }
  ],
  fechaCreacion: "2026-05-29T10:00:00Z"
}
```

¿Ves la diferencia? En SQL, los datos se separan en tablas y se reúnen con JOINs cuando los necesitas. En Mongo, los datos relacionados pueden vivir juntos en el mismo documento. Cada enfoque tiene ventajas e inconvenientes, y por eso ambos coexisten en el mundo profesional.

**Una pista importante para el lunes:** no intentes pensar Mongo como si fuera SQL. No es "SQL con sintaxis distinta". Es un paradigma diferente. Si llegas el lunes intentando hacer JOINs y tablas en Mongo, te vas a pelear con la herramienta. Llega con la mente abierta a la idea de documentos anidados.

---

## Algunas cosas más que conviene que sepas (sin profundizar)

Estas no las vas a usar hoy, pero te van a sonar en cualquier contexto SQL:

- **Índices (`CREATE INDEX`):** estructuras que aceleran las búsquedas en columnas concretas. Si vas a buscar mucho por email, creas un índice en `email`. Las primary keys ya tienen índice automáticamente.

- **Transacciones (`BEGIN`, `COMMIT`, `ROLLBACK`):** permiten ejecutar varias queries como una unidad atómica. Si una falla, todas se deshacen. Crítico en sistemas como bancos: la transferencia que resta dinero de una cuenta y lo suma a otra tiene que ocurrir entera o no ocurrir.

- **`NULL`:** la ausencia de valor. No es lo mismo que `0` o que cadena vacía. Para comparar usas `IS NULL` o `IS NOT NULL`, **nunca** `= NULL`.

- **Vistas (`CREATE VIEW`):** queries guardadas que se comportan como tablas virtuales. Útiles para queries complejas que se usan muchas veces.

- **ORMs:** librerías que te permiten trabajar con bases de datos SQL desde código JavaScript escribiendo objetos en vez de SQL a pelo. Las más conocidas: Prisma, Sequelize, TypeORM. Es lo que Mongoose hace para Mongo, salvando las distancias.

---

## Resolución de la kata del warm-up en SQL

Si tuvieras los posts en una tabla, las tres preguntas se contestan así:

```sql
-- Tabla supuesta:
-- CREATE TABLE posts (id, titulo, autor, likes);

-- 1. Títulos de posts con más de 20 likes
SELECT titulo FROM posts WHERE likes > 20;

-- 2. Autores únicos
SELECT DISTINCT autor FROM posts;

-- 3. Likes totales por autor
SELECT autor, SUM(likes) AS total_likes
FROM posts
GROUP BY autor;
```

Compara esto con lo que has escrito en JavaScript. La diferencia de **expresividad** es lo que ha mantenido a SQL vivo durante 50 años. Cuando trabajas con datos tabulares, SQL es brutal.

---

## Cierre de la semana

Has terminado la primera semana de M2:

- Lunes: Node, npm, primer setup.
- Martes: Express, rutas GET.
- Miércoles: middlewares, CORS, POST.
- Jueves: CRUD completo, persistencia a fichero, refactor a MVC.
- Hoy: SQL.

Y has cerrado un sprint con un entregable real. No está mal para cinco días.

---

## Lo que tienes que llevarte grabado

- Una **base de datos relacional** organiza la información en **tablas** con columnas tipadas y filas. Es **rígida**: el esquema se define al crear la tabla.
- **SQL** es el lenguaje estándar para hablar con bases de datos relacionales. Las cinco operaciones que más vas a ver: `CREATE TABLE`, `INSERT`, `SELECT`, `UPDATE`, `DELETE`.
- **Primary key** identifica una fila de forma única. **Foreign key** referencia la primary key de otra tabla. Así se modelan las relaciones.
- Un **JOIN** une filas de dos tablas según una condición. `INNER JOIN` trae solo las coincidencias.
- `GROUP BY` + funciones de agregación (`COUNT`, `SUM`, `AVG`) son el superpoder de SQL para resumir datos.
- **`UPDATE` y `DELETE` sin `WHERE` borran todo.** Costumbre obligatoria: comprobar primero con `SELECT`.
- **SQL ≠ Mongo.** No son la misma cosa con sintaxis distinta: son paradigmas diferentes. El lunes llegas con la cabeza abierta a un modelo nuevo: documentos, no tablas.
