# React de verdad: Vite + tu primer componente

Ya hemos construido a mano un mini-React: un `estado`, una función `render()` que dibujaba la pantalla a partir de ese estado, y un `setEstado` que repintaba al cambiar los datos. Funcionaba… hasta que el input perdía el foco en cada tecla. Ese dolor te dejó claro qué problema resuelve React (el **Virtual DOM**) y de dónde salen sus piezas (`UI = f(estado)`, `useState`).

Ahora dejamos el juguete y montamos la cosa real. Vas a instalar React con **Vite**, entender qué te genera, arrancar el servidor de desarrollo y escribir tu **primer componente**. Y verás que casi todo lo de hoy es reconocer, con otro nombre, lo que ya montaste tú a mano.

---

## Paso 1 — ¿Por qué Vite y no un `<script>`?

En M0/M1 abrías un `.html` con un `<script>` dentro y listo. ¿Por qué hoy necesitamos instalar algo?

Porque React no se escribe en HTML, se escribe en **JSX** (ahora lo ves), y el navegador no entiende JSX. Hace falta una herramienta que **traduzca** tu código a algo que el navegador sí entienda, y que además te dé un servidor de desarrollo cómodo.

Esa herramienta es **Vite** (se pronuncia "vit", viene del francés _rápido_). Hace dos cosas que te importan:

- **Traduce** tu JSX y tus `import` modernos a JavaScript que el navegador ejecuta.
- Te levanta un **servidor de desarrollo** con _recarga en caliente_ (**HMR**): guardas un archivo y el navegador refleja el cambio al instante, **sin perder el estado de la página**.

> 💡 Tu mini-render parpadeaba entero por cambiar una letra. El HMR de Vite es la misma idea llevada al flujo de trabajo: actualiza solo lo que cambió en tu código, sin recargar toda la página a mano.

---

## Paso 2 — Crea el proyecto

Colócate en la carpeta donde quieras el proyecto y ejecuta:

```bash
npm create vite@latest mi-spa
```

Te hará un par de preguntas. Elige:

- **Framework** → `React`
- **Variant** → `JavaScript` (seguimos en JS puro, como todo el curso)

Si prefieres no responder nada, el mismo comando en una línea:

```bash
npm create vite@latest mi-spa -- --template react
```

Cuando termine, entra y instala las dependencias:

```bash
cd mi-spa
npm install
```

> ⚠️ `npm create vite` solo **genera los archivos**. Las librerías (React incluido) se bajan con `npm install`. Sin ese paso, `node_modules` no existe y nada arranca.

---

## Paso 3 — Tour del scaffold

Vite te ha dejado esta estructura (te marco lo que importa):

```
mi-spa/
├── node_modules/        ← las librerías instaladas (no se toca, no se sube a git)
├── public/              ← archivos estáticos tal cual (favicon, imágenes…)
├── src/                 ← AQUÍ vives tú
│   ├── App.jsx          ← tu primer componente
│   ├── App.css
│   ├── main.jsx         ← el punto de entrada: conecta React con el HTML
│   └── index.css
├── index.html           ← el ÚNICO .html de toda la app
├── package.json         ← dependencias y scripts (dev, build…)
└── vite.config.js       ← config de Vite (no la tocamos por ahora)
```

Fíjate en dos archivos, porque son el puente con el mini-React que construiste.

**`index.html`** — busca esta línea:

```html
<div id="root"></div>
<script type="module" src="/src/main.jsx"></script>
```

¿Te suena ese `<div id="root">`? Es **exactamente** el contenedor que creabas a mano en el mini-React (`document.getElementById("root")`). Toda la app de React se va a dibujar **dentro de ese único div**. Por eso esto se llama una **SPA** (_Single Page Application_): una sola página HTML, y todo lo demás lo pinta JavaScript dentro de ese hueco.

**`src/main.jsx`** — el arranque:

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Léelo recordando el mini-React:

- `document.getElementById("root")` → coges el mismo div de siempre.
- `createRoot(...).render(<App />)` → le dices a React "dibuja `App` aquí dentro". Es tu `render()` del mini-React, pero ahora **lo llama React**, no tú a mano.
- De `StrictMode` no te preocupes por ahora: es una ayuda de desarrollo que avisa de errores comunes.

---

## Paso 4 — Arranca el servidor

```bash
npm run dev
```

Vite te dará una URL, normalmente **http://localhost:5173**. Ábrela en el navegador: verás la página de bienvenida de Vite + React.

Ahora comprueba la magia del HMR: abre `src/App.jsx`, cambia cualquier texto, **guarda**, y mira el navegador. El cambio aparece solo, al instante, sin que recargues nada.

> 💡 El servidor se queda corriendo en esa terminal. Para pararlo: `Ctrl + C`. Para seguir trabajando, déjalo abierto en una terminal y usa otra para el resto de comandos.

---

## Paso 5 — Limpia el scaffold

La plantilla trae un ejemplo (el contador, los logos…) que ahora estorba. Vamos a dejar `App.jsx` en los huesos. Borra todo su contenido y déjalo así:

```jsx
function App() {
  return (
    <div>
      <h1>Mi primera SPA con React</h1>
      <p>Hola, esto ya es React de verdad.</p>
    </div>
  );
}

export default App;
```

Guarda y mira el navegador. Tu página, limpia. **Acabas de leer tu primer componente.** Vamos a entender qué es.

---

## Paso 6 — ¿Qué es un componente?

Mira bien lo que tienes delante:

> Un **componente** de React es una **función** que **devuelve la descripción de un trozo de interfaz**.

Eso es todo. `App` es una función normal de JavaScript que `return`ea algo que parece HTML. Compáralo con el `render()` del mini-React:

- Antes: una función que hacía `root.innerHTML = "..."` (tocaba el DOM a mano).
- Ahora: una función que **devuelve** cómo debe verse, y React se encarga de pintarlo (y de actualizar solo lo mínimo, gracias al Virtual DOM).

Tú **describes**, React **pinta**. Esa es la diferencia que ya sufriste en tus carnes.

Dos reglas de los componentes, para que no te pillen:

1. **El nombre empieza por mayúscula** (`App`, `Saludo`). Con minúscula React cree que es una etiqueta HTML normal.
2. Se **exporta** (`export default App`) para poder usarlo desde otro archivo.

---

## Paso 7 — Eso que devuelve no es HTML, es JSX

Lo que hay dentro del `return` se llama **JSX**. _Parece_ HTML, pero es JavaScript disfrazado, y tiene tres trampas que conviene conocer ya:

**1. Un solo elemento raíz.** El `return` solo puede devolver **un** elemento. Esto **peta**:

```jsx
return (
  <h1>Hola</h1>
  <p>Adiós</p>   // ❌ dos elementos sueltos
);
```

Se envuelve en un `<div>`, o en un _fragment_ vacío `<>...</>` si no quieres un div de más:

```jsx
return (
  <>
    <h1>Hola</h1>
    <p>Adiós</p>
  </>
);
```

**2. JavaScript dentro, con `{ }`.** Aquí está lo bueno. Dentro del JSX, las llaves `{}` te dejan meter **cualquier expresión** de JavaScript. Es como los `${...}` de las template literals que ya usas:

```jsx
function App() {
  const nombre = "Fran";
  const hora = new Date().getHours();

  return (
    <div>
      <h1>Hola, {nombre}</h1>
      <p>Son las {hora}h.</p>
      <p>2 + 2 = {2 + 2}</p>
    </div>
  );
}
```

Pruébalo. Ese `{nombre}` es la semilla de `UI = f(estado)`: la interfaz se construye a partir de datos. Ahora son constantes; muy pronto serán estado que cambia.

**3. Algunos atributos cambian de nombre.** Como JSX es JavaScript, `class` (palabra reservada) se escribe **`className`**:

```jsx
<p className="aviso">Ojo con esto</p>
```

---

## Paso 8 — Compón: un componente dentro de otro

La gracia de los componentes es que se **reutilizan** y se **combinan**, como las funciones. Crea un archivo nuevo `src/Saludo.jsx`:

```jsx
function Saludo() {
  return <p>¡Bienvenido al módulo de frontend! 👋</p>;
}

export default Saludo;
```

Y úsalo dentro de `App.jsx`. Lo importas arriba y lo escribes como si fuera una etiqueta más:

```jsx
import Saludo from "./Saludo.jsx";

function App() {
  return (
    <div>
      <h1>Mi primera SPA con React</h1>
      <Saludo />
      <Saludo />
      <Saludo />
    </div>
  );
}

export default App;
```

Guarda y mira: el saludo aparece **tres veces**. Escribiste el componente una vez y lo reutilizas las que quieras. Así se construye una app de React: **componentes pequeños que se ensamblan** en componentes más grandes, todos colgando de `App`, que cuelga del `root`.

> 💡 Esto es justo lo que le faltaba al mini-React: el `render()` era un bloque gigante de `innerHTML`. Aquí partes la interfaz en piezas con nombre, cada una con su responsabilidad, y las combinas.

---

## Si te sobra tiempo

1. **Tu propia tarjeta.** Crea un componente `Tarjeta.jsx` que devuelva un bloque con un título y una descripción (un `<h2>` y un `<p>`). Úsalo varias veces en `App`. _De momento todas las tarjetas dirán lo mismo_ — apunta ya esa molestia: lo arreglaremos pronto con **props**, que es cómo se le pasan datos distintos a cada componente.
2. **Mete lógica en el JSX.** Dentro de un componente, declara `const aprobado = 7 >= 5;` y muestra `{aprobado ? "Aprobado ✅" : "Suspenso ❌"}` en pantalla. Acabas de pintar interfaz a partir de un dato calculado: `UI = f(estado)` otra vez.
3. **Rompe a propósito.** Quita el `export default` de `Saludo.jsx` y mira el error que da Vite. Vuelve a ponerlo. Conocer los errores típicos ahora te ahorra horas después.
