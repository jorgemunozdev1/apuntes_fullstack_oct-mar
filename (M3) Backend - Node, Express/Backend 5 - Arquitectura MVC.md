# Arquitectura MVC: ordenar el caos

Tu `index.js` ya pasa de las 100 líneas. Tienes el logger, el CORS, el `express.json()`, las funciones de cargar y guardar, y los cinco handlers del CRUD, todo amontonado en un mismo archivo. Funciona, pero abrirlo empieza a dar pereza: para cambiar una cosa pequeña tienes que rebuscar entre todo lo demás.

Hoy le ponemos orden. Vamos a separar el código en capas siguiendo el patrón **MVC** (Modelo–Vista–Controlador), que es la forma estándar de organizar una aplicación backend. Al final del día tu API hará exactamente lo mismo que ahora —ni una funcionalidad nueva— pero estará repartida en archivos con responsabilidades claras. Eso es un **refactor**: cambiar la estructura del código sin cambiar lo que hace.

Con esto cerramos el sprint 1: una API CRUD, con persistencia real, y ahora también con una arquitectura limpia.

---

## Objetivos del día

- Entender qué es el patrón MVC y qué responsabilidad tiene cada capa.
- Separar tu `index.js` monolítico en: punto de entrada, rutas, controladores y modelo.
- Entender qué es un router de Express y cómo se monta.
- Dejar la capa de datos (el modelo) aislada del resto.
- Tener tu proyecto funcionando igual que ayer, pero ordenado.

---

## Warm-up: kata

```javascript
// Tienes una función que hace demasiadas cosas a la vez:

function procesarUsuario(datos) {
  // valida
  if (!datos.email.includes("@")) return "Email inválido";
  // transforma
  const usuario = {
    nombre: datos.nombre.trim().toLowerCase(),
    email: datos.email.trim().toLowerCase(),
  };
  // "guarda"
  console.log("Guardando en BD:", usuario);
  return usuario;
}

// Pregunta para pensar (no hay que programar nada):
// Esta función valida, transforma Y guarda. ¿Por qué puede ser
// un problema que una sola función haga tres cosas distintas?
// ¿Cómo lo separarías?
```

Piénsalo un par de minutos. La respuesta tiene que ver con todo lo que vamos a hacer hoy: **cada pieza de código debería tener una sola responsabilidad.**

---

## ¿Qué es MVC?

MVC reparte la aplicación en tres responsabilidades:

- **Modelo (Model):** los datos y cómo se accede a ellos. Es la capa que sabe leer y escribir. Hoy lee y escribe un fichero JSON; cuando migremos hará lo mismo contra MongoDB. El resto de la app no debería enterarse de ese cambio.

- **Vista (View):** lo que ve el usuario. En una API REST la "vista" es el JSON que devolvemos. Como nosotros devolvemos JSON directamente, no tenemos una capa de vistas separada (eso aplica a apps que renderizan HTML en el servidor). En nuestro caso, la vista es la respuesta JSON.

- **Controlador (Controller):** la lógica que conecta lo anterior. Recibe la petición, pide datos al modelo, decide qué responder y con qué status code. Es el "cerebro" que coordina.

A esto le sumamos una pieza que técnicamente no es parte del MVC clásico pero es clave en Express:

- **Rutas (Routes):** el mapa que dice "cuando llegue `GET /api/notas`, ejecuta tal controlador". Separa el _qué URL_ del _qué se hace_.

### El flujo completo

```
Petición HTTP
     │
     ▼
  [ Rutas ]         → "esta URL la maneja este controlador"
     │
     ▼
[ Controlador ]     → recibe req, decide, responde
     │
     ▼
  [ Modelo ]        → lee/escribe los datos
     │
     ▼
  Respuesta JSON (la "vista")
```

### Por qué molestarse

Ahora mismo todo en un archivo "funciona". Las ventajas de separarlo se notan cuando:

- **El proyecto crece.** Con 3 entidades (notas, usuarios, etiquetas) y 15 rutas, un solo archivo es ingobernable.
- **Cambias una capa sin tocar las demás.** Cuando cambias el modelo de fichero a MongoDB. Si el modelo está aislado, solo tocas ese archivo: los controladores y las rutas ni se enteran.
- **Encuentras las cosas.** ¿Un bug al crear notas? Vas al controlador de notas. ¿Problema al guardar? Vas al modelo. Cada cosa en su sitio.

---

## La estructura de carpetas

Vamos a pasar de esto:

```
mi-proyecto/
├── index.js          ← TODO está aquí
├── datos.json
├── package.json
└── .gitignore
```

A esto:

```
mi-proyecto/
├── index.js              ← solo arranca el servidor
├── src/
│   ├── app.js            ← configura Express y middlewares
│   ├── routes/
│   │   └── notas.routes.js
│   ├── controllers/
│   │   └── notas.controller.js
│   └── models/
│       └── notas.model.js
├── datos.json
├── package.json
└── .gitignore
```

Cada archivo tiene un único trabajo. Vamos a construirlo paso a paso, moviendo el código que ya tienes.

---

## Paso 1: el modelo (`src/models/notas.model.js`)

El modelo es todo lo que toca los datos. Aquí van las funciones `cargarNotas` y `guardarNotas` que ya tienes, más unas cuantas funciones nuevas que encapsulan las operaciones del CRUD. La idea es que el controlador **nunca** lea o escriba el fichero directamente: siempre se lo pide al modelo.

```javascript
const fs = require("fs").promises;
const path = require("path");

const RUTA_DATOS = path.join(__dirname, "..", "..", "datos.json");

// ─── Funciones internas (no se exportan) ───

async function leerFichero() {
  try {
    const contenido = await fs.readFile(RUTA_DATOS, "utf-8");
    return JSON.parse(contenido);
  } catch (error) {
    if (error.code === "ENOENT") {
      await fs.writeFile(RUTA_DATOS, "[]", "utf-8");
      return [];
    }
    throw error;
  }
}

async function escribirFichero(notas) {
  await fs.writeFile(RUTA_DATOS, JSON.stringify(notas, null, 2), "utf-8");
}

// ─── Funciones públicas (la "API" del modelo) ───

async function obtenerTodas() {
  return await leerFichero();
}

async function obtenerPorId(id) {
  const notas = await leerFichero();
  return notas.find((n) => n.id === id) || null;
}

async function crear(datos) {
  const notas = await leerFichero();
  const nuevoId =
    notas.length > 0 ? Math.max(...notas.map((n) => n.id)) + 1 : 1;
  const nueva = { id: nuevoId, ...datos };
  notas.push(nueva);
  await escribirFichero(notas);
  return nueva;
}

async function actualizar(id, datos) {
  const notas = await leerFichero();
  const indice = notas.findIndex((n) => n.id === id);
  if (indice === -1) return null;
  notas[indice] = { id, ...datos };
  await escribirFichero(notas);
  return notas[indice];
}

async function borrar(id) {
  const notas = await leerFichero();
  const indice = notas.findIndex((n) => n.id === id);
  if (indice === -1) return null;
  const [borrada] = notas.splice(indice, 1);
  await escribirFichero(notas);
  return borrada;
}

module.exports = {
  obtenerTodas,
  obtenerPorId,
  crear,
  actualizar,
  borrar,
};
```

Fíjate en dos cosas:

**La ruta del fichero cambió.** Ahora es `path.join(__dirname, "..", "..", "datos.json")`. Como el modelo está dentro de `src/models/`, hay que subir dos niveles (`../../`) para llegar a la raíz donde vive `datos.json`. `__dirname` siempre apunta a la carpeta del archivo actual, así que desde `src/models/` subimos a `src/` y luego a la raíz.

**Las funciones públicas tienen nombres de negocio, no de fichero.** `obtenerTodas`, `crear`, `borrar`… no dicen "lee el JSON" ni "escribe el fichero". Dicen _qué hacen_ en términos del dominio. Esto es lo que va a permitir que cuando cambiemos toda la implementación interna (a MongoDB) sin tocar ni una línea de los controladores: ellos siguen llamando a `crear` y `obtenerTodas`, y les da igual si por debajo es un fichero o una base de datos.

---

## Paso 2: el controlador (`src/controllers/notas.controller.js`)

El controlador recibe la petición, llama al modelo y decide qué responder. **No sabe nada de cómo se guardan los datos** — solo le pide cosas al modelo.

```javascript
const Notas = require("../models/notas.model");

async function listarNotas(req, res) {
  try {
    const notas = await Notas.obtenerTodas();
    res.json(notas);
  } catch (error) {
    console.error("Error al listar notas:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
}

async function obtenerNota(req, res) {
  try {
    const id = Number(req.params.id);
    const nota = await Notas.obtenerPorId(id);
    if (!nota) {
      return res.status(404).json({ error: "Nota no encontrada" });
    }
    res.json(nota);
  } catch (error) {
    console.error("Error al obtener nota:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
}

async function crearNota(req, res) {
  try {
    const { titulo, contenido } = req.body;
    if (!titulo || !contenido) {
      return res.status(400).json({ error: "Falta título o contenido" });
    }
    const nueva = await Notas.crear({ titulo, contenido });
    res.status(201).json(nueva);
  } catch (error) {
    console.error("Error al crear nota:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
}

async function actualizarNota(req, res) {
  try {
    const id = Number(req.params.id);
    const { titulo, contenido } = req.body;
    if (!titulo || !contenido) {
      return res.status(400).json({ error: "Falta título o contenido" });
    }
    const actualizada = await Notas.actualizar(id, { titulo, contenido });
    if (!actualizada) {
      return res.status(404).json({ error: "Nota no encontrada" });
    }
    res.json(actualizada);
  } catch (error) {
    console.error("Error al actualizar nota:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
}

async function borrarNota(req, res) {
  try {
    const id = Number(req.params.id);
    const borrada = await Notas.borrar(id);
    if (!borrada) {
      return res.status(404).json({ error: "Nota no encontrada" });
    }
    res.json({ mensaje: "Nota borrada", nota: borrada });
  } catch (error) {
    console.error("Error al borrar nota:", error);
    res.status(500).json({ error: "Error interno del servidor" });
  }
}

module.exports = {
  listarNotas,
  obtenerNota,
  crearNota,
  actualizarNota,
  borrarNota,
};
```

Date cuenta de cómo ha quedado cada controlador: validar la petición, pedir algo al modelo, decidir el status code, responder. Esa es toda su responsabilidad. La conversión de `id` a número, las comprobaciones de "existe / no existe" y los status codes viven aquí, en el controlador, porque son **decisiones sobre la petición HTTP**. El modelo no sabe qué es un 404 — eso es lenguaje HTTP, y el HTTP es cosa del controlador.

---

## Paso 3: las rutas (`src/routes/notas.routes.js`)

Un **router** es como un mini-Express que agrupa rutas relacionadas. En vez de colgar todas las rutas de `app`, las colgamos de un router y luego "montamos" ese router en la app.

```javascript
const express = require("express");
const router = express.Router();
const controlador = require("../controllers/notas.controller");

router.get("/", controlador.listarNotas);
router.get("/:id", controlador.obtenerNota);
router.post("/", controlador.crearNota);
router.put("/:id", controlador.actualizarNota);
router.delete("/:id", controlador.borrarNota);

module.exports = router;
```

Mira qué limpio queda: este archivo es solo el **mapa**. De un vistazo ves todas las rutas de notas y qué controlador maneja cada una. Nada de lógica aquí.

Fíjate también en que las rutas son `/` y `/:id`, **no** `/api/notas` y `/api/notas/:id`. Eso es porque el prefijo `/api/notas` se lo vamos a poner al montar el router, en el siguiente paso. El router solo conoce las rutas _relativas a sí mismo_.

---

## Paso 4: la configuración de Express (`src/app.js`)

Aquí va toda la configuración de Express: los middlewares globales y el montaje de los routers.

```javascript
const express = require("express");
const cors = require("cors");
const notasRoutes = require("./routes/notas.routes");

const app = express();

// ─── Middlewares globales ───
app.use(cors());
app.use(express.json());

// Logger
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// ─── Rutas ───
app.get("/api/health", (req, res) => {
  res.json({ status: "ok" });
});

app.use("/api/notas", notasRoutes);

module.exports = app;
```

La línea clave es esta:

```javascript
app.use("/api/notas", notasRoutes);
```

Esto **monta** el router en la ruta `/api/notas`. Significa: "todas las rutas que defina `notasRoutes` van precedidas de `/api/notas`". Así, la ruta `/` del router se convierte en `/api/notas`, y la ruta `/:id` se convierte en `/api/notas/:id`. Por eso en el router las definimos sin el prefijo.

Cuando tengas más entidades (por ejemplo usuarios), añadirás otra línea: `app.use("/api/usuarios", usuariosRoutes)`. Cada recurso, su router.

---

## Paso 5: el punto de entrada (`index.js`)

El `index.js` que tenías lleno de código ahora queda reducido a lo mínimo: importar la app configurada y ponerla a escuchar.

```javascript
require("dotenv").config();
const app = require("./src/app");

const PUERTO = process.env.PUERTO || 3000;

app.listen(PUERTO, () => {
  console.log(`Servidor escuchando en http://localhost:${PUERTO}`);
});
```

Eso es todo lo que queda en la raíz. Tres responsabilidades: cargar las variables de entorno, traer la app, y arrancar el servidor. Si mañana cambias el puerto o añades algo al arranque, sabes que es aquí.

> Separamos `index.js` (arranca el servidor) de `app.js` (configura Express) por un motivo que entenderás de lleno en la clase de testing: para los tests necesitamos la `app` **sin** que esté escuchando en un puerto. Tener la configuración separada del `listen` nos lo permite. Por ahora basta con saber que es una buena práctica estándar.

---

## El resultado

Tu API hace **exactamente lo mismo** que ayer. Mismos endpoints, mismas respuestas, misma persistencia. Pero ahora:

- `index.js` arranca el servidor y nada más.
- `app.js` configura Express.
- Las rutas dicen qué URL va a qué controlador.
- Los controladores deciden qué responder.
- El modelo es el único que toca los datos.

Cuando cambiemos el modelo de fichero JSON a MongoDB, **solo tocaremos `notas.model.js`**. Los controladores seguirán llamando a `Notas.crear(...)`, `Notas.obtenerTodas(...)` igual que ahora, sin enterarse de que por debajo ahora hay una base de datos de verdad. Eso es exactamente para lo que sirve separar en capas.

---

## Comprobación: ¿sigue funcionando todo?

Antes de dar el refactor por bueno, prueba las cinco operaciones desde tu cliente (Postman, Bruno, ThunderClient) y confirma que se comportan igual que ayer:

```http
### Listar
GET http://localhost:3000/api/notas

### Health check
GET http://localhost:3000/api/health

### Crear
POST http://localhost:3000/api/notas
Content-Type: application/json

{ "titulo": "Refactor MVC", "contenido": "Todo más ordenado" }

### Obtener una
GET http://localhost:3000/api/notas/1

### Actualizar
PUT http://localhost:3000/api/notas/1
Content-Type: application/json

{ "titulo": "Editada", "contenido": "Contenido nuevo" }

### Borrar
DELETE http://localhost:3000/api/notas/2
```

Si todas responden como antes, el refactor es correcto. **Esa es la regla de oro de un refactor: el comportamiento no cambia, solo la organización.**

---

## Aplicación al proyecto

Lleva esta misma estructura a tu API personal:

1. **Crea la estructura de carpetas** `src/` con `models/`, `controllers/`, `routes/`.
2. **Mueve tu lógica de datos al modelo**, con funciones de nombre de negocio (`obtenerTodas`, `crear`, etc.) que no revelen que por debajo es un fichero.
3. **Mueve tus handlers a un controlador**, dejándolos centrados en validar, pedir al modelo y responder.
4. **Crea el router** de tu entidad con las rutas relativas.
5. **Configura `app.js`** con los middlewares y el montaje del router.
6. **Reduce `index.js`** a arrancar el servidor.
7. **Prueba las cinco operaciones** y confirma que todo sigue igual que ayer.

Cuando lo tengas funcionando idéntico a antes:

```bash
git add .
git commit -m "refactor: reestructurar proyecto a arquitectura MVC"
git push
```

Y mueve a `Done` las tarjetas correspondientes. **Con esto cierras el sprint 1.**

---

## Cierre de sprint

Has terminado la primera semana de M2 con un entregable real: una API REST con CRUD completo, persistencia que sobrevive a reinicios, y una arquitectura MVC limpia y escalable. No es un juguete que se resetea: es una base sólida sobre la que vas a construir todo lo demás.

Mírate un momento el tablero Kanban. La columna `Done` debería tener ya unas cuantas tarjetas. Eso es lo que significa iterar: cada día has añadido una capa, y nada de lo que has hecho se ha tirado.

Para cerrar, una pequeña reflexión que te conviene anotar (te servirá para la presentación final del proyecto):

- ¿Qué parte de la semana te ha costado más entender?
- ¿Qué harías distinto si empezaras el proyecto otra vez?
- ¿Hay algo de tu código que sabes que está "apañado" y te gustaría mejorar?

---

## Lo que tienes que llevarte grabado

- **MVC** reparte responsabilidades: modelo (datos), controlador (lógica de la petición), vista (en una API, el JSON). Las rutas son el mapa URL → controlador.
- Un **router** de Express agrupa rutas relacionadas y se monta con `app.use("/prefijo", router)`. Las rutas dentro del router son relativas al prefijo.
- El **modelo aislado** es lo que permite cambiar de fichero a base de datos sin tocar nada más. Funciones con nombre de negocio, no de fichero.
- El **controlador** habla HTTP (status codes, validación de la petición); el **modelo** no sabe nada de HTTP.
- Separar `index.js` (arranca) de `app.js` (configura) es buena práctica y nos hará falta para los tests.
- Un **refactor** cambia la estructura, no el comportamiento. Si después de refactorizar la API responde distinto, algo se ha roto.
