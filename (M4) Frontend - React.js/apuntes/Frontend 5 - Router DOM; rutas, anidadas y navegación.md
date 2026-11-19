# React Router DOM: rutas, anidadas y navegación

Hasta ahora tu app es **una sola pantalla**. Cambias estado, se repinta, pero siempre es "la misma página". Hoy le das **páginas de verdad**: inicio, listado, detalle, "acerca de"… cada una con su propia URL, y todo **sin recargar el navegador**.

En M2 ya montaste rutas, ¿te acuerdas? Pero allí las rutas vivían en el **servidor** (Express decidía qué responder según la URL). Hoy las rutas viven en el **navegador**: React Router intercepta los cambios de URL y cambia qué componente se muestra, sin pedirle nada al servidor. Eso es lo que hace que una SPA se sienta instantánea.

---

## Repaso: fetch de un recurso

Calienta con lo del viernes. Coge tu componente de detalle de un personaje, pero esta vez haz que el `id` sea una **variable** arriba del componente, no un número fijo en la URL del fetch:

```jsx
import { useState, useEffect } from "react";

function Personaje() {
  const id = 2; // de momento a mano
  const [personaje, setPersonaje] = useState(null);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function cargar() {
      try {
        setCargando(true);
        const res = await fetch(
          `https://rickandmortyapi.com/api/character/${id}`,
        );
        if (!res.ok) throw new Error("No se pudo cargar");
        setPersonaje(await res.json());
      } catch (err) {
        setError(err.message);
      } finally {
        setCargando(false);
      }
    }
    cargar();
  }, [id]);

  if (cargando) return <p>Cargando…</p>;
  if (error) return <p>Error: {error}</p>;
  return <h2>{personaje.name}</h2>;
}
```

Guárdalo en la cabeza: **ese `id` que ahora pones a mano, al final de la clase vendrá de la URL.** Y como ya está en el array de dependencias (`[id]`), el día que cambie, el fetch se relanzará solo.

---

## Montar React Router

Instala la librería:

```bash
npm install react-router-dom
```

Son tres conceptos y ya tienes routing funcionando.

### 1. Envuelve la app en `<BrowserRouter>`

Esto va **una sola vez**, arriba del todo, en `src/main.jsx`:

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>,
);
```

`BrowserRouter` es quien "escucha" la URL del navegador. Sin él, nada de lo demás funciona.

### 2. Define las rutas con `<Routes>` y `<Route>`

En `App.jsx`, cada `<Route>` dice "para esta URL, muestra este componente":

```jsx
import { Routes, Route } from "react-router-dom";
import Home from "./pages/Home.jsx";
import About from "./pages/About.jsx";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
    </Routes>
  );
}
```

> Convención útil: las páginas completas van en una carpeta `src/pages/` y los trozos reutilizables en `src/components/`. No es obligatorio, pero te ordena la cabeza.

### 3. Navega con `<Link>`, nunca con `<a>`

```jsx
import { Link } from "react-router-dom";

<nav>
  <Link to="/">Inicio</Link>
  <Link to="/about">Acerca de</Link>
</nav>;
```

> **Trampa importante.** Si usas `<a href="/about">` en vez de `<Link to="/about">`, el navegador **recarga la página entera**: pierdes todo el estado y rompes la magia de la SPA. Dentro de tu app, siempre `<Link>`. El `<a>` se reserva para enlaces a sitios externos.

---

## Práctica guiada 1: app multipágina

Crea tres páginas en `src/pages/`: `Home.jsx`, `About.jsx` y una `Personajes.jsx` (reutiliza tu lista del viernes). Cada una, de momento, puede ser tan simple como un `<h1>`.

Monta la navegación y las rutas:

```jsx
import { Routes, Route, Link } from "react-router-dom";
import Home from "./pages/Home.jsx";
import About from "./pages/About.jsx";
import Personajes from "./pages/Personajes.jsx";

function App() {
  return (
    <div>
      <nav>
        <Link to="/">Inicio</Link> | <Link to="/personajes">Personajes</Link> |{" "}
        <Link to="/about">Acerca de</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/personajes" element={<Personajes />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </div>
  );
}

export default App;
```

Arranca, pulsa los enlaces y mira la barra de direcciones: la URL cambia, el contenido cambia, **el navegador no recarga**. Eso es client-side routing.

Commit:

```bash
git add .
git commit -m "feat: routing básico con tres páginas y navegación"
```

---

## Rutas dinámicas: `:id` + `useParams`

Aquí es donde routing y el fetch del viernes se dan la mano. Una ruta puede tener un **parámetro** marcado con dos puntos:

```jsx
<Route path="/personajes/:id" element={<DetallePersonaje />} />
```

Ese `:id` es un comodín: `/personajes/1`, `/personajes/42`… todas entran por esa ruta. ¿Cómo lees el valor desde dentro del componente? Con el hook **`useParams`**:

```jsx
import { useParams } from "react-router-dom";

function DetallePersonaje() {
  const { id } = useParams(); // el :id de la URL
  // ...usas `id` en tu fetch
}
```

¿Te suena? Es **exactamente** el componente del calentamiento, pero ahora el `id` no lo pones a mano: **lo trae la URL**. Y como `id` está en el array de dependencias del `useEffect`, cuando navegues a otro personaje, el fetch se relanza solo.

> Ojo: `useParams` siempre devuelve **strings**. `id` será `"1"`, no `1`. Para una URL de fetch da igual, pero tenlo presente si alguna vez lo comparas con un número.

---

## Navegación: `NavLink` y `useNavigate`

Dos herramientas más para moverte.

**`NavLink`** es como `Link` pero sabe si su ruta está activa, ideal para resaltar el enlace de la página actual:

```jsx
import { NavLink } from "react-router-dom";

<NavLink
  to="/personajes"
  className={({ isActive }) => (isActive ? "activo" : "")}
>
  Personajes
</NavLink>;
```

**`useNavigate`** navega desde código (no desde un clic en un enlace): útil tras enviar un formulario, o para un botón "volver":

```jsx
import { useNavigate } from "react-router-dom";

function BotonVolver() {
  const navigate = useNavigate();
  return <button onClick={() => navigate(-1)}>← Volver</button>;
}
```

`navigate("/personajes")` te lleva a una ruta; `navigate(-1)` es "atrás", como la flecha del navegador. Esto lo usarás mucho el día del login: "tras iniciar sesión, llévame al dashboard".

---

## Práctica guiada 2: de la lista al detalle

El patrón estrella de cualquier app: una lista donde cada elemento enlaza a su página de detalle.

**Paso 1 — Enlaza cada elemento de la lista.** En tu página `Personajes`, envuelve cada nombre en un `<Link>` a su detalle:

```jsx
import { Link } from "react-router-dom";

<ul>
  {personajes.map((p) => (
    <li key={p.id}>
      <Link to={`/personajes/${p.id}`}>{p.name}</Link>
    </li>
  ))}
</ul>;
```

**Paso 2 — Crea la página de detalle** en `src/pages/DetallePersonaje.jsx`, leyendo el `id` con `useParams` y haciendo el fetch (el del viernes, calcadito):

```jsx
import { useState, useEffect } from "react";
import { useParams } from "react-router-dom";

function DetallePersonaje() {
  const { id } = useParams();
  const [personaje, setPersonaje] = useState(null);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function cargar() {
      try {
        setCargando(true);
        const res = await fetch(
          `https://rickandmortyapi.com/api/character/${id}`,
        );
        if (!res.ok) throw new Error("Personaje no encontrado");
        setPersonaje(await res.json());
      } catch (err) {
        setError(err.message);
      } finally {
        setCargando(false);
      }
    }
    cargar();
  }, [id]);

  if (cargando) return <p>Cargando…</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <div>
      <h2>{personaje.name}</h2>
      <img src={personaje.image} alt={personaje.name} width="200" />
      <p>Estado: {personaje.status}</p>
      <p>Especie: {personaje.species}</p>
    </div>
  );
}

export default DetallePersonaje;
```

**Paso 3 — Registra la ruta dinámica** en `App.jsx`:

```jsx
<Route path="/personajes/:id" element={<DetallePersonaje />} />
```

Pruébalo: lista → clic en un personaje → su detalle con la URL `/personajes/3`. Cambia el número en la barra de direcciones a mano: el detalle se recarga solo, porque `id` cambió y el `useEffect` reacciona. **Ahí tienes el día 39 y el 40 trabajando juntos.**

Commit:

```bash
git add .
git commit -m "feat: navegación lista-detalle con ruta dinámica y useParams"
```

---

## Rutas anidadas: un layout compartido con `<Outlet>`

Te habrás fijado en que el `<nav>` se repite y que todas las páginas comparten la misma cabecera. En vez de copiarlo, creas un **layout** que envuelve a las páginas hijas. El sitio donde React Router "inserta" la página hija se marca con `<Outlet>`.

`src/pages/Layout.jsx`:

```jsx
import { NavLink, Outlet } from "react-router-dom";

function Layout() {
  return (
    <div>
      <nav>
        <NavLink to="/">Inicio</NavLink> |{" "}
        <NavLink to="/personajes">Personajes</NavLink> |{" "}
        <NavLink to="/about">Acerca de</NavLink>
      </nav>

      <main>
        <Outlet /> {/* aquí se pinta la ruta hija */}
      </main>
    </div>
  );
}

export default Layout;
```

Y en `App.jsx`, anidas las rutas dentro del `Layout`:

```jsx
<Routes>
  <Route path="/" element={<Layout />}>
    <Route index element={<Home />} />
    <Route path="personajes" element={<Personajes />} />
    <Route path="personajes/:id" element={<DetallePersonaje />} />
    <Route path="about" element={<About />} />
  </Route>
</Routes>
```

Dos detalles:

- La ruta padre lleva el `element={<Layout />}`; las hijas van **dentro** y se pintan en el `<Outlet>`.
- `index` es la ruta por defecto del padre: lo que se muestra en `/` (tu Home). Y fíjate que las hijas ya **no llevan `/` inicial** (`path="personajes"`, no `"/personajes"`), porque cuelgan del padre.

Ahora el `<nav>` vive en un único sitio y todas las páginas lo heredan. Esto es la base del layout que tendrá tu proyecto.

Commit:

```bash
git add .
git commit -m "feat: layout con rutas anidadas y Outlet"
```

---

## Tu proyecto

Define el **mapa de rutas** de tu SPA. El objetivo de hoy: que tu app tenga al menos un layout con navegación y tres o cuatro rutas reales, incluida una **ruta de detalle dinámica** si tu proyecto la necesita.

- Monta el `Layout` con `<Outlet>` y mueve tu navegación ahí.
- Define las páginas principales (inicio, listado, detalle, perfil… lo que pida tu idea).
- Deja preparada (aunque sea vacía) la ruta que más adelante será **privada**: el lunes 29 la protegeremos con el login. Hoy solo que exista.

```bash
git add .
git commit -m "feat: estructura de rutas de la SPA"
git push
```

Tablero: mueve a **Hecho** las tarjetas de enrutado y navegación, y crea en **Por hacer** las de estado global (mañana entramos con `useRef` y `useContext`).

---

## Reto opcional

1. **Página 404.** Añade una ruta comodín al final que capture cualquier URL no definida:
   ```jsx
   <Route path="*" element={<NoEncontrada />} />
   ```
   Crea la página `NoEncontrada` con un mensaje y un `<Link>` para volver al inicio.
2. **Botón "Volver" en el detalle.** Usa `useNavigate` con `navigate(-1)` para volver a la lista desde la página de detalle.
3. **Resaltado activo.** Da estilo al enlace de la página actual con `NavLink` y la clase `activo` (define el CSS tú). Bonus: usa la forma con función (`style={({ isActive }) => ...}`) en vez de `className`.
4. **Parámetros de búsqueda.** Investiga el hook `useSearchParams` para leer y escribir cosas como `/personajes?page=2` en la URL. Te servirá para que la paginación del viernes sea compartible por enlace.
