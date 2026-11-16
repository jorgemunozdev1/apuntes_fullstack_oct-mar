# Props: componentes que reciben datos

En la clase anterior creaste tu primer componente y lo reutilizaste: `<Saludo />` tres veces. Pero te quedó una espinita. Las tres veces decía **exactamente lo mismo**, y si creabas una `Tarjeta`, todas las tarjetas eran clones. Un componente que solo sabe pintar una cosa fija no sirve de mucho: querrás una tarjeta para cada producto, un saludo para cada persona, una fila para cada alumno.

Lo que te falta es poder **pasarle datos distintos a cada uso del componente**. Eso son las **props**. Con ellas, un mismo componente se convierte en un molde reutilizable que pinta cosas diferentes según lo que le des. Hoy aprendes a pasar props, a leerlas, a darles valores por defecto, y a usar todo eso para **pintar listas** a partir de un array de datos.

---

## Paso 1 — Una prop es un argumento del componente

Piénsalo con algo que ya dominas: una **función** recibe argumentos y se comporta distinto según ellos.

```js
saludar("Fran"); // Hola, Fran
saludar("Ada"); // Hola, Ada
```

Un **componente es lo mismo**: le pasas datos y pinta distinto según ellos. Esos datos se llaman **props** (de _properties_), y se los pasas como si fueran **atributos** de la etiqueta:

```jsx
<Saludo nombre="Fran" />
<Saludo nombre="Ada" />
```

¿Y dónde llegan esos datos? React los empaqueta en **un objeto** y se lo pasa a tu función como **primer parámetro**, llamado por convención `props`:

```jsx
function Saludo(props) {
  return <p>¡Hola, {props.nombre}! 👋</p>;
}

export default Saludo;
```

Cuando escribes `<Saludo nombre="Fran" />`, React llama a `Saludo({ nombre: "Fran" })`. La prop entra por el objeto `props` y la sacas con `{props.nombre}` dentro del JSX (esas llaves de la clase pasada, metiendo JavaScript en medio).

> 💡 La idea clave: **el componente lo escribes una vez; los datos los pones desde fuera, en cada uso.** El mismo `Saludo` ahora saluda a quien tú quieras.

---

## Paso 2 — Léelas con destructuring

Escribir `props.nombre`, `props.edad`, `props.email`… se hace pesado. Como `props` es un objeto normal, le aplicas **destructuring** (lo que ya usabas en M1) directamente en el parámetro:

```jsx
function Saludo({ nombre }) {
  return <p>¡Hola, {nombre}! 👋</p>;
}
```

Es idéntico a lo anterior, pero más limpio: ves de un vistazo **qué props espera** el componente con solo leer su primera línea. A partir de ahora lo escribiremos siempre así.

---

## Paso 3 — Varias props: tu primer componente base

Un componente puede recibir tantas props como quieras. Vamos a construir una **`Tarjeta`** de verdad, un componente _base_ reutilizable que pinta un título y una descripción que le pases.

Crea `src/Tarjeta.jsx`:

```jsx
function Tarjeta({ titulo, descripcion }) {
  return (
    <div className="tarjeta">
      <h2>{titulo}</h2>
      <p>{descripcion}</p>
    </div>
  );
}

export default Tarjeta;
```

Y úsalo en `App.jsx` pasándole datos distintos cada vez:

```jsx
import Tarjeta from "./Tarjeta.jsx";

function App() {
  return (
    <div>
      <h1>Catálogo</h1>
      <Tarjeta titulo="Teclado" descripcion="Mecánico, retroiluminado" />
      <Tarjeta titulo="Ratón" descripcion="Inalámbrico, 6 botones" />
      <Tarjeta titulo="Monitor" descripcion="27 pulgadas, 144Hz" />
    </div>
  );
}

export default App;
```

Tres tarjetas, **un solo componente**, y cada una dice algo diferente. Eso es un **componente base**: una pieza pequeña, genérica y reutilizable con la que construyes la interfaz, como ladrillos.

> 💡 Hasta ahora las props eran texto (`titulo="..."`). Si quieres pasar un **número**, un **booleano** o una **variable**, usa llaves: `<Tarjeta precio={30} oferta={true} titulo={miVariable} />`. Las comillas son solo para texto literal; para cualquier otra cosa de JavaScript, `{ }`.

---

## Paso 4 — Las props son de solo lectura

Una regla que **no se rompe**: un componente **nunca modifica sus props**. Las recibe, las lee, y con ellas devuelve JSX. Punto.

```jsx
function Tarjeta({ titulo }) {
  titulo = titulo.toUpperCase(); // ❌ NO. No toques las props.
  return <h2>{titulo}</h2>;
}
```

¿Te suena de algo? Es la misma idea de las **funciones puras** de la kata del carrito: entran datos, sale un resultado, y no se manosea la entrada. Un componente es eso: **`props` entran → JSX sale**. Esa disciplina es lo que hace que una app de React sea predecible: el padre manda los datos, el hijo solo los pinta.

> ⚠️ Las props viajan **en un solo sentido**: de fuera (el padre) hacia dentro (el hijo). El hijo no puede cambiar los datos de quien lo usó. Cuando necesites que algo cambie con el tiempo (un contador, un input), eso será **estado**, y lo verás en la próxima clase.

---

## Paso 5 — Valores por defecto

¿Y si a veces no pasas una prop? Mejor que se pinte un valor razonable a que aparezca un hueco vacío. Como es destructuring normal, le das un **valor por defecto** igual que a cualquier parámetro:

```jsx
function Tarjeta({ titulo, descripcion = "Sin descripción" }) {
  return (
    <div className="tarjeta">
      <h2>{titulo}</h2>
      <p>{descripcion}</p>
    </div>
  );
}
```

Ahora `<Tarjeta titulo="Cable" />` (sin descripción) no deja un párrafo vacío, sino "Sin descripción". Pequeño detalle, gran robustez.

---

## Paso 6 — El caso de verdad: pintar una lista con `.map()`

En una app real no escribes las tarjetas a mano una a una: los datos vienen en un **array** (de una API, de una base de datos…) y tú lo recorres. Aquí se juntan dos cosas que ya tienes: las props y el `.map()` de M1.

Parte de un array de datos en `App.jsx`:

```jsx
import Tarjeta from "./Tarjeta.jsx";

const productos = [
  { id: 1, nombre: "Teclado", detalle: "Mecánico, retroiluminado" },
  { id: 2, nombre: "Ratón", detalle: "Inalámbrico, 6 botones" },
  { id: 3, nombre: "Monitor", detalle: "27 pulgadas, 144Hz" },
];

function App() {
  return (
    <div>
      <h1>Catálogo</h1>
      {productos.map((producto) => (
        <Tarjeta
          key={producto.id}
          titulo={producto.nombre}
          descripcion={producto.detalle}
        />
      ))}
    </div>
  );
}

export default App;
```

Léelo despacio, porque es uno de los patrones que más vas a repetir en React:

- `.map()` transforma **cada objeto del array en un componente `<Tarjeta>`**. De un array de datos sale un array de elementos JSX.
- Ese `.map()` va dentro de `{ }` porque es JavaScript dentro del JSX.
- A cada `<Tarjeta>` le pasas las props sacadas de su producto.

Si el array tiene 3 productos o 300, **no tocas el JSX**: el `.map()` pinta los que haya. Tus datos mandan; la interfaz se genera a partir de ellos. Otra vez `UI = f(datos)`.

---

## Paso 7 — La prop `key`: no es decorativa

¿Has visto el `key={producto.id}`? No es opcional. Cuando pintas una lista, React **te exige** una prop especial `key` con un valor **único** por elemento.

¿Por qué? Recuerda el Virtual DOM: cuando los datos cambian, React compara la lista nueva con la vieja para tocar **solo lo mínimo**. La `key` es la **etiqueta de identidad** de cada elemento: gracias a ella React sabe "este de aquí es el mismo de antes, este es nuevo, este se borró", en vez de repintar la lista entera.

Reglas de la `key`:

- Tiene que ser **única entre los hermanos** y **estable** (que no cambie entre repintados).
- Usa un **id de verdad** de tus datos (`producto.id`). Casi siempre lo tendrás.
- **Evita usar el índice del `.map()`** como key salvo que no haya nada mejor: si la lista se reordena o se borra algo, el índice engaña a React y provoca bugs sutiles.

> ⚠️ Si te olvidas de la `key`, la app funciona pero React te chilla en la consola con un _warning_ amarillo. No lo ignores: es un bug esperando a aparecer en cuanto la lista cambie.

---

## Si te sobra tiempo

1. **Más props.** Añade a `Tarjeta` una prop `precio` (número) y otra `disponible` (booleano). Pinta el precio y, según `disponible`, muestra "En stock ✅" o "Agotado ❌" con un ternario dentro de `{ }`.
2. **Componente base de verdad.** Crea un componente `Boton` que reciba una prop `texto` y pinte un `<button>{texto}</button>`. Úsalo en varias tarjetas. Acabas de empezar tu propia librería de piezas reutilizables.
3. **Datos más grandes.** Amplía el array `productos` a 8 ítems. Comprueba que **no tocas nada del JSX**: el `.map()` los pinta todos solo. Ese es el premio de separar datos de presentación.
4. **Rompe la `key`.** Quita el `key={producto.id}` y mira el warning en la consola del navegador. Vuelve a ponerlo. Recordar ese mensaje te ahorrará tiempo cuando lo veas en un proyecto real.
