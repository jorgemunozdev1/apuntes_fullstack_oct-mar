# `useEffect` y fetch a APIs

Hoy juntas dos mundos que ya conoces por separado. Por un lado, el `fetch` + `async/await` que dominaste en M1. Por otro, el `useState` y el renderizado condicional de ayer. La pregunta del día es: **¿dónde pongo el `fetch` dentro de un componente de React?** Y la respuesta es un hook nuevo: `useEffect`.

Al final de la clase tu app pedirá datos a una API real y los pintará en pantalla, gestionando los tres estados de toda petición: **cargando, error y datos**.

---

## Repaso: los tres estados, sin fetch todavía

Antes de tocar nada nuevo, calienta con lo de ayer. Crea un componente que tenga **tres estados** con `useState`: `cargando` (booleano), `error` (string o `null`) y `datos` (string o `null`). Pon tres botones que simulen cada situación cambiando esos estados a mano, y pinta con **renderizado condicional**:

- Si `cargando` es `true` → "Cargando…"
- Si hay `error` → el mensaje de error
- Si hay `datos` → los datos

```jsx
import { useState } from "react";

function Demo() {
  const [cargando, setCargando] = useState(false);
  const [error, setError] = useState(null);
  const [datos, setDatos] = useState(null);

  if (cargando) return <p>Cargando…</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <div>
      <button onClick={() => setCargando(true)}>Simular carga</button>
      <button onClick={() => setError("Algo falló")}>Simular error</button>
      <button onClick={() => setDatos("¡Datos listos!")}>Simular datos</button>
      {datos && <p>{datos}</p>}
    </div>
  );
}
```

Quédate con este esqueleto. **Hoy solo vas a cambiar una cosa: en vez de pulsar botones, será una API quien rellene esos estados.**

---

## `useEffect`: dónde van las cosas que no son "calcular JSX"

Recuerda cómo funciona un componente: es una **función que React ejecuta en cada render** para calcular el JSX a partir de sus props y su estado. Eso es lo único que debería hacer el cuerpo del componente: calcular y devolver JSX. A eso lo llamamos ser **puro**.

Pedir datos a una API **no** es calcular JSX. Es un _efecto secundario_: algo que pasa "hacia fuera" (una petición de red, un temporizador, tocar el DOM, leer `localStorage`…). Y los efectos secundarios **no van en el cuerpo del componente**.

¿Por qué no? Mira la trampa:

```jsx
function Personaje() {
  const [personaje, setPersonaje] = useState(null);

  // ❌ NUNCA hagas esto
  fetch("https://rickandmortyapi.com/api/character/1")
    .then((res) => res.json())
    .then((data) => setPersonaje(data));

  return <h2>{personaje?.name}</h2>;
}
```

`setPersonaje` provoca un render → el componente se ejecuta otra vez → vuelve a lanzar el `fetch` → otro `setPersonaje` → otro render… **bucle infinito de peticiones.** Tu pestaña echando humo.

`useEffect` resuelve esto: es el sitio donde metes los efectos secundarios y le dices a React **cuándo** ejecutarlos.

```jsx
import { useState, useEffect } from "react";

function Personaje() {
  const [personaje, setPersonaje] = useState(null);

  useEffect(() => {
    // este código corre DESPUÉS del render, no durante
    console.log("efecto ejecutado");
  }, []);

  return <h2>{personaje?.name}</h2>;
}
```

Anatomía:

```jsx
useEffect(() => {
  // 1. la función con el efecto
}, []);
//  ↑ 2. el array de dependencias
```

El **array de dependencias** es la pieza clave. Controla cada cuánto se ejecuta el efecto:

| Array de dependencias | Cuándo se ejecuta el efecto                       |
| --------------------- | ------------------------------------------------- |
| `[]` (vacío)          | **Una sola vez**, al montar el componente         |
| `[pagina]`            | Al montar **y** cada vez que `pagina` cambie      |
| _(sin array)_         | **En cada render** → casi siempre un bug, cuidado |

Para "pedir datos al cargar", lo que quieres es `[]`: una vez, al aparecer el componente.

---

## Práctica guiada 1: tu primer fetch en React

Vamos a pedir **un** personaje a la [Rick and Morty API](https://rickandmortyapi.com/) y pintarlo. Crea `src/components/Personaje.jsx`.

> Si prefieres seguir con la PokeAPI que ya usasteis en M1, adelante: el patrón es idéntico. Es buen momento para comparar cómo lo hacías a pelo con `fetch` en vanilla y cómo se organiza ahora en React.

```jsx
import { useState, useEffect } from "react";

function Personaje() {
  const [personaje, setPersonaje] = useState(null);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function cargar() {
      try {
        setCargando(true);
        const res = await fetch("https://rickandmortyapi.com/api/character/1");
        if (!res.ok) throw new Error("No se pudo cargar el personaje");
        const data = await res.json();
        setPersonaje(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setCargando(false);
      }
    }

    cargar();
  }, []);

  if (cargando) return <p>Cargando…</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <div>
      <h2>{personaje.name}</h2>
      <img src={personaje.image} alt={personaje.name} width="200" />
      <p>Estado: {personaje.status}</p>
    </div>
  );
}

export default Personaje;
```

Fíjate en tres cosas importantes.

**1. No puedes hacer el callback de `useEffect` directamente `async`.**

```jsx
useEffect(async () => { ... }, []);   // ❌ React se queja
```

Un efecto puede devolver una función de limpieza, y una función `async` siempre devuelve una promesa, así que React se lía. La solución es la de arriba: **defines una función `async` dentro y la llamas**.

**2. La trampa del `fetch` que ya conoces de M1 sigue viva.** `fetch` **no lanza error** si la respuesta es 404 o 500; solo falla si no hay red. Por eso compruebas `if (!res.ok) throw new Error(...)` a mano. Sin esa línea, un 404 te llegaría como "datos" y reventaría al pintar.

**3. El `try / catch / finally` reparte el trabajo:** el `try` con el camino feliz, el `catch` guardando el error en el estado, y el `finally` apagando el "cargando" pase lo que pase.

Pruébalo. Verás el "Cargando…" un instante y luego el personaje. Y ahora el detalle bonito: **tú no has tocado el DOM ni una vez.** Cambias estado con los `set...`, React repinta. El mismo principio del martes: UI = f(estado).

Commit:

```bash
git add .
git commit -m "feat: primer fetch a API con useEffect y estados de carga/error"
```

---

## El array de dependencias, en serio

El `[]` vacío significa "una vez y ya". Pero muchas veces quieres **volver a pedir datos cuando algo cambia**: el usuario pasa de página, escribe una búsqueda, selecciona una categoría…

Ahí metes ese "algo" en el array de dependencias:

```jsx
useEffect(() => {
  // se ejecuta al montar y cada vez que `pagina` cambie
  cargarPersonajes();
}, [pagina]);
```

La regla mental: **dentro del array va todo valor del que dependa el efecto.** Si el efecto usa `pagina`, `pagina` va en el array. Así React sabe que, cuando cambie, debe volver a ejecutarlo.

> Más adelante el linter te avisará si te dejas dependencias fuera. De momento quédate con la idea: `[]` = una vez; `[algo]` = cuando `algo` cambie.

---

## Práctica guiada 2: lista con paginación

Ahora una lista de personajes que se recarga al cambiar de página. Crea `src/components/ListaPersonajes.jsx`. Aquí juntas todo: estado, efecto con dependencia, eventos, renderizado condicional y pintar una lista con `.map` (lo de ayer).

```jsx
import { useState, useEffect } from "react";

function ListaPersonajes() {
  const [personajes, setPersonajes] = useState([]);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState(null);
  const [pagina, setPagina] = useState(1);

  useEffect(() => {
    async function cargar() {
      try {
        setCargando(true);
        setError(null);
        const res = await fetch(
          `https://rickandmortyapi.com/api/character?page=${pagina}`,
        );
        if (!res.ok) throw new Error("Error al cargar la lista");
        const data = await res.json();
        setPersonajes(data.results);
      } catch (err) {
        setError(err.message);
      } finally {
        setCargando(false);
      }
    }

    cargar();
  }, [pagina]);

  return (
    <div>
      <div>
        <button onClick={() => setPagina(pagina - 1)} disabled={pagina === 1}>
          Anterior
        </button>
        <span> Página {pagina} </span>
        <button onClick={() => setPagina(pagina + 1)}>Siguiente</button>
      </div>

      {cargando && <p>Cargando…</p>}
      {error && <p>Error: {error}</p>}

      {!cargando && !error && (
        <ul>
          {personajes.map((p) => (
            <li key={p.id}>
              {p.name} — {p.status}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

export default ListaPersonajes;
```

Móntalo y pulsa "Siguiente". Cada vez que cambias `pagina`, el efecto se vuelve a ejecutar (porque `pagina` está en el array de dependencias) y trae datos nuevos. **Ese es el array de dependencias trabajando de verdad**, no en teoría.

Repasa los tres estados pintándose en condicional, igual que el calentamiento de esta mañana, pero ahora rellenados por la API en vez de por botones.

Commit:

```bash
git add .
git commit -m "feat: lista paginada con refetch al cambiar de página"
```

---

## Tu proyecto

Toca traer **datos reales** a tu SPA. El objetivo de hoy: que tu app pida algo a una API (la que uses en tu proyecto) dentro de un `useEffect` y lo pinte gestionando los tres estados.

- Monta el patrón cargando / error / datos como en clase.
- Si tu proyecto consume tu **propio** backend de M2, todavía no: la conexión real (con login y token) es el lunes 29. Hoy practica con una API pública para tener el patrón dominado.
- Asegúrate de meter el `if (!res.ok)` y de pintar un mensaje de error decente. No lo dejes para "luego".

Cuando lo tengas:

```bash
git add .
git commit -m "feat: carga de datos reales desde API en la SPA"
git push
```

Y actualiza el tablero: mueve a **Hecho** las tarjetas de fetch / carga de datos, y deja preparadas en **Por hacer** las de enrutado (el lunes empezamos con React Router).

---

## Reto opcional

Si has terminado y quieres ir más allá:

1. **Buscador.** Añade un input con su `useState` (`busqueda`) y usa el endpoint `?name=${busqueda}`. Mete `busqueda` en el array de dependencias para que recargue al escribir. Aviso: así dispara una petición por cada tecla; piensa cómo lo arreglarías (la palabra mágica es _debounce_, lo investigas).
2. **Función de limpieza.** `useEffect` puede devolver una función que React ejecuta al desmontar el componente o antes de volver a lanzar el efecto. Pruébalo con un temporizador:
   ```jsx
   useEffect(() => {
     const id = setInterval(() => console.log("tic"), 1000);
     return () => clearInterval(id); // limpieza
   }, []);
   ```
   Quita el `return` y mira cómo se acumulan los "tic". Eso es una _fuga_: por eso existe la limpieza.
3. **Evitar el aviso del race condition.** Si cambias de página muy rápido, puede llegar antes la respuesta de una página vieja. Investiga cómo se ignora una respuesta obsoleta usando una bandera dentro del efecto (`let activo = true; ... return () => { activo = false; }`).
