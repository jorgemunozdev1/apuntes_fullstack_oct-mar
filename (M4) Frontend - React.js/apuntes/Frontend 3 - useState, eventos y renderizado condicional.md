# `useState`, eventos y renderizado condicional

Hoy es **el día**. ¿Te acuerdas del martes, cuando construiste tu propio React a mano? Tenías un objeto `estado` y una función `render()` que repintaba toda la app cada vez que algo cambiaba, y tú eras quien llamaba a `render()`.

`useState` es exactamente eso, pero **React llama a `render()` por ti**. Cuando cambias el estado, React vuelve a ejecutar tu componente y actualiza la pantalla sola. Tú ya no vuelves a tocar el DOM. Hoy le das memoria a tus componentes, aprendes a reaccionar a clics y a mostrar cosas distintas según ese estado.

---

## Repaso: props

Antes de empezar con lo nuevo, calienta con lo de ayer. Tienes este componente:

```jsx
function Saludo(props) {
  return <h2>Hola, {props.nombre}</h2>;
}
```

1. Renderiza tres `<Saludo />` con nombres distintos.
2. Cambia el componente para que use **destructuring** en los parámetros: `function Saludo({ nombre })`.
3. Pregunta rápida para ti: si dentro del componente haces `nombre = "otro"`, ¿se actualiza algo en pantalla? ¿Puede un componente cambiar sus propias props?

> La respuesta a la 3 es **no**: las props son de solo lectura, llegan desde el padre. Si un componente necesita datos que cambian _por sí mismo_… eso es justo lo que vamos a ver ahora.

---

## `useState`: el estado que React repinta por ti

Imagina un contador. Lo intuitivo sería:

```jsx
function Contador() {
  let cuenta = 0;

  function sumar() {
    cuenta = cuenta + 1;
    console.log(cuenta); // sube en consola...
  }

  return <button onClick={sumar}>Cuenta: {cuenta}</button>;
}
```

Pruébalo. **En consola la cuenta sube, pero en pantalla siempre pone 0.** ¿Por qué? Porque React solo repinta cuando _sabe_ que algo cambió, y una variable normal le pasa totalmente desapercibida. Es como en tu React de juguete: si cambiabas el objeto `estado` pero no llamabas a `render()`, la pantalla se quedaba igual.

`useState` resuelve las dos cosas a la vez: **guarda un valor que sobrevive entre renders** y **te da una función que, al usarla, dispara el repintado** (ese `render()` que antes llamabas tú).

```jsx
import { useState } from "react";

function Contador() {
  const [cuenta, setCuenta] = useState(0);

  function sumar() {
    setCuenta(cuenta + 1);
  }

  return <button onClick={sumar}>Cuenta: {cuenta}</button>;
}
```

Desglosemos la línea clave:

```jsx
const [cuenta, setCuenta] = useState(0);
//     ↑ valor   ↑ función     ↑ valor inicial
```

- `useState(0)` arranca el estado en `0`.
- Te devuelve un array de dos cosas, que recoges con **destructuring** (lo de ayer, otra vez): el **valor actual** y una **función para cambiarlo**.
- Por convención la función se llama `set` + nombre: `cuenta` → `setCuenta`.

La regla de oro:

> **Nunca cambies el valor a mano (`cuenta = ...`). Siempre con su función `setCuenta(...)`.** Esa función es la que avisa a React de que repinte. Es tu `render()` automático.

Dos detalles para que no te pillen:

- `useState` siempre se llama **arriba del todo del componente**, nunca dentro de un `if`, un bucle o una función anidada.
- Cuando llamas a `setCuenta`, React **no** cambia el valor en esa misma línea; programa un nuevo render y en el _siguiente_ render `cuenta` ya tiene el valor nuevo. Por eso si haces `console.log(cuenta)` justo después de `setCuenta`, todavía ves el valor viejo. Es normal.

---

## Eventos en React

Ya has usado `onClick`. Repasemos las reglas porque aquí hay dos trampas clásicas.

```jsx
<button onClick={sumar}>Sumar</button>
```

- En React los eventos van en **camelCase**: `onClick`, `onChange`, `onSubmit`… (no `onclick`).
- Le pasas **la función, sin paréntesis**.

### Trampa 1: los paréntesis

```jsx
<button onClick={sumar}>   ✅ le pasas la función, React la llama al hacer clic
<button onClick={sumar()}> ❌ la EJECUTAS al renderizar, no al hacer clic
```

Si pones `sumar()`, la función se ejecuta una vez al pintar el componente y nunca más. Síntoma típico: "se ejecuta solo al cargar" o un bucle infinito de renders.

### Trampa 2: pasar argumentos

¿Y si necesitas pasarle un dato al manejador? No puedes poner `onClick={borrar(id)}` (eso es la trampa 1). Lo envuelves en una arrow:

```jsx
<button onClick={() => borrar(id)}>Borrar</button>
```

Así le pasas una función nueva que, _cuando se haga clic_, llamará a `borrar(id)`.

Eventos más usados: `onClick` (botones), `onChange` (inputs), `onSubmit` (formularios). El `onChange` lo exprimirás en la clase de formularios; hoy con `onClick` y algún `onChange` suelto vas sobrado.

---

## Renderizado condicional

"Mostrar una cosa u otra según el estado." Tienes tres formas, de menos a más potente.

### 1. Operador `&&` (mostrar algo o nada)

```jsx
{
  hayError && <p className="error">Algo ha ido mal</p>;
}
```

Se lee: "si `hayError` es verdadero, pinta el `<p>`; si no, no pintes nada".

> **Trampa importante.** `&&` solo es seguro con booleanos. Con un número se te cuela en pantalla:
>
> ```jsx
> {
>   lista.length && <Lista />;
> } // ❌ si length es 0, ¡pinta un 0 en la pantalla!
> ```
>
> React pinta los números, y `0` es número. Solución: convierte a booleano de verdad.
>
> ```jsx
> {
>   lista.length > 0 && <Lista />;
> } // ✅
> ```

### 2. Ternario (mostrar una cosa **u** otra)

```jsx
{
  estaLogueado ? (
    <button>Cerrar sesión</button>
  ) : (
    <button>Iniciar sesión</button>
  );
}
```

### 3. Variable o `return` temprano (para casos largos)

Cuando la lógica crece, no metas un ternario gigante en el JSX. Sácalo antes del `return`:

```jsx
function Perfil({ usuario }) {
  if (!usuario) {
    return <p>Cargando…</p>;
  }

  return <h1>Hola, {usuario.nombre}</h1>;
}
```

---

## Práctica guiada 1: Contador y Toggle

Crea un componente `Contador` en `src/components/Contador.jsx` y monta esto paso a paso.

**Paso 1 — Contador básico.** Estado `cuenta` empezando en `0` y tres botones: `+`, `−` y `Reset`.

```jsx
import { useState } from "react";

function Contador() {
  const [cuenta, setCuenta] = useState(0);

  return (
    <div>
      <h2>Cuenta: {cuenta}</h2>
      <button onClick={() => setCuenta(cuenta + 1)}>+</button>
      <button onClick={() => setCuenta(cuenta - 1)}>−</button>
      <button onClick={() => setCuenta(0)}>Reset</button>
    </div>
  );
}

export default Contador;
```

Móntalo en `App.jsx`, abre el navegador y comprueba que los tres botones funcionan.

**Paso 2 — Renderizado condicional.** Debajo de la cuenta, muestra un mensaje **solo cuando la cuenta sea negativa**: "⚠️ Estás en negativo". Pista: `&&` (cuidado con la trampa del `0`, aquí no aplica porque comparas, pero acostúmbrate a comparar).

**Paso 3 — Toggle.** En el mismo componente añade un segundo estado, `visible`, que empiece en `true`, y un botón que lo cambie:

```jsx
const [visible, setVisible] = useState(true);
// ...
<button onClick={() => setVisible(!visible)}>
  {visible ? "Ocultar" : "Mostrar"} contador
</button>;

{
  visible && <h2>Cuenta: {cuenta}</h2>;
}
```

Fíjate en el detalle: un mismo componente puede tener **varios `useState`**, uno por cada cosa que cambia de forma independiente. Eso es justo lo que sufrías el martes cuando todo estaba en un único objeto `estado` y tenías que repintarlo entero a mano.

Commit:

```bash
git add .
git commit -m "feat: componente Contador con estado, toggle y render condicional"
```

---

## Práctica guiada 2: Lista filtrable

Aquí juntas todo: estado, eventos, condicional y pintar una lista. Crea `src/components/ListaTareas.jsx`.

**Paso 1 — Pintar la lista.** Parte de un array de datos y píntalo con `.map` (sí, el `.map` de M1, el mismo). En React, cada elemento de una lista necesita una prop especial **`key`** única, para que React sepa cuál es cuál al repintar:

```jsx
import { useState } from "react";

const TAREAS = [
  { id: 1, texto: "Estudiar useState", hecha: true },
  { id: 2, texto: "Repasar props", hecha: true },
  { id: 3, texto: "Montar la lista filtrable", hecha: false },
  { id: 4, texto: "Desplegar el proyecto", hecha: false },
];

function ListaTareas() {
  return (
    <ul>
      {TAREAS.map((tarea) => (
        <li key={tarea.id}>
          {tarea.texto} {tarea.hecha ? "✅" : "⬜"}
        </li>
      ))}
    </ul>
  );
}

export default ListaTareas;
```

> El `key` va siempre en el elemento de más fuera del `.map`, y debe ser **único y estable** (el `id`, nunca el índice del array si puedes evitarlo).

**Paso 2 — Estado del filtro.** Añade un estado `filtro` que pueda valer `"todas"`, `"hechas"` o `"pendientes"`, y tres botones que lo cambien:

```jsx
const [filtro, setFiltro] = useState("todas");
```

```jsx
<div>
  <button onClick={() => setFiltro("todas")}>Todas</button>
  <button onClick={() => setFiltro("hechas")}>Hechas</button>
  <button onClick={() => setFiltro("pendientes")}>Pendientes</button>
</div>
```

**Paso 3 — Filtrar antes de pintar.** Justo antes del `return`, calcula qué tareas mostrar según el filtro (otra vez tus métodos de array de M1):

```jsx
const tareasVisibles = TAREAS.filter((tarea) => {
  if (filtro === "hechas") return tarea.hecha;
  if (filtro === "pendientes") return !tarea.hecha;
  return true; // "todas"
});
```

Y pinta `tareasVisibles` en lugar de `TAREAS`.

**Paso 4 — Caso vacío (render condicional).** Si después de filtrar no queda ninguna tarea, muestra un mensaje en vez de una lista vacía:

```jsx
{
  tareasVisibles.length > 0 ? (
    <ul>
      {tareasVisibles.map((tarea) => (
        <li key={tarea.id}>
          {tarea.texto} {tarea.hecha ? "✅" : "⬜"}
        </li>
      ))}
    </ul>
  ) : (
    <p>No hay tareas en este filtro 🎉</p>
  );
}
```

Pruébalo: cambia de filtro y mira cómo la lista se actualiza sola. **Tú nunca has tocado el DOM.** Cambias `filtro` con `setFiltro`, React repinta. UI = f(estado), como en tu React de juguete.

Commit:

```bash
git add .
git commit -m "feat: lista filtrable con estado de filtro y caso vacío"
```

---

## Tu proyecto

Lleva lo de hoy a tu SPA. El objetivo es que **algo de tu proyecto reaccione al usuario con estado local**:

- Si tu app muestra una colección de cosas, añade un filtro o una búsqueda con `useState` (como la lista filtrable).
- Si no, mete algo interactivo más simple: un botón de "mostrar más / mostrar menos", un contador, un toggle de algún detalle.

Lo importante hoy no es la feature definitiva, sino que practiques el ciclo **estado → evento → repintado** sobre tu propio código.

Cuando lo tengas:

```bash
git add .
git commit -m "feat: estado local interactivo en la SPA"
git push
```

Y actualiza tu tablero: mueve a **Hecho** las tarjetas de estado local / interactividad, y crea las tarjetas de lo que viene (rutas, fetch a API) en **Por hacer**.

---

## Reto opcional

Si has terminado y quieres rizar el rizo:

1. **Botón de doble salto.** Crea un botón que sume 2 de golpe con dos llamadas seguidas: `setCuenta(cuenta + 1); setCuenta(cuenta + 1);`. Verás que **solo sube 1**. Investiga por qué y arréglalo usando la forma de función: `setCuenta((c) => c + 1)`. Esto te ahorrará bugs en el futuro.
2. **Añadir tareas.** Convierte el array `TAREAS` en estado (`useState`) y añade un input + botón para crear nuevas tareas. Pista: para añadir sin mutar, `setTareas([...tareas, nuevaTarea])`.
3. **Marcar como hecha.** Un clic en una tarea cambia su `hecha`. Pista: `setTareas(tareas.map(t => t.id === id ? { ...t, hecha: !t.hecha } : t))`.

> Si tocas el reto 2 o 3, fíjate en una regla que veremos mucho: **nunca modifiques el array o el objeto del estado directamente** (`tareas.push(...)` ❌). Siempre creas uno nuevo con spread (`[...tareas]`, `{ ...tarea }`) y se lo pasas a la función `set`.
