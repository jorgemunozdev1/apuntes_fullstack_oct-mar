# CRUD completo + persistencia con `fs`

## Warm-up: kata

```javascript
const tareas = [
  { id: 1, titulo: "Comprar pan", hecha: true },
  { id: 2, titulo: "Llamar al médico", hecha: false },
  { id: 3, titulo: "Estudiar Express", hecha: true },
  { id: 4, titulo: "Pasear al perro", hecha: false },
];

// 1. Devuelve un NUEVO array igual pero con la tarea de id=2 marcada como hecha
//    (sin mutar el original)

// 2. Devuelve un nuevo array sin la tarea de id=4
//    (sin mutar el original)

// 3. Devuelve un objeto: { hechas: 2, pendientes: 2 }
```

> Los ejercicios 1 y 2 son **exactamente** lo que vais a hacer hoy en los handlers de `PUT` y `DELETE`. La kata es una pre-vista.

---

## Recordatorio: dónde estamos

Vuestro `index.js` ahora mismo tiene aproximadamente:

```javascript
const express = require("express");
const cors = require("cors");
const app = express();

app.use(cors());
app.use(express.json());

// Logger
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

const notas = [{ id: 1, titulo: "Primera nota", contenido: "Hola" }];

app.get("/api/notas", (req, res) => {
  res.json(notas);
});

app.get("/api/notas/:id", (req, res) => {
  const id = Number(req.params.id);
  const nota = notas.find((n) => n.id === id);
  if (!nota) return res.status(404).json({ error: "No encontrada" });
  res.json(nota);
});

app.post("/api/notas", (req, res) => {
  const { titulo, contenido } = req.body;
  if (!titulo || !contenido) {
    return res.status(400).json({ error: "Falta título o contenido" });
  }
  const nueva = { id: notas.length + 1, titulo, contenido };
  notas.push(nueva);
  res.status(201).json(nueva);
});

app.listen(3000);
```

Hoy le añadimos `PUT` y `DELETE`, y luego cambiamos el array en memoria por persistencia a fichero.

---

## PUT: actualizar un recurso

`PUT` se usa para **reemplazar** un recurso completo. La ruta sigue el patrón REST:

```
PUT /api/notas/:id
```

Implementación:

```javascript
app.put("/api/notas/:id", (req, res) => {
  const id = Number(req.params.id);
  const indice = notas.findIndex((n) => n.id === id);

  if (indice === -1) {
    return res.status(404).json({ error: "Nota no encontrada" });
  }

  const { titulo, contenido } = req.body;
  if (!titulo || !contenido) {
    return res.status(400).json({ error: "Falta título o contenido" });
  }

  const notaActualizada = { id, titulo, contenido };
  notas[indice] = notaActualizada;

  res.json(notaActualizada);
});
```

Cosas importantes:

- Buscamos con `findIndex` en vez de `find` porque necesitamos la **posición** para reemplazar.
- El `id` lo mantenemos del param, no del body. Aunque alguien envíe un id distinto en el body, lo ignoramos.
- Respondemos con el objeto actualizado (algunas APIs devuelven 204 sin body, otras devuelven el objeto; haremos lo segundo).

Probadlo desde `peticiones.http`:

```http
### Actualizar nota
PUT http://localhost:3000/api/notas/1
Content-Type: application/json

{
  "titulo": "Título nuevo",
  "contenido": "Contenido actualizado"
}
```

### PUT vs PATCH

- `PUT` reemplaza el recurso entero. Si no envías un campo, se pierde.
- `PATCH` actualiza solo los campos que envías.

En este módulo usaremos `PUT`. `PATCH` lo veis en el día a día pero la diferencia conceptual es la que acabáis de leer.

---

## DELETE: borrar un recurso

```javascript
app.delete("/api/notas/:id", (req, res) => {
  const id = Number(req.params.id);
  const indice = notas.findIndex((n) => n.id === id);

  if (indice === -1) {
    return res.status(404).json({ error: "Nota no encontrada" });
  }

  const [notaBorrada] = notas.splice(indice, 1);

  res.json({ mensaje: "Nota borrada", nota: notaBorrada });
});
```

`splice(indice, 1)` elimina **un** elemento del array a partir de la posición indicada y devuelve un array con los borrados. Por eso usamos destructuring `const [notaBorrada]` para sacar el primero.

Probadlo:

```http
### Borrar nota
DELETE http://localhost:3000/api/notas/1
```

### ¿200 con body o 204 sin body?

Hay dos convenciones:

```javascript
// Opción A: devolver el recurso borrado
res.json({ mensaje: "Nota borrada", nota: notaBorrada });

// Opción B: 204 No Content, sin body
res.status(204).end();
```

Ambas son correctas. Iremos con la **A** porque es más útil para depurar y porque el frontend a veces necesita confirmar qué se borró.

---

## El problema de la memoria

Tenéis CRUD completo. Genial. Pero probad esto:

1. Hacéis `POST` con 3 notas nuevas.
2. Hacéis `GET` → veis las 3.
3. Paráis el servidor (`Ctrl+C`) y lo arrancáis de nuevo.
4. `GET` → solo veis la nota inicial. **Las 3 se han perdido.**

Esto pasa porque vuestros datos viven en una variable de JavaScript: un array. Cuando el proceso de Node muere, la memoria desaparece. Todo lo que estuviese en variables se va con ella.

**La solución de verdad** es una base de datos (MongoDB el lunes). Pero hoy vamos a un paso intermedio que ya os enseña una herramienta importante: **leer y escribir ficheros desde Node**.

---

## El módulo `fs`

`fs` (filesystem) es un módulo nativo de Node — no se instala, viene con Node. Sirve para leer, escribir, borrar y mover ficheros.

Hay dos versiones de cada función: **síncrona** y **asíncrona**.

### Síncrona (lo que ya conocéis)

```javascript
const fs = require("fs");

const contenido = fs.readFileSync("datos.json", "utf-8");
console.log(contenido);
```

`readFileSync` **bloquea** el proceso hasta que termina de leer. Mientras está leyendo, Node no puede hacer nada más.

### Asíncrona con Promesas (la que vamos a usar)

```javascript
const fs = require("fs").promises;

const contenido = await fs.readFile("datos.json", "utf-8");
console.log(contenido);
```

`readFile` (sin el `Sync`) devuelve una Promesa. Mientras está leyendo, Node sigue atendiendo otras peticiones.

> **Recordáis de M1:** `await` solo dentro de funciones `async`. Y la regla de la contaminación: `async` es contagioso **solo hasta el `await`**. Después de que resuelva, volvéis a código síncrono normal.

### ¿Por qué la asíncrona en un servidor?

Imaginad que vuestra API recibe 100 peticiones a la vez. Si usáis `readFileSync`, cada petición **bloquea** el servidor durante el rato que tarda en leer el fichero. Las otras 99 peticiones esperan.

Con `readFile` asíncrono, mientras una petición espera al disco, Node atiende las otras. **En un servidor siempre, siempre usad las versiones asíncronas.**

`readFileSync` se usa para scripts puntuales o configuración al arranque. Hoy mismo, mañana o en el deploy, NUNCA dentro de un handler.

---

## Persistir las notas en un fichero JSON

Vamos a guardar el array `notas` en un fichero llamado `datos.json`. Cada vez que se modifique, lo reescribimos. Cada vez que se lea, lo leemos.

### Paso 1: crear el fichero inicial

Manualmente, crear en la raíz del proyecto `datos.json`:

```json
[{ "id": 1, "titulo": "Primera nota", "contenido": "Hola desde el fichero" }]
```

### Paso 2: funciones de lectura y escritura

Encima de las rutas, antes de los `app.get/post/...`:

```javascript
const fs = require("fs").promises;
const path = require("path");

const RUTA_DATOS = path.join(__dirname, "datos.json");

async function cargarNotas() {
  const contenido = await fs.readFile(RUTA_DATOS, "utf-8");
  return JSON.parse(contenido);
}

async function guardarNotas(notas) {
  await fs.writeFile(RUTA_DATOS, JSON.stringify(notas, null, 2), "utf-8");
}
```

Detalles que merecen explicación:

- **`path.join(__dirname, "datos.json")`** construye la ruta absoluta al fichero. `__dirname` es una variable mágica de Node que apunta a la carpeta del fichero actual. Si usáis solo `"datos.json"`, funciona cuando arrancáis con `npm run dev` desde la raíz pero falla si lo arrancáis desde otra carpeta. **Siempre rutas absolutas con `path.join`** en servidores.

- **`JSON.stringify(notas, null, 2)`** convierte el array a string JSON. El `2` es para que el fichero quede indentado con 2 espacios, legible para humanos. Sin él queda todo en una línea.

- **`JSON.parse(contenido)`** convierte el string del fichero al array de objetos.

### Paso 3: usar las funciones en las rutas

Quitad la línea `const notas = [...]` que teníais en memoria. Las rutas pasan a ser:

```javascript
// GET todas
app.get("/api/notas", async (req, res) => {
  const notas = await cargarNotas();
  res.json(notas);
});

// GET una
app.get("/api/notas/:id", async (req, res) => {
  const notas = await cargarNotas();
  const id = Number(req.params.id);
  const nota = notas.find((n) => n.id === id);

  if (!nota) return res.status(404).json({ error: "Nota no encontrada" });
  res.json(nota);
});

// POST crear
app.post("/api/notas", async (req, res) => {
  const { titulo, contenido } = req.body;
  if (!titulo || !contenido) {
    return res.status(400).json({ error: "Falta título o contenido" });
  }

  const notas = await cargarNotas();
  const nuevoId =
    notas.length > 0 ? Math.max(...notas.map((n) => n.id)) + 1 : 1;
  const nueva = { id: nuevoId, titulo, contenido };

  notas.push(nueva);
  await guardarNotas(notas);

  res.status(201).json(nueva);
});

// PUT actualizar
app.put("/api/notas/:id", async (req, res) => {
  const id = Number(req.params.id);
  const { titulo, contenido } = req.body;

  if (!titulo || !contenido) {
    return res.status(400).json({ error: "Falta título o contenido" });
  }

  const notas = await cargarNotas();
  const indice = notas.findIndex((n) => n.id === id);

  if (indice === -1)
    return res.status(404).json({ error: "Nota no encontrada" });

  notas[indice] = { id, titulo, contenido };
  await guardarNotas(notas);

  res.json(notas[indice]);
});

// DELETE
app.delete("/api/notas/:id", async (req, res) => {
  const id = Number(req.params.id);
  const notas = await cargarNotas();
  const indice = notas.findIndex((n) => n.id === id);

  if (indice === -1)
    return res.status(404).json({ error: "Nota no encontrada" });

  const [borrada] = notas.splice(indice, 1);
  await guardarNotas(notas);

  res.json({ mensaje: "Nota borrada", nota: borrada });
});
```

**Cambios clave** respecto a antes:

- Todos los handlers son `async`.
- Empezamos cargando las notas del fichero con `await cargarNotas()`.
- Después de cualquier modificación, `await guardarNotas(notas)`.
- El nuevo ID ya no es `notas.length + 1` (que falla si borráis y creáis), sino `Math.max(...notas.map(n => n.id)) + 1`. Si borráis la nota con id=3 y luego creáis una nueva, no se reutiliza el id=3.

### Paso 4: probadlo

Arrancad el servidor, haced varios `POST`, `PUT`, `DELETE`. Mirad cómo cambia `datos.json` en VS Code en tiempo real. **Ese es el momento mágico**: vuestros datos sobreviven al reinicio.

---

## Manejo de errores: ¿qué pasa si falla `fs`?

Probad esto: borrad `datos.json` y haced una petición `GET /api/notas`. ¿Qué pasa?

El servidor **se cuelga o devuelve un error feo**. Porque el `await fs.readFile(...)` lanza una excepción y nadie la captura.

Solución provisional: `try/catch`:

```javascript
app.get("/api/notas", async (req, res) => {
  try {
    const notas = await cargarNotas();
    res.json(notas);
  } catch (error) {
    console.error("Error cargando notas:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
});
```

Esto es repetitivo (un `try/catch` en cada handler). Más adelante veremos cómo centralizar el manejo de errores con un middleware. Por hoy, ponedlo para que la API no se caiga.

### Bonus: que `cargarNotas` cree el fichero si no existe

```javascript
async function cargarNotas() {
  try {
    const contenido = await fs.readFile(RUTA_DATOS, "utf-8");
    return JSON.parse(contenido);
  } catch (error) {
    if (error.code === "ENOENT") {
      // El fichero no existe, lo creamos vacío
      await fs.writeFile(RUTA_DATOS, "[]", "utf-8");
      return [];
    }
    throw error; // otros errores se siguen propagando
  }
}
```

`ENOENT` = "Error No ENTry". Es el código que Node devuelve cuando un fichero no existe. Esto lo veis muy a menudo en backend, vale la pena memorizarlo.

---

Al terminar:

```bash
git add .
git commit -m "feat: CRUD completo con persistencia a fichero"
git push
```

Y mover las tarjetas del Kanban a `Done`. Hoy se cierran varias.

---
