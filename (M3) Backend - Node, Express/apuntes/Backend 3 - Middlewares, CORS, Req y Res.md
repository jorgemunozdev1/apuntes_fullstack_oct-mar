# Middlewares, CORS y el objeto Request

Hoy desbloqueamos el concepto más importante de Express: los **middlewares**. Una vez los entendéis, todo lo que viene después (validación, autenticación, parsing de JSON, CORS, manejo de errores) deja de ser magia y pasa a ser "ah, claro, otro middleware".

También conocemos en profundidad el objeto `req` — todo lo que llega del cliente vive ahí.

---

## Objetivos del día

- Entender qué es un middleware y la cadena `req → middleware → middleware → handler → res`.
- Saber escribir vuestros propios middlewares.
- Conocer y usar los middlewares nativos de Express (`express.json`).
- Entender qué es CORS, por qué existe y cómo configurarlo.
- Acceder a `body`, `params`, `query` y `headers` del objeto `req`.
- Aceptar peticiones `POST` con body JSON en vuestra API.

---

## Warm-up: kata

```javascript
const usuarios = [
  { id: 1, nombre: "Ana", rol: "admin", activo: true },
  { id: 2, nombre: "Luis", rol: "user", activo: false },
  { id: 3, nombre: "Carmen", rol: "user", activo: true },
  { id: 4, nombre: "Pedro", rol: "admin", activo: true },
];

// 1. Devuelve un array con SOLO los nombres de los usuarios activos
// 2. Devuelve un objeto con la cuenta de usuarios por rol:
//    { admin: 2, user: 2 }
// 3. Devuelve true si TODOS los admins están activos
```

---

## ¿Qué es un middleware?

Hasta ahora habéis visto esto:

```javascript
app.get("/api/notas", (req, res) => {
  res.json(notas);
});
```

Una petición entra, un handler responde. Fin.

Pero **¿y si necesitamos hacer algo antes de llegar al handler?** Por ejemplo:

- Loggear cada petición que entra
- Comprobar que el usuario está autenticado
- Parsear el body de la petición a JSON
- Validar que los datos enviados son correctos
- Permitir que el frontend pueda llamar a la API (CORS)

Todo eso son **middlewares**: funciones que se ejecutan entre que llega la petición y se envía la respuesta.

### La cadena de middlewares

```
Petición → MW1 → MW2 → MW3 → Handler → Respuesta
```

Cada middleware decide:

- **Hacer algo y pasar al siguiente** (llamando a `next()`)
- **Cortar la cadena y responder él mismo** (por ejemplo, devolver un 401 si falta el token)

### Anatomía de un middleware

Un middleware es una función con **tres parámetros**: `req`, `res` y `next`.

```javascript
const miMiddleware = (req, res, next) => {
  // Hacer algo con req
  console.log("Pasando por el middleware");
  next(); // pasar al siguiente
};
```

La única diferencia con un handler es ese tercer parámetro `next`. Si lo llamáis, la petición sigue su camino. Si no lo llamáis y no respondéis tampoco, la petición se queda colgada para siempre.

---

## Primer middleware: un logger

Vamos a escribir un middleware que loguee cada petición que entra. En `index.js`:

```javascript
const express = require("express");
const app = express();

// Middleware: se ejecuta en TODAS las peticiones
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});

app.get("/api/notas", (req, res) => {
  res.json([{ id: 1, titulo: "Test" }]);
});

app.listen(3000, () => console.log("Servidor en puerto 3000"));
```

Arrancad con `npm run dev`, haced peticiones desde el navegador y mirad la terminal:

```
[2026-05-27T10:15:23.456Z] GET /api/notas
[2026-05-27T10:15:24.789Z] GET /favicon.ico
```

> Cada vez que el navegador abre una pestaña intenta cargar el `favicon.ico`. Por eso veis esas peticiones extra. Es normal.

**Clave aquí:** `app.use()` registra un middleware que se aplica a **todas las rutas**. El orden importa: los middlewares se ejecutan en el orden en que los registráis.

---

## Middleware nativo: `express.json()`

Hasta ahora solo habéis hecho `GET`. ¿Cómo recibe el servidor datos en un `POST`?

Probadlo sin nada:

```javascript
app.post("/api/notas", (req, res) => {
  console.log("Body recibido:", req.body);
  res.json({ recibido: req.body });
});
```

Si hacéis un `POST` con datos JSON desde Postman o cualquier cliente HTTP, veréis:

```
Body recibido: undefined
```

`req.body` es `undefined` porque **Express por defecto no parsea el body**. Hay que decírselo con un middleware:

```javascript
app.use(express.json());

app.post("/api/notas", (req, res) => {
  console.log("Body recibido:", req.body);
  res.json({ recibido: req.body });
});
```

Ahora sí. `express.json()` es un middleware que viene incluido en Express: lee el body de la petición y, si es JSON, lo parsea y lo deja en `req.body`.

> **Importante:** este middleware tiene que ir registrado **antes** de las rutas que vayan a usar `req.body`. Por eso `app.use(express.json())` se pone al principio del archivo.

### Hacer POST de verdad

Vamos a crear una nota nueva:

```javascript
const express = require("express");
const app = express();

app.use(express.json());

const notas = [{ id: 1, titulo: "Primera nota", contenido: "Hola mundo" }];

app.get("/api/notas", (req, res) => {
  res.json(notas);
});

app.post("/api/notas", (req, res) => {
  const { titulo, contenido } = req.body;

  if (!titulo || !contenido) {
    return res.status(400).json({ error: "Falta título o contenido" });
  }

  const nuevaNota = {
    id: notas.length + 1,
    titulo,
    contenido,
  };

  notas.push(nuevaNota);
  res.status(201).json(nuevaNota);
});

app.listen(3000);
```

Como el navegador no sabe hacer `POST` con body directamente, hoy vamos a probarlo de dos maneras:

### Opción A: desde la consola del navegador

Abrid DevTools en cualquier pestaña y pegad:

```javascript
fetch("http://localhost:3000/api/notas", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ titulo: "Desde fetch", contenido: "¡Funciona!" }),
})
  .then((r) => r.json())
  .then(console.log);
```

### Opción B: REST Client en VS Code

Instalad la extensión **REST Client** (Huachao Mao). Crear un archivo `peticiones.http` en el proyecto:

```http
### Listar notas
GET http://localhost:3000/api/notas

### Crear nota
POST http://localhost:3000/api/notas
Content-Type: application/json

{
  "titulo": "Desde REST Client",
  "contenido": "Mucho más cómodo que fetch"
}
```

Veréis un botón "Send Request" encima de cada petición. Clic y respuesta abajo.

> El lunes que viene veremos Postman. REST Client en VS Code es la alternativa ligera que no requiere otra app.

---

## El objeto `req` a fondo

Toda la información que llega del cliente vive en `req`. Las cuatro fuentes principales:

### 1. `req.params` — parámetros de la URL

Ya los conocéis del día anterior:

```javascript
app.get("/api/notas/:id", (req, res) => {
  // GET /api/notas/42
  console.log(req.params.id); // "42"
});
```

Se definen con `:` en la ruta. Siempre son strings.

### 2. `req.query` — la query string

Es lo que va después del `?` en la URL:

```javascript
app.get("/api/notas", (req, res) => {
  // GET /api/notas?limite=10&orden=fecha
  console.log(req.query);
  // { limite: "10", orden: "fecha" }
});
```

Útil para filtros, paginación, ordenación. **Siempre strings**, también.

Ejemplo aplicado:

```javascript
app.get("/api/notas", (req, res) => {
  const { limite } = req.query;

  if (limite) {
    return res.json(notas.slice(0, Number(limite)));
  }

  res.json(notas);
});
```

Probad: `GET /api/notas?limite=2` → solo las 2 primeras.

### 3. `req.body` — el body de la petición

Solo existe en `POST`, `PUT`, `PATCH`. Necesita `express.json()` para estar parseado.

```javascript
app.post("/api/notas", (req, res) => {
  console.log(req.body); // { titulo: "...", contenido: "..." }
});
```

### 4. `req.headers` — los headers de la petición

Metadatos. Los headers más típicos:

```javascript
app.get("/api/info", (req, res) => {
  res.json({
    "user-agent": req.headers["user-agent"],
    "content-type": req.headers["content-type"],
    authorization: req.headers["authorization"],
  });
});
```

Los nombres de los headers **siempre llegan en minúscula**, da igual cómo los envíe el cliente.

### Resumen visual

```
URL:    GET /api/notas/42?orden=fecha
                       │      └────► req.query.orden = "fecha"
                       └───────────► req.params.id = "42"

BODY:   { "titulo": "Hola" }  ─────► req.body.titulo = "Hola"

HEADERS: Authorization: Bearer xyz  ─► req.headers.authorization = "Bearer xyz"
```

---

## Middlewares personalizados útiles

### Logger más completo

```javascript
const logger = (req, res, next) => {
  const inicio = Date.now();
  console.log(`→ ${req.method} ${req.url}`);

  // Cuando la respuesta termine, loguear el tiempo
  res.on("finish", () => {
    const ms = Date.now() - inicio;
    console.log(`← ${req.method} ${req.url} ${res.statusCode} (${ms}ms)`);
  });

  next();
};

app.use(logger);
```

### Middleware aplicado a una ruta concreta

`app.use(middleware)` aplica a todo. Pero también podéis aplicar un middleware **solo a una ruta**:

```javascript
const verificarApiKey = (req, res, next) => {
  const apiKey = req.headers["x-api-key"];

  if (apiKey !== "secreto123") {
    return res.status(401).json({ error: "API key inválida o ausente" });
  }

  next();
};

// Solo esta ruta tiene la protección
app.get("/api/admin", verificarApiKey, (req, res) => {
  res.json({ mensaje: "Bienvenido, admin" });
});
```

Probadlo:

- `GET /api/admin` sin header → 401
- `GET /api/admin` con header `x-api-key: secreto123` → 200

Este patrón es **exactamente** el que usaremos con JWT cuando lleguemos a autenticación.

---

## CORS: el que da problemas todos los días

### El problema

Imaginad: vuestro frontend de React (que haréis en M3) corre en `http://localhost:5173`. Vuestro backend corre en `http://localhost:3000`. El frontend hace `fetch("http://localhost:3000/api/notas")`.

El navegador lo **bloquea** con un error feo en consola:

```
Access to fetch at 'http://localhost:3000/api/notas' from origin
'http://localhost:5173' has been blocked by CORS policy
```

### Por qué pasa

CORS = **Cross-Origin Resource Sharing**. Es una política de seguridad del navegador (no del servidor) que **por defecto bloquea las peticiones a otros dominios** desde JavaScript.

¿Por qué? Para evitar que una web maliciosa haga peticiones a, por ejemplo, vuestro banco usando vuestras cookies sin permiso.

**Solo afecta al navegador.** Postman, REST Client, `curl`, otro backend… ninguno tiene CORS. Por eso hasta ahora no os ha dado problemas.

### La solución

El servidor tiene que **explícitamente decir** qué orígenes le pueden hacer peticiones. En Express, con un paquete:

```bash
npm install cors
```

```javascript
const cors = require("cors");

app.use(cors()); // permite TODOS los orígenes
```

**Eso es todo.** Con `app.use(cors())` ya permitís peticiones desde cualquier sitio.

### CORS más restrictivo (para cuando despleguéis)

En producción no queréis que cualquiera os llame. Lo configuráis así:

```javascript
app.use(
  cors({
    origin: "https://mi-frontend.netlify.app",
  }),
);
```

O con varios orígenes:

```javascript
app.use(
  cors({
    origin: ["http://localhost:5173", "https://mi-frontend.netlify.app"],
  }),
);
```

Por ahora, durante el desarrollo, `app.use(cors())` sin opciones os vale.

---

## Aplicación al proyecto

Ahora lleváis todo esto a vuestra API personal. Lo mínimo a conseguir hoy:

1. **`express.json()` registrado** antes de las rutas.
2. **`cors()` instalado y activado.**
3. **Logger propio**: un middleware que imprima cada petición que entra con método, URL y status code de respuesta.
4. **`POST /api/[entidad]`** que cree un nuevo elemento. Devuelve 201 con el objeto creado.
5. **`req.query` aplicado**: alguna ruta `GET` que acepte un parámetro de filtrado o paginación (por ejemplo, `?limite=5` o `?busqueda=texto`).
6. **Archivo `peticiones.http`** con las peticiones de prueba documentadas.

Cuando lo tengáis:

```bash
git add .
git commit -m "feat: middlewares, CORS y POST"
git push
```

Y movéis las tarjetas correspondientes del Kanban a `Done`.

---

## Cierre

Lo que tenéis que llevar grabado:

- Un middleware es `(req, res, next) => { ... }`. La diferencia con un handler es el `next`.
- El **orden** de los `app.use()` importa. Logger y `express.json()` siempre arriba.
- `req.params` viene de la URL (`:id`). `req.query` viene del `?clave=valor`. `req.body` viene del body (necesita `express.json()`).
- CORS lo arregla `app.use(cors())`. Es problema del navegador, no del backend.
- Si un middleware no llama a `next()` ni responde, la petición se cuelga.
