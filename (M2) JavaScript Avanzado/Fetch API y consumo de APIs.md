# D14 · Fetch API y consumo de APIs

**Módulo 1 · JavaScript Avanzado · Viernes 15/05/2026**

---

## 🧠 Conexión con ayer

Ayer hicimos _data crunching_ con un `pokemons.json` **que ya estaba en local**. Hoy ese mismo JSON lo pedimos al servidor en vivo: ya no lo tenemos, lo **traemos** cada vez que la página se carga.

Lo demás (map, filter, reduce, render con `innerHTML`) no cambia. Solo cambia **de dónde vienen los datos**.

---

# 📚 Anatomía de una petición HTTP

## ¿Qué es una API?

Una **API** (Application Programming Interface) es un "contrato" que un servidor publica para que otros programas le pidan datos o le pidan que haga cosas. Cuando hablamos de **API HTTP** o **API REST**, nos referimos a que ese contrato funciona sobre el protocolo HTTP — el mismo que usa el navegador para cargar webs.

Una API expone **endpoints** (URLs). Cada endpoint hace algo concreto:

```
GET  https://pokeapi.co/api/v2/pokemon          → lista de pokémons
GET  https://pokeapi.co/api/v2/pokemon/pikachu  → datos de Pikachu
GET  https://pokeapi.co/api/v2/pokemon/25       → datos del pokémon nº 25
```

## Las 4 partes de una petición

Cuando hacéis un `fetch`, por debajo se está montando esto:

```
┌──────────────────────────────────────────────────┐
│  MÉTODO    URL                                   │
│  GET       https://pokeapi.co/api/v2/pokemon/25  │
│                                                  │
│  HEADERS                                         │
│  Accept: application/json                        │
│  Content-Type: application/json                  │
│                                                  │
│  BODY                                            │
│  (vacío en GET, JSON en POST/PUT/PATCH)          │
└──────────────────────────────────────────────────┘
```

### Métodos HTTP (los que vais a usar este año)

| Método          | Para qué sirve            | ¿Lleva body?   |
| --------------- | ------------------------- | -------------- |
| `GET`           | Pedir datos               | No             |
| `POST`          | Crear algo nuevo          | Sí             |
| `PUT` / `PATCH` | Actualizar algo existente | Sí             |
| `DELETE`        | Borrar algo               | Normalmente no |

> Hoy **solo hacemos GET**. POST/PUT/DELETE entran de lleno en M2 (backend).

### Status codes — la respuesta del servidor

El servidor siempre devuelve un **código de estado**. Los rangos importantes:

| Rango   | Significado                                | Ejemplos típicos                                       |
| ------- | ------------------------------------------ | ------------------------------------------------------ |
| **2xx** | ✅ Todo bien                               | `200 OK`, `201 Created`                                |
| **3xx** | 🔀 Redirección                             | `301 Moved`, `304 Not Modified`                        |
| **4xx** | ❌ Error del cliente (tú la has liado)     | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` |
| **5xx** | 💥 Error del servidor (ellos la han liado) | `500 Internal Server Error`, `503 Service Unavailable` |

**Regla mental:** _4xx = la has pifiado tú, 5xx = la han pifiado ellos._

---

# 🛠️ Mi primer `fetch`

**Setup:** crea una carpeta `d14-fetch/` con `index.html`, `style.css` y `app.js`. Enlaza el JS al final del `<body>`.

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <title>D14 · PokeAPI</title>
  </head>
  <body>
    <h1>Pokédex</h1>
    <div id="resultado"></div>
    <script src="app.js"></script>
  </body>
</html>
```

## Paso 1 — Disparar una petición y mirarla en consola

```js
// app.js
fetch("https://pokeapi.co/api/v2/pokemon/pikachu").then((response) => {
  console.log(response);
});
```

Abrid la consola. Vais a ver un objeto **`Response`**. **No es el pokémon todavía.** Es la "envoltura" de la respuesta: contiene el status, los headers, y el cuerpo aún sin procesar.

```js
fetch("https://pokeapi.co/api/v2/pokemon/pikachu").then((response) => {
  console.log("Status:", response.status); // 200
  console.log("OK?:", response.ok); // true
  console.log("Headers:", response.headers); // Headers {}
});
```

## Paso 2 — Convertir el cuerpo a JSON

El cuerpo viene en formato JSON pero como texto crudo. Hay que **parsearlo**. El método `.json()` hace eso — y devuelve **otra promesa**.

```js
fetch("https://pokeapi.co/api/v2/pokemon/pikachu")
  .then((response) => response.json()) // ← devuelve otra promesa
  .then((pokemon) => {
    console.log(pokemon); // ya es el objeto JS
    console.log(pokemon.name); // "pikachu"
    console.log(pokemon.weight); // 60
  });
```

**Idea clave:** cada `.then()` recibe lo que devuelve el anterior. Si devuelves una promesa, el siguiente `.then()` espera a que se resuelva. Eso es lo que nos permite encadenar.

## Paso 3 — Probar un error

```js
fetch("https://pokeapi.co/api/v2/pokemon/no-existe-este-bicho")
  .then((response) => {
    console.log(response.status); // 404
    console.log(response.ok); // false
    return response.json();
  })
  .then((data) => console.log(data));
```

⚠️ **Trampa importante:** `fetch` **no lanza error** con un 404 o un 500. Solo lanza error si la petición no llega (sin internet, DNS roto, CORS bloqueado…). Si el servidor responde "no encontrado", `fetch` lo considera una respuesta válida.

**Esto es responsabilidad nuestra:**

```js
fetch("https://pokeapi.co/api/v2/pokemon/no-existe")
  .then((response) => {
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return response.json();
  })
  .then((pokemon) => console.log(pokemon))
  .catch((error) => console.error("Algo ha fallado:", error.message));
```

---

# 📚 BLOQUE TEÓRICO 2 — Promesas y `.then()`

## ¿Qué es una promesa?

Una **promesa** es un objeto que representa el resultado **futuro** de una operación asíncrona. Cuando llamas a `fetch()`, no recibes los datos al momento — recibes una **promesa de que llegarán** (o de que algo fallará).

### Los 3 estados de una promesa

```
        ┌──────────────┐
        │   pending    │  ← acaba de empezar, todavía no hay resultado
        └──────┬───────┘
               │
       ┌───────┴────────┐
       ▼                ▼
┌─────────────┐  ┌─────────────┐
│  fulfilled  │  │   rejected  │
│  (✅ éxito) │  │  (❌ error) │
└─────────────┘  └─────────────┘
```

- **`.then(callback)`** se ejecuta si la promesa se cumple (fulfilled).
- **`.catch(callback)`** se ejecuta si falla (rejected).
- **`.finally(callback)`** se ejecuta siempre, pase lo que pase.

## El patrón completo

```js
fetch(URL)
  .then((response) => {
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  })
  .then((data) => {
    // hacemos algo con data
  })
  .catch((error) => {
    // capturamos cualquier error de la cadena
    console.error(error);
  })
  .finally(() => {
    // limpieza, ocultar spinners, etc.
  });
```

**Por qué un solo `.catch()` al final funciona:** los errores en una cadena de promesas **caen hacia abajo** hasta que encuentran un `.catch()`. Si lanzas un `throw` en el primer `.then()`, el segundo `.then()` se salta y se va directo al `.catch()`.

---

# 🛠️ Lista de pokémons en el DOM

**Objetivo:** pintar los primeros 20 pokémons con su nombre, imagen y tipos.

## La estructura de PokeAPI

Hay dos endpoints que vais a combinar:

```
GET /pokemon?limit=20      → lista con { name, url } de 20 pokémons
GET /pokemon/{nombre}      → detalle completo de uno
```

La primera llamada te devuelve solo nombres y URLs. Para tener imagen y tipos, hay que llamar al detalle de cada uno.

## Paso 1 — Traer la lista básica

```js
const API = "https://pokeapi.co/api/v2";

fetch(`${API}/pokemon?limit=20`)
  .then((res) => res.json())
  .then((data) => {
    console.log(data.results); // array de { name, url }
  });
```

## Paso 2 — Para cada pokémon, traer su detalle

Aquí entra `Promise.all`. Recibe un array de promesas y devuelve **una sola** promesa que se resuelve cuando **todas** terminan.

```js
fetch(`${API}/pokemon?limit=20`)
  .then((res) => res.json())
  .then((data) => {
    // array de promesas, una por pokémon
    const promesasDetalle = data.results.map((p) =>
      fetch(p.url).then((r) => r.json()),
    );
    return Promise.all(promesasDetalle);
  })
  .then((pokemons) => {
    // pokemons es un array con los 20 objetos completos
    console.log(pokemons);
  });
```

**Idea clave:** `Promise.all` lanza las 20 peticiones **en paralelo**, no una detrás de otra. Mucho más rápido.

## Paso 3 — Renderizar con template strings

```js
function renderizar(pokemons) {
  const contenedor = document.getElementById("resultado");
  contenedor.innerHTML = pokemons
    .map(
      (p) => `
    <article class="card">
      <img src="${p.sprites.front_default}" alt="${p.name}">
      <h2>${p.name}</h2>
      <p>${p.types.map((t) => t.type.name).join(" · ")}</p>
    </article>
  `,
    )
    .join("");
}

fetch(`${API}/pokemon?limit=20`)
  .then((res) => res.json())
  .then((data) =>
    Promise.all(data.results.map((p) => fetch(p.url).then((r) => r.json()))),
  )
  .then(renderizar)
  .catch((err) => {
    document.getElementById("resultado").innerHTML =
      `<p class="error">Ups: ${err.message}</p>`;
  });
```

## Paso 4 — CSS rápido para que se vea decente

```css
#resultado {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 1rem;
  padding: 1rem;
}
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
  text-align: center;
  text-transform: capitalize;
}
.card img {
  width: 96px;
  height: 96px;
}
.error {
  color: crimson;
  font-weight: bold;
}
```

---

# 🎯 RETO

Partiendo de la práctica guiada 2, añadid **una** de estas dos funcionalidades (la que prefiráis):

### Opción A — Buscador por nombre

Un `<input>` arriba. Cuando el usuario escribe, filtras los pokémons ya cargados (no haces fetch otra vez).

> 💡 Pista: necesitas guardar los pokémons en una variable fuera del fetch para poder filtrarlos después. Usa `filter` (D13).

### Opción B — Filtro por tipo

Un `<select>` con los tipos disponibles (fire, water, grass, electric…). Al cambiar, muestras solo los del tipo elegido.

> 💡 Pista: un pokémon puede tener varios tipos. Mira si **alguno** coincide con `some` (D13).

---

# 💡 Errores típicos y cómo detectarlos

| Síntoma                                          | Causa probable                                     | Solución                                               |
| ------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------ |
| `undefined` al imprimir el pokémon               | Olvidaste `return` en algún `.then()`              | Asegúrate de devolver la promesa                       |
| `[object Promise]` en pantalla                   | Estás imprimiendo la promesa, no su resultado      | Espera con otro `.then()`                              |
| `TypeError: Cannot read properties of undefined` | Estructura JSON distinta de la esperada            | `console.log` de la respuesta para inspeccionarla      |
| Nada se ve, sin errores                          | Script enlazado antes del DOM o ID mal escrito     | Mueve `<script>` al final o usa `defer`                |
| Error de CORS                                    | El servidor no permite peticiones desde tu dominio | No aplica con PokeAPI (es abierta), pero ojo con otras |

---

# 📝 Preguntas de cierre

1. ¿Qué diferencia hay entre el objeto `Response` y los datos que realmente queremos?
2. ¿Por qué `fetch` **no** considera un 404 como un error?
3. ¿Qué hace `Promise.all` y por qué es más eficiente que encadenar fetches uno a uno?
4. ¿Dónde va el `.catch()` y por qué solo hace falta uno al final?

---
