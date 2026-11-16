# Día 21 — Express I: primer servidor + rutas GET

---

## Antes de empezar

Comprueba que tienes todo listo:

- Node funcionando: `node -v`
- El repo M2 en GitHub
- Idea de proyecto y tablero Kanban

Si algo de esto te falta, resuélvelo antes de seguir. No avances al día 21 sin haber cerrado el día 20.

## Teoría: HTTP + Express

### ¿Qué es un servidor HTTP?

```
                    petición HTTP
   [NAVEGADOR]  ──────────────────►  [SERVIDOR]
                                          │
                    respuesta HTTP        │
              ◄───────────────────────────┘
```

Una petición HTTP tiene:

- **Método**: GET, POST, PUT, DELETE, PATCH
- **URL**: `/usuarios/42`
- **Headers**: metadatos (Content-Type, Authorization…)
- **Body** (opcional): datos enviados (solo en POST, PUT, PATCH)

Una respuesta HTTP tiene:

- **Status code**: 200, 201, 404, 500…
- **Headers**
- **Body**: lo que devuelves (HTML, JSON, imagen…)

Cuando tu app de M1 hacía `fetch("https://api.algo.com/cosas")`, alguien al otro lado tenía un servidor escuchando y respondiendo. Ahora montas tú ese alguien.

### Node sin Express

Este código funciona solo con Node, sin librerías:

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  if (req.url === "/" && req.method === "GET") {
    res.statusCode = 200;
    res.setHeader("Content-Type", "application/json");
    res.end(JSON.stringify({ mensaje: "Hola" }));
  } else if (req.url === "/usuarios" && req.method === "GET") {
    res.statusCode = 200;
    res.setHeader("Content-Type", "application/json");
    res.end(JSON.stringify([{ id: 1, nombre: "Ana" }]));
  } else {
    res.statusCode = 404;
    res.end("No encontrado");
  }
});

server.listen(3000);
```

Funciona: Node trae `http` de serie, no necesitas nada más. Pero fíjate en lo verboso que es. Imagina 30 rutas, y que cada una valida headers, parsea el body y gestiona errores. Sería un infierno.

### ¿Qué es Express?

Express es una **librería minimalista** que envuelve el módulo `http` de Node y te da:

- Sintaxis declarativa para rutas: `app.get("/usuarios", handler)`
- **Middlewares**: trozos de código que se ejecutan en cadena (lo veremos más adelante)
- Parsing automático de JSON, params, query, etc.
- Un ecosistema enorme de plugins

Express no es magia: por debajo es el módulo `http` con muchos atajos. Lo importante es que es **el estándar de facto** en Node desde hace años. Si trabajas con Node, vas a trabajar con Express.

Existen alternativas (Fastify, Koa, Hapi, NestJS), pero no nos detenemos en ellas.

---

## Primer servidor Express

### Setup

En el repo del proyecto M2:

```bash
npm install express
```

Mira tu `package.json`: ahora aparece `express` en `dependencies`.

Crea `index.js`:

```javascript
const express = require("express");

const app = express();
const PUERTO = 3000;

app.listen(PUERTO, () => {
  console.log(`Servidor escuchando en http://localhost:${PUERTO}`);
});
```

Ejecútalo:

```bash
node index.js
```

Abre el navegador en `http://localhost:3000` → **error 404**. Bien: el servidor está vivo pero no tiene ninguna ruta. Páralo con `Ctrl+C`.

Recuerda: `Ctrl+C` para parar un proceso en la terminal. No cierres la pestaña.

### Primera ruta

```javascript
const express = require("express");

const app = express();
const PUERTO = 3000;

app.get("/", (req, res) => {
  res.send("Hola desde mi primera API");
});

app.listen(PUERTO, () => {
  console.log(`Servidor escuchando en http://localhost:${PUERTO}`);
});
```

Reinicia y refresca el navegador → texto plano.

Fíjate en el patrón: `app.METODO("ruta", (req, res) => { ... })`. Ese callback es el **handler**. Recibe siempre `req` (request, la petición) y `res` (response, la respuesta). En `res` tienes métodos para enviar la respuesta de vuelta.

Prueba `res.send()` con distintos tipos:

```javascript
app.get("/texto", (req, res) => {
  res.send("Esto es texto plano");
});

app.get("/html", (req, res) => {
  res.send("<h1>Esto es HTML</h1><p>Express lo detecta</p>");
});

app.get("/numero", (req, res) => {
  res.send(String(42)); // res.send con número se confunde con status code
});
```

> ⚠️ **Aviso útil:** `res.send(42)` no envía "42", lo interpreta como **status code 42** y rompe. Si quieres enviar un número, conviértelo a string o usa `res.json()`.

### JSON

Esto es lo que harás el 95% del tiempo en M2:

```javascript
app.get("/api/saludo", (req, res) => {
  res.json({ mensaje: "Hola", timestamp: new Date() });
});

app.get("/api/usuarios", (req, res) => {
  res.json([
    { id: 1, nombre: "Ana", edad: 28 },
    { id: 2, nombre: "Luis", edad: 34 },
    { id: 3, nombre: "Carmen", edad: 22 },
  ]);
});
```

Pruébalo en el navegador. Abre DevTools → Network → mira los headers de respuesta: verás `Content-Type: application/json`. Express lo ha puesto solo porque has usado `res.json()`. Esa es la convención: cuando devuelves datos, devuelves JSON.

**Idea importante:** un servidor backend bien diseñado **devuelve datos, no HTML**. El HTML lo monta el frontend (M3 con React) consumiendo estos JSON. Por eso decimos _API REST_.

### Datos "en memoria"

Crea un array de datos relacionado con tu proyecto (ejemplo con notas):

```javascript
const notas = [
  { id: 1, titulo: "Comprar pan", contenido: "Y leche también" },
  { id: 2, titulo: "Llamar al fontanero", contenido: "Gotea el grifo" },
  {
    id: 3,
    titulo: "Clase de Express",
    contenido: "Mola más de lo que pensaba",
  },
];

app.get("/api/notas", (req, res) => {
  res.json(notas);
});
```

Reinicia y prueba.

Esto vive en memoria: si reinicias el servidor, se pierde. Es un problema temporal — lo solucionaremos más adelante con persistencia a fichero, y después con MongoDB. Por ahora nos vale así para centrarnos en Express.

---

## Rutas dinámicas, nodemon y status codes

### Nodemon

¿Cuántas veces has reiniciado ya el servidor? Nodemon lo hace por ti: reinicia el servidor solo cada vez que guardas un cambio.

```bash
npm install -D nodemon
```

La flag `-D` lo guarda en `devDependencies`: son cosas que solo necesitas en desarrollo, no en producción. Render no necesita nodemon.

Edita `package.json`:

```json
"scripts": {
  "start": "node index.js",
  "dev": "nodemon index.js"
}
```

Ejecútalo:

```bash
npm run dev
```

Cambia algo en `index.js`, guarda → se reinicia solo.

A partir de ahora usa siempre `npm run dev` mientras programas. `npm start` es para producción.

### Rutas dinámicas con `req.params`

```javascript
app.get("/api/notas/:id", (req, res) => {
  console.log("Parámetros recibidos:", req.params);
  res.json({ id: req.params.id });
});
```

Prueba:

- `http://localhost:3000/api/notas/1` → `{ "id": "1" }`
- `http://localhost:3000/api/notas/42` → `{ "id": "42" }`
- `http://localhost:3000/api/notas/manzana` → `{ "id": "manzana" }`

> ⚠️ **Detalle clave:** `req.params.id` **siempre es string**, aunque le pases un número. Si quieres trabajar con número, conviértelo: `Number(req.params.id)`.

Ahora algo más útil: buscar una nota concreta.

```javascript
app.get("/api/notas/:id", (req, res) => {
  const id = Number(req.params.id);
  const nota = notas.find((n) => n.id === id);

  if (!nota) {
    return res.status(404).json({ error: "Nota no encontrada" });
  }

  res.json(nota);
});
```

Prueba:

- `/api/notas/1` → la nota
- `/api/notas/999` → error 404 con JSON

Fíjate en el `return`. Sin él, después de `res.status(404).json(...)` seguiría ejecutándose `res.json(nota)` y daría error: "no se puede enviar respuesta dos veces". Pon **siempre** `return` cuando salgas temprano de un handler.

### Múltiples parámetros

```javascript
app.get("/api/usuarios/:userId/notas/:notaId", (req, res) => {
  res.json({
    usuario: req.params.userId,
    nota: req.params.notaId,
  });
});
```

Útil para recursos anidados. Lo verás en cualquier API REST seria.

### Status codes

Estos son los que usarás en M2:

| Código                        | Significado                  | Cuándo                                 |
| ----------------------------- | ---------------------------- | -------------------------------------- |
| **200** OK                    | Todo bien                    | GET, PUT, DELETE exitosos              |
| **201** Created               | Recurso creado               | POST exitoso                           |
| **204** No Content            | OK sin body                  | DELETE exitoso (alternativa)           |
| **400** Bad Request           | El cliente la cagó           | Datos mal formados, validación fallida |
| **401** Unauthorized          | No autenticado               | Falta token o es inválido              |
| **403** Forbidden             | Autenticado pero sin permiso | Usuario normal accede a ruta de admin  |
| **404** Not Found             | No existe                    | ID que no está en BD                   |
| **500** Internal Server Error | La cagué yo                  | Errores del servidor                   |

En M2 vas a usar sobre todo 200, 201, 400, 401, 404 y 500. Memoriza esos seis.

Aplícalo:

```javascript
app.get("/api/saludo/:nombre", (req, res) => {
  const nombre = req.params.nombre;

  if (nombre.length < 2) {
    return res.status(400).json({ error: "El nombre es muy corto" });
  }

  res.status(200).json({ mensaje: `Hola, ${nombre}` });
});
```

`res.status(200)` se puede omitir, es el default. Pero ponerlo explícito es buena práctica cuando hay otros status en el mismo handler.

---

## Aplícalo a tu proyecto

Lleva esto a tu API personal. **Mínimo a conseguir hoy:**

1. Servidor Express corriendo con `npm run dev`
2. Ruta `GET /api/health` que devuelva `{ status: "ok" }` — útil para verificar que el servidor está vivo
3. Array de tu entidad principal en memoria (al menos 3-5 elementos de ejemplo)
4. `GET /api/[entidad]` — devuelve el array completo
5. `GET /api/[entidad]/:id` — devuelve uno por id, con 404 si no existe

Mientras programas, revisa que:

- Estás usando `npm run dev` (no `node index.js`)
- Tienes el `return` antes de `res.json(404)`
- Conviertes los IDs a `Number`
- El `.gitignore` sigue teniendo `node_modules/`

Cuando termines las 5 rutas, mueve la tarjeta correspondiente del Kanban a `Done` y haz commit:

```bash
git add .
git commit -m "feat: servidor Express con rutas GET básicas"
git push
```

---

## Repaso

Comprueba que sabes responder:

- ¿Qué hace Express que no hace `http` de Node?
- ¿Diferencia entre `res.send` y `res.json`?
- ¿Cómo se accede a un parámetro de URL?
- ¿Qué status code para "no existe"? ¿Y para "datos mal"?
- ¿Por qué `npm run dev` y no `node index.js`?

## Lo que viene después

En la siguiente sesión veremos middlewares y CORS. Te sonará raro al principio, pero es el concepto más importante de Express. Ven con el servidor de hoy funcionando y subido.

---

## Entrega de hoy

1. Push al repo con las rutas `GET` funcionando
2. Tarjetas correspondientes del Kanban movidas a `Done`
3. (Opcional) Captura de Postman / navegador con una de las rutas respondiendo, en el canal de Slack
