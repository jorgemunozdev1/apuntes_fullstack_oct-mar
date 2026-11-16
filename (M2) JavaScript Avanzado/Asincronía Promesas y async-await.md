# D15 · Asincronía: Promesas y async/await

---

## 🔥 Warm-up — Kata: replicar `find`

Mientras llegan, en la pizarra:

> Implementa `miFind(array, callback)` que devuelva el **primer elemento** que cumpla la condición, o `undefined` si ninguno la cumple. Sin usar `find` ni `filter`.

```js
// Resultado esperado:
miFind([1, 2, 3, 4], (n) => n > 2); // 3
miFind([1, 2, 3], (n) => n > 10); // undefined
```

<details>
<summary>Solución</summary>

```js
function miFind(array, callback) {
  for (let i = 0; i < array.length; i++) {
    if (callback(array[i], i, array)) {
      return array[i];
    }
  }
  return undefined;
}
```

</details>

---

## 🧠 ¿Por qué necesitamos asincronía?

### El problema

JavaScript es **single-thread**: hace una cosa a la vez. Si una tarea tarda (pedir datos a un servidor, leer un fichero), bloquear el hilo significa **congelar la página**.

```js
// ❌ Imagina que esto bloquea 3 segundos
const datos = pedirAlServidor(); // la página se queda congelada
console.log(datos);
console.log("Esto no se ve hasta que termine lo anterior");
```

### La solución: código asíncrono

JavaScript dice: _"vale, lanzo la tarea lenta, pero sigo con lo demás. Cuando termine, me avisas."_

```js
console.log("1");
setTimeout(() => console.log("2"), 0); // se aparta
console.log("3");
// Imprime: 1, 3, 2
```

### Modelo mental: la cola de tareas

> Lo que tarda (fetch, timers, eventos) **se aparta** en una cola. JavaScript sigue ejecutando el resto. Cuando el hilo principal está libre, recoge lo que acabó y lo ejecuta.

Esto explica el orden raro de los `console.log` cuando hay asincronía.

---

## 🧠 Teoría 2 — Promesas

### ¿Qué es una Promesa?

Un objeto que representa un **valor que aún no existe** pero existirá (o fallará) en el futuro. Tres estados:

- **pending** — esperando
- **fulfilled** — resolvió bien (tenemos valor)
- **rejected** — falló (tenemos error)

```js
// fetch devuelve una Promesa
const promesa = fetch("https://api.github.com/users/octocat");
console.log(promesa); // Promise { <pending> }
```

### Consumir una Promesa: `.then` / `.catch` / `.finally`

```js
fetch("https://api.github.com/users/octocat")
  .then((response) => response.json()) // 1º then: parsear JSON
  .then((data) => console.log(data)) // 2º then: usar los datos
  .catch((error) => console.error(error)) // si algo falla
  .finally(() => console.log("Terminado")); // pase lo que pase
```

**Idea clave:** cada `.then` recibe lo que devolvió el anterior. Si devuelves otra Promesa, encadena. Esto se llama **chaining**.

### Encadenar varias peticiones

```js
fetch("/api/usuario/1")
  .then((res) => res.json())
  .then((usuario) => fetch(`/api/posts?autor=${usuario.id}`))
  .then((res) => res.json())
  .then((posts) => console.log(posts));
```

### El "callback hell" que evitamos

Antes de las Promesas, esto era pesadilla anidando callbacks. Las Promesas **aplanan** ese código.

---

## 🧠 Teoría 3 — `async/await`

### La sintaxis que parece síncrona

`async/await` es **azúcar sintáctico** sobre Promesas. El código se lee de arriba abajo, como si fuera síncrono, pero por debajo siguen siendo Promesas.

```js
// Con .then
function cargarUsuario() {
  return fetch("/api/usuario/1")
    .then((res) => res.json())
    .then((data) => console.log(data));
}

// Con async/await
async function cargarUsuario() {
  const res = await fetch("/api/usuario/1");
  const data = await res.json();
  console.log(data);
}
```

### Reglas básicas

1. `await` **solo funciona dentro de una función `async`**.
2. Una función `async` **siempre devuelve una Promesa**, lo quieras o no.
3. `await` pausa la ejecución de **esa función** hasta que la Promesa resuelve. No bloquea el resto del programa.

### Manejo de errores con `try/catch`

Aquí está la ventaja real: errores que parecen errores normales.

```js
async function cargarUsuario(id) {
  try {
    const res = await fetch(`/api/usuario/${id}`);
    if (!res.ok) throw new Error(`Error ${res.status}`);
    const data = await res.json();
    return data;
  } catch (error) {
    console.error("Algo falló:", error.message);
    return null;
  } finally {
    console.log("Petición terminada");
  }
}
```

> ⚠️ **Trampa habitual:** `fetch` **no lanza error** si la respuesta es 404 o 500. Solo falla si la red está caída. Por eso hay que comprobar `res.ok` a mano.

---

## 💻 Cadena de fetchs con manejo de errores

### Contexto

Vamos a montar una mini-app que, dado el nombre de un usuario de GitHub:

1. Pide su perfil.
2. Pide sus repositorios.
3. Muestra el perfil + los 5 repos más recientes.

### Starter code

Crea `index.html` y `script.js`:

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <title>GitHub Explorer</title>
  </head>
  <body>
    <input id="input-user" placeholder="ej: octocat" />
    <button id="btn-buscar">Buscar</button>
    <div id="resultado"></div>
    <script src="script.js"></script>
  </body>
</html>
```

```js
// script.js
const input = document.getElementById("input-user");
const boton = document.getElementById("btn-buscar");
const resultado = document.getElementById("resultado");

boton.addEventListener("click", () => {
  const username = input.value.trim();
  if (!username) return;
  // TODO: llamar a buscarUsuario(username)
});

async function buscarUsuario(username) {
  // TODO: implementar
}
```

### Pasos guiados

**Paso 1 ** — Pedir el perfil.

```js
async function buscarUsuario(username) {
  try {
    const res = await fetch(`https://api.github.com/users/${username}`);
    if (!res.ok) throw new Error(`Usuario no encontrado (${res.status})`);
    const usuario = await res.json();
    console.log(usuario);
  } catch (error) {
    resultado.textContent = error.message;
  }
}
```

**Paso 2** — Encadenar con los repos.

```js
async function buscarUsuario(username) {
  try {
    const resUsuario = await fetch(`https://api.github.com/users/${username}`);
    if (!resUsuario.ok) throw new Error(`Usuario no encontrado`);
    const usuario = await resUsuario.json();

    const resRepos = await fetch(usuario.repos_url);
    if (!resRepos.ok) throw new Error(`No se pudieron cargar los repos`);
    const repos = await resRepos.json();

    pintar(usuario, repos);
  } catch (error) {
    resultado.innerHTML = `<p style="color:red">${error.message}</p>`;
  }
}
```

**Paso 3** — Función `pintar`.

```js
function pintar(usuario, repos) {
  const top5 = repos
    .sort((a, b) => new Date(b.updated_at) - new Date(a.updated_at))
    .slice(0, 5);

  resultado.innerHTML = `
    <h2>${usuario.name || usuario.login}</h2>
    <img src="${usuario.avatar_url}" width="100" />
    <p>${usuario.bio || "Sin bio"}</p>
    <h3>Últimos 5 repos:</h3>
    <ul>
      ${top5.map((r) => `<li><a href="${r.html_url}" target="_blank">${r.name}</a></li>`).join("")}
    </ul>
  `;
}
```

**Paso 4** — Reto: añadir un **indicador de carga** (`resultado.textContent = "Cargando..."`) antes del primer await, y manejar el caso de que el usuario no tenga repos.

**Paso 5** — Reto avanzado: probar `Promise.all` para pedir perfil + repos **en paralelo** y comparar tiempos.

```js
const [resUsuario, resRepos] = await Promise.all([
  fetch(`https://api.github.com/users/${username}`),
  fetch(`https://api.github.com/users/${username}/repos`),
]);
```

---

## 🤔 Preguntas de reflexión

1. ¿Qué pasa si quito el `await` de delante de `fetch`?
2. ¿Por qué hace falta `await res.json()` si ya tenemos la respuesta?
3. Diferencia entre encadenar awaits y usar `Promise.all`.
4. ¿En qué casos prefieres `.then` sobre `async/await`?

---

## 📌 Tarea para el proyecto

Integra **al menos una llamada async** en tu proyecto del Módulo 1:

- Si tu API tarda, añade un indicador de "Cargando...".
- Maneja al menos un error (red caída o respuesta no-ok) mostrando un mensaje al usuario.
- Usa `async/await` con `try/catch` (no `.then` encadenados).

Sube los cambios a tu rama.
