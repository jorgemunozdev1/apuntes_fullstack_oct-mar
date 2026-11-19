# `useRef` y `useContext`: referencias y estado compartido

Hasta ahora tienes dos formas de guardar información en un componente: las **props** (vienen de fuera, de solo lectura) y el **estado** (`useState`, cambia y repinta). Hoy añades dos piezas que cubren los huecos que dejan esas dos.

- A veces quieres guardar un valor **entre renders pero sin repintar** cuando cambia (un temporizador, el render anterior), o necesitas **tocar un elemento del DOM directamente** (dar foco a un input). Para eso está **`useRef`**.
- A veces un dato lo necesita medio árbol de componentes (el usuario logueado, el tema claro/oscuro) y acabas pasándolo prop a prop a prop por componentes que ni lo usan. Eso es el **prop drilling**, y lo resuelve **`useContext`**.

Las dos cosas las vas a necesitar en tu proyecto: `useRef` para detalles finos de UI, y `useContext` para tener al usuario logueado disponible en toda la app sin arrastrarlo por props. De hecho, el `useContext` de hoy es la base sobre la que el lunes montaremos las **rutas protegidas**.

---

## Repaso: estado que repinta

Calienta recordando lo que ya domina tu cabeza. Este contador funciona porque `useState` guarda el valor **y** dispara el repintado:

```jsx
import { useState } from "react";

function Contador() {
  const [cuenta, setCuenta] = useState(0);
  return <button onClick={() => setCuenta(cuenta + 1)}>Cuenta: {cuenta}</button>;
}
```

Ten esto presente, porque `useRef` es el contraste exacto: también guarda un valor entre renders, pero **no repinta** cuando lo cambias.

---

## `useRef`: una caja que sobrevive a los renders y no repinta

`useRef` te da un objeto con una sola propiedad, `.current`, donde guardas lo que quieras. Tiene dos usos, y conviene no mezclarlos en la cabeza.

### Uso 1 — un valor mutable que persiste pero no repinta

Imagina que quieres contar **cuántas veces se ha renderizado** un componente. Si usas `useState`, cada vez que sumas disparas otro render → bucle. Necesitas un valor que sobreviva entre renders pero que cambiarlo **no** provoque repintado:

```jsx
import { useRef, useState } from "react";

function Demo() {
  const [texto, setTexto] = useState("");
  const renders = useRef(0);

  renders.current = renders.current + 1; // cambia, pero NO repinta

  return (
    <div>
      <input value={texto} onChange={(e) => setTexto(e.target.value)} />
      <p>Renders: {renders.current}</p>
    </div>
  );
}
```

La diferencia con el estado, en una frase:

> **`useState`** = "guarda esto **y repinta** cuando cambie". **`useRef`** = "guárdame esto entre renders, pero **no repintes** cuando lo cambie".

Por eso un ref es perfecto para guardar el `id` de un `setInterval`, el valor anterior de algo, o cualquier dato "de fontanería" que no tiene que aparecer en pantalla.

> ⚠️ Como cambiar un ref no repinta, **no metas en un ref nada que deba verse en pantalla**. Si lo cambias, la pantalla no se entera hasta el próximo render por otro motivo. Para lo que se ve, `useState`. Para lo que no se ve, `useRef`.

### Uso 2 — acceder a un elemento del DOM

Este es el que más vas a usar. En vanilla hacías `document.getElementById(...)` para coger un elemento. En React no quieres tocar el DOM a mano (de eso huíamos), pero a veces necesitas una referencia directa a un elemento real: para darle foco, para hacer scroll hasta él, para medirlo.

El patrón es siempre el mismo, tres pasos:

```jsx
import { useRef } from "react";

function Buscador() {
  const inputRef = useRef(null); // 1. creas el ref

  function enfocar() {
    inputRef.current.focus(); // 3. lo usas: inputRef.current ES el <input> real
  }

  return (
    <div>
      <input ref={inputRef} placeholder="Buscar…" /> {/* 2. lo enganchas con ref={} */}
      <button onClick={enfocar}>Ir al buscador</button>
    </div>
  );
}
```

- **Creas** el ref con `useRef(null)`.
- **Lo enganchas** a un elemento JSX con el atributo `ref={inputRef}`.
- A partir de ahí, **`inputRef.current` es ese nodo del DOM de verdad**, con todos sus métodos (`.focus()`, `.scrollIntoView()`, `.value`…).

> 💡 ¿Por qué `null` al crearlo? Porque en el primer render el elemento aún no existe; React rellena `inputRef.current` justo después de pintar. Por eso no lo uses durante el render, sino dentro de un evento o un `useEffect`.

---

## Práctica guiada 1: foco automático al cargar

Un detalle de UX que se ve profesional: cuando abres una pantalla de búsqueda o un login, el cursor ya está en el primer campo. Júntalo con lo de la clase anterior: `useEffect` con `[]` se ejecuta **una vez al montar**, justo cuando el ref ya apunta al DOM.

```jsx
import { useRef, useEffect } from "react";

function CampoBusqueda() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus(); // al montar, foco directo al input
  }, []);

  return <input ref={inputRef} placeholder="Empieza a escribir…" />;
}
```

Pruébalo: recarga y verás el cursor ya parpadeando en el campo, sin tocar nada. Eso es `useRef` + `useEffect` trabajando juntos.

Commit:

```bash
git add .
git commit -m "feat: foco automático en input con useRef y useEffect"
```

---

## El problema que resuelve `useContext`: el prop drilling

Monta mentalmente este árbol de componentes:

```
App  (aquí vive el usuario logueado)
 └─ Layout
     └─ Cabecera
         └─ MenuUsuario   ← AQUÍ se necesita el usuario
```

`App` tiene el dato (`usuario`), pero quien lo necesita es `MenuUsuario`, tres niveles más abajo. Con lo que sabes hoy, tendrías que pasarlo de prop en prop:

```jsx
<Layout usuario={usuario} />
//   <Cabecera usuario={usuario} />
//       <MenuUsuario usuario={usuario} />
```

`Layout` y `Cabecera` reciben `usuario` **solo para pasarlo hacia abajo**; ellos no lo usan. Eso es el **prop drilling**: un dato atravesando componentes que no le importan, ensuciando sus firmas. Con uno o dos niveles se aguanta; con un usuario o un tema que necesita media app, es insufrible.

> 💡 No todo dato compartido necesita context. Si solo baja un nivel, una prop está perfecta. El context es para datos **globales** que necesita mucha gente repartida por el árbol: usuario, tema, idioma, carrito.

---

## `useContext`: un canal directo de arriba a abajo

El context crea una especie de "emisora": un componente de arriba **emite** un valor (`Provider`), y cualquier componente de abajo, por hondo que esté, **sintoniza** ese valor sin que nadie se lo pase por props. Son tres piezas.

### 1. Crear el context

En su propio archivo, `src/context/TemaContext.jsx`:

```jsx
import { createContext } from "react";

export const TemaContext = createContext(null);
```

### 2. Emitir el valor con `<Provider>`

Arriba del árbol, envuelves lo que quieras "alimentar" y le pasas el valor en la prop `value`:

```jsx
import { useState } from "react";
import { TemaContext } from "./context/TemaContext.jsx";

function App() {
  const [tema, setTema] = useState("claro");

  return (
    <TemaContext.Provider value={{ tema, setTema }}>
      <Layout />
    </TemaContext.Provider>
  );
}
```

Todo lo que cuelgue de `<Layout />` tiene ya acceso a `{ tema, setTema }`, sin recibirlos por props.

### 3. Consumir el valor con `useContext`

En cualquier componente de abajo, por hondo que esté:

```jsx
import { useContext } from "react";
import { TemaContext } from "../context/TemaContext.jsx";

function BotonTema() {
  const { tema, setTema } = useContext(TemaContext);

  return (
    <button onClick={() => setTema(tema === "claro" ? "oscuro" : "claro")}>
      Tema actual: {tema}
    </button>
  );
}
```

`useContext(TemaContext)` te devuelve exactamente lo que pusiste en `value`. `Cabecera`, `Layout` y compañía ya no se enteran de que existe `tema`: el dato viaja directo del `Provider` al `useContext`, saltándose a los intermediarios.

> ⚠️ `useContext` lee del **Provider más cercano por encima** en el árbol. Si un componente no está envuelto por el `Provider`, recibe el valor por defecto que pusiste en `createContext(null)` → normalmente `null`, y ahí llega el error "no puedo leer `tema` de null". Regla: el `Provider` va **por encima** de todo el que vaya a consumirlo.

---

## Práctica guiada 2: contexto de tema en toda la app

Vamos a montar el flujo completo, que es justo el patrón que usarás para el usuario logueado.

**Paso 1 — Crea el context** en `src/context/TemaContext.jsx` (el de arriba).

**Paso 2 — Envuelve la app** con el `Provider` en `App.jsx`, guardando el tema en estado.

**Paso 3 — Crea dos componentes que consuman el tema** en sitios distintos del árbol, para comprobar que ninguno recibe props:

```jsx
// src/components/Cabecera.jsx
import { useContext } from "react";
import { TemaContext } from "../context/TemaContext.jsx";

function Cabecera() {
  const { tema } = useContext(TemaContext);
  return <header className={tema}>Mi app · modo {tema}</header>;
}

export default Cabecera;
```

```jsx
// src/components/BotonTema.jsx
import { useContext } from "react";
import { TemaContext } from "../context/TemaContext.jsx";

function BotonTema() {
  const { tema, setTema } = useContext(TemaContext);
  return (
    <button onClick={() => setTema(tema === "claro" ? "oscuro" : "claro")}>
      Cambiar a {tema === "claro" ? "oscuro" : "claro"}
    </button>
  );
}

export default BotonTema;
```

Pulsa el botón y mira: `Cabecera` reacciona al cambio aunque el `setTema` lo dispara `BotonTema`, que está en otra rama del árbol. El estado vive arriba (en `App`), pero lo comparten a través del context. **Eso es estado compartido sin prop drilling.**

Commit:

```bash
git add .
git commit -m "feat: contexto de tema compartido con createContext y useContext"
```

---

## El patrón que de verdad te llevas: un contexto con su hook

En proyectos reales no se reparte el `Context` pelado: se envuelve todo en un componente `Provider` propio y un **hook a medida** para consumirlo. Quédate con esta forma, porque es la que montarás para la autenticación:

```jsx
// src/context/AuthContext.jsx
import { createContext, useContext, useState } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [usuario, setUsuario] = useState(null); // null = nadie logueado

  const login = (datos) => setUsuario(datos);
  const logout = () => setUsuario(null);

  return (
    <AuthContext.Provider value={{ usuario, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// hook a medida: en vez de useContext(AuthContext) por toda la app
export function useAuth() {
  return useContext(AuthContext);
}
```

Y en `main.jsx` envuelves la app entera:

```jsx
<AuthProvider>
  <BrowserRouter>
    <App />
  </BrowserRouter>
</AuthProvider>
```

A partir de ahí, en cualquier componente: `const { usuario, login, logout } = useAuth();`. Fíjate en el `{ children }`: es la prop especial que contiene "todo lo que metas dentro de la etiqueta". Por eso `<AuthProvider><App /></AuthProvider>` funciona: `<App />` entra como `children`.

> 💡 Hoy `login` solo guarda datos en memoria. Más adelante, el día de la integración con el backend, `login` hará el fetch real al `/auth/login`, guardará el token y rellenará el usuario. La **forma** es la misma; solo cambiamos las tripas.

---

## Tu proyecto

Tu app va a necesitar saber **quién está logueado** desde muchos sitios: la cabecera (para mostrar el nombre o el botón de salir), las páginas privadas, el menú… Es el caso de libro para context. Hoy dejas montada la fontanería, aunque el login real venga después.

- Crea `src/context/AuthContext.jsx` con `AuthProvider` y el hook `useAuth`, tal cual el patrón de arriba.
- Envuelve tu app con `<AuthProvider>` en `main.jsx` (por fuera del `BrowserRouter`).
- En tu cabecera, consume `useAuth()` y muestra de forma condicional: si hay `usuario`, su nombre + un botón "Salir" (que llame a `logout`); si no, un enlace a "/login".
- De momento, para probarlo, puedes simular el login con un botón que llame a `login({ nombre: "Prueba" })`. La conexión de verdad llega pronto.

Y un toque de UX con lo de hoy: en tu futura página de login, pon `useRef` + `useEffect` para que el campo de email tenga el foco al cargar.

```bash
git add .
git commit -m "feat: AuthContext y useAuth listos para la sesión de usuario"
git push
```

Tablero: mueve a **Hecho** las tarjetas de estado compartido / contexto de sesión, y crea en **Por hacer** las de lógica reutilizable (en la próxima clase extraemos hooks propios y montamos un mini-store con `useReducer`).

---

## Reto opcional

1. **Persistir el tema.** Haz que el tema se guarde en `localStorage` y se recupere al recargar. Pista: lee el valor inicial con `useState(() => localStorage.getItem("tema") || "claro")` y guárdalo en un `useEffect` con `[tema]` en dependencias.
2. **`usePrevious`.** Usa `useRef` para recordar el **valor anterior** de un estado. En un `useEffect`, guarda el valor actual en el ref *después* de pintarlo; así, en el siguiente render, el ref aún tiene el anterior. Te sorprenderá lo útil que es.
3. **Scroll a un elemento.** Pon un `ref` en un elemento del final de una lista larga y un botón "Ir al final" que haga `ref.current.scrollIntoView({ behavior: "smooth" })`.
4. **Context de idioma.** Monta un segundo context (`IdiomaContext`) con "es"/"en" y un par de textos que cambien. Comprobarás que puedes tener **varios Providers anidados** sin que se estorben.
