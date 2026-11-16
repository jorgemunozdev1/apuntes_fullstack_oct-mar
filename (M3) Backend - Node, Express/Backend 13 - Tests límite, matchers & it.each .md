# Testing unitario a fondo: casos límite, matchers y `test.each`

Ayer escribiste tu primer test verde sobre una función pura. Fue el "hola mundo" del testing: una entrada, una salida, una comprobación. Pero te dejé un anzuelo en el reto: ¿qué pasa si llamas a `esPasswordSegura(undefined)`? Si lo probaste, viste que **revienta**. Ese es el tipo de cosa que un test del _happy path_ nunca habría pillado, y donde se esconden la mayoría de los bugs reales.

Hoy te quedas en lo unitario, pero vas en serio: aprendes a buscar los **casos límite** (las entradas raras que rompen las cosas), a usar **más matchers** que `toBe` (para arrays, para `null`, para errores), y a no repetirte cuando tienes muchos casos parecidos con **`test.each`**. Al final del día sabrás cubrir una pieza de lógica de verdad, no solo cuando todo va bien.

---

### ¿Qué es exactamente una "unidad"?

Una **unidad** es la pieza más pequeña que puedes probar por separado: normalmente, una función. Un **test unitario** prueba esa pieza **aislada**, sin sus compañeras: sin base de datos, sin red, sin `req`/`res`, sin depender de otros módulos. Solo la función, sus entradas y su salida.

Por eso ayer y hoy trabajamos con **funciones puras**: una función pura (misma entrada → misma salida, sin tocar nada de fuera) ya está aislada por naturaleza. No hay nada que apagar ni preparar. Le das algo, te devuelve algo, y compruebas.

> ¿Y un controlador, que toca la base de datos? Ese **no** es puro: depende del mundo exterior. Para testearlo aislado hay que "falsear" esa base de datos. Eso son los _mocks_, y es la clase de mañana. Hoy exprimimos lo que se puede testear sin trucos.

### Los bugs viven en los bordes

El _happy path_ —la entrada bonita y esperada— casi nunca falla. Los bugs se esconden en los **casos límite**: las entradas que están en el borde de lo válido, o directamente fuera. Antes de dar una función por probada, pregúntate "¿y si...?":

- ¿Y si la entrada está **vacía**? (`""`, `[]`, `{}`)
- ¿Y si es **justo el límite**? (si la regla es "≥ 6", prueba 5, 6 y 7)
- ¿Y si es `null`, `undefined`, o de **otro tipo** del que esperabas?
- ¿Y si es **cero, negativo, o enorme**?

Un junior prueba que su función funciona cuando todo va bien y la da por buena. Quien piensa en serio prueba **los bordes**, porque sabe que ahí es donde se rompe. A esto se le llama pensar en **valores frontera**: si hay una valla en el 6, los casos interesantes son los que están justo a cada lado de la valla.

### Más matchers: cuando `toBe` no basta

Ya conoces `toBe` (primitivos) y `toEqual` (objetos/arrays). Hay muchos más, y elegir el preciso hace que el test se lea solo y que el mensaje de error sea claro:

| Matcher                                  | Comprueba que…                                    |
| ---------------------------------------- | ------------------------------------------------- |
| `toBe(x)`                                | es exactamente `x` (primitivos, identidad)        |
| `toEqual(obj)`                           | mismo contenido en profundidad (objetos/arrays)   |
| `toBeNull()` / `toBeUndefined()`         | es `null` / es `undefined`                        |
| `toBeTruthy()` / `toBeFalsy()`           | es "verdadero"/"falso" en sentido booleano        |
| `toContain(x)`                           | el array contiene **ese mismo** valor             |
| `toContainEqual(obj)`                    | el array contiene un objeto **con ese contenido** |
| `toHaveLength(n)`                        | tiene longitud `n` (array o string)               |
| `toBeGreaterThan(n)` / `toBeLessThan(n)` | es mayor / menor que `n`                          |
| `toThrow()`                              | la función **lanza un error**                     |

Fíjate en la pareja `toContain` / `toContainEqual`: es **exactamente la misma trampa** que `toBe` / `toEqual` de ayer. `toContain` busca el mismo objeto en memoria; `toContainEqual` busca uno con el mismo contenido. Identidad vs. contenido, otra vez.

### `test.each`: no copies y pegues casos

Cuando tienes muchos casos que solo cambian en la entrada y la salida esperada, copiar y pegar bloques `it` es ruido. **`test.each`** (o `it.each`) te deja escribir una **tabla** de casos y genera un test por cada fila. Menos repetición, y se ve de un vistazo qué entradas cubres.

---

Trabaja sobre tu proyecto. Commit después de cada bloque.

### Bloque 1 — Casos límite: blinda tu propia función

Retomamos `esPasswordSegura` de ayer y la rompemos a propósito para encontrar sus bordes.

#### 1.1 Encuentra los bordes

Abre `utils/password.test.js` y añade estos casos. Antes de ejecutar, **predice** qué va a pasar con cada uno:

```js
it("acepta una contraseña de exactamente 6 caracteres", () => {
  expect(esPasswordSegura("123456")).toBe(true); // justo en la valla
});

it("rechaza una de 5 caracteres", () => {
  expect(esPasswordSegura("12345")).toBe(false); // justo por debajo
});

it("rechaza la cadena vacía", () => {
  expect(esPasswordSegura("")).toBe(false);
});

it("no revienta si le pasan undefined", () => {
  expect(esPasswordSegura(undefined)).toBe(false);
});
```

Ejecuta `npm test`. Los tres primeros pasan; **el último peta** con algo como `Cannot read properties of undefined (reading 'length')`. No es que el test esté mal: es que tu función **tiene un bug** con entradas que no son string. El test acaba de hacer su trabajo: encontrarlo.

#### 1.2 Arregla la función

Hazla robusta comprobando el tipo antes de mirar la longitud:

```js
export const esPasswordSegura = (password) => {
  if (typeof password !== "string") return false;
  return password.length >= 6;
};
```

Vuelve a ejecutar: los cuatro en verde. Acabas de cubrir el _happy path_, las dos fronteras (5 y 6), la entrada vacía y un tipo inválido. **Eso es probar una unidad de verdad**, no solo cuando le das lo que espera.

> 💾 **Commit:** `test: casos límite de esPasswordSegura + robustez`

---

### Bloque 2 — Matchers para arrays y "no encontrado"

Casi toda app filtra y busca en listas. Vamos a testear esa lógica con los matchers adecuados. Usaré videojuegos como ejemplo; **adáptalo a tu recurso** (filtrar notas por etiqueta, productos por categoría…).

#### 2.1 Las funciones

Crea `utils/juegos.js`:

```js
export const filtrarPorGenero = (juegos, genero) => {
  return juegos.filter((juego) => juego.genero === genero);
};

export const buscarPorTitulo = (juegos, titulo) => {
  return juegos.find((juego) => juego.titulo === titulo);
};
```

#### 2.2 Los tests

Crea `utils/juegos.test.js`. Fíjate cómo el `Arrange` (la lista) se comparte preparándolo una vez:

```js
import { describe, it, expect } from "vitest";
import { filtrarPorGenero, buscarPorTitulo } from "./juegos.js";

const juegos = [
  { titulo: "Hades", genero: "roguelike" },
  { titulo: "Celeste", genero: "plataformas" },
  { titulo: "Dead Cells", genero: "roguelike" },
];

describe("filtrarPorGenero", () => {
  it("devuelve solo los juegos del género pedido", () => {
    const resultado = filtrarPorGenero(juegos, "roguelike");
    expect(resultado).toHaveLength(2);
    expect(resultado).toContainEqual({ titulo: "Hades", genero: "roguelike" });
  });

  it("devuelve un array vacío si ningún juego coincide", () => {
    expect(filtrarPorGenero(juegos, "carreras")).toEqual([]);
  });
});

describe("buscarPorTitulo", () => {
  it("encuentra el juego por su título", () => {
    expect(buscarPorTitulo(juegos, "Celeste")).toEqual({
      titulo: "Celeste",
      genero: "plataformas",
    });
  });

  it("devuelve undefined si no existe", () => {
    expect(buscarPorTitulo(juegos, "No existe")).toBeUndefined();
  });
});
```

Tres cosas que mirar:

- **`toHaveLength(2)`** dice "esperaba dos resultados". Más claro que `expect(resultado.length).toBe(2)`.
- **`toContainEqual`**, no `toContain`: el array tiene objetos, y buscamos por contenido. Con `toContain` fallaría aunque el contenido fuera idéntico (la trampa identidad/contenido de ayer).
- El **array vacío** (`toEqual([])`) y el **no encontrado** (`toBeUndefined()`) son los casos límite. `.filter` y `.find` se comportan distinto cuando no hay coincidencias: uno da `[]`, el otro `undefined`. Tus tests lo dejan documentado.

> 💾 **Commit:** `test: filtrado y búsqueda de juegos`

---

### Bloque 3 — Testear errores con `toThrow`

A veces lo correcto no es devolver un valor raro, sino **lanzar un error**. ¿Cómo se testea que algo peta cuando _debe_ petar? Con `toThrow`, que tiene una sintaxis especial que confunde a todo el mundo la primera vez.

#### 3.1 Una función que valida y lanza

Añade a `utils/juegos.js`:

```js
export const precioConDescuento = (precio, descuento) => {
  if (descuento < 0 || descuento > 100) {
    throw new Error("El descuento debe estar entre 0 y 100");
  }
  return precio - (precio * descuento) / 100;
};
```

#### 3.2 El test del error

```js
import { precioConDescuento } from "./juegos.js";

describe("precioConDescuento", () => {
  it("aplica el descuento correctamente", () => {
    expect(precioConDescuento(60, 50)).toBe(30);
  });

  it("lanza un error si el descuento es mayor que 100", () => {
    expect(() => precioConDescuento(60, 150)).toThrow();
  });

  it("el mensaje del error explica el problema", () => {
    expect(() => precioConDescuento(60, 150)).toThrow("entre 0 y 100");
  });
});
```

**El detalle clave**: fíjate en que dentro del `expect` no pasas `precioConDescuento(60, 150)` directamente, sino **una función que la envuelve**: `() => precioConDescuento(60, 150)`.

¿Por qué? Si la llamaras directa, la función se ejecutaría **antes** de que `expect` pueda hacer nada, lanzaría el error ahí mismo y el test reventaría sin poder comprobarlo. Al envolverla en una flecha, le das a Vitest la función **sin ejecutarla todavía**: es Vitest quien la llama, dentro de un `try/catch`, y comprueba que efectivamente lanzó.

> Regla: **para `toThrow`, siempre envuelve la llamada en `() => ...`.**

Y `toThrow("entre 0 y 100")` además comprueba que el **mensaje** contiene ese texto: así te aseguras de que lanzó por el motivo correcto, no por otro error cualquiera.

> 💾 **Commit:** `test: precioConDescuento valida el rango`

---

### Bloque 4 — No te repitas: `test.each`

Mira el Bloque 1: cuatro `it` casi idénticos, cambiando solo la entrada y el resultado esperado. Eso es una **tabla** disfrazada de copia-pega. Vamos a escribirla como tabla.

#### 4.1 Parametriza los casos

Añade en `utils/password.test.js` (puedes sustituir los `it` sueltos del Bloque 1 por esto):

```js
describe("esPasswordSegura (tabla de casos)", () => {
  it.each([
    ["123456", true], // justo 6 → válida
    ["12345", false], // 5 → corta
    ["", false], // vacía
    ["unaClaveLargaYsegura", true], // larga
  ])("para '%s' devuelve %s", (password, esperado) => {
    expect(esPasswordSegura(password)).toBe(esperado);
  });
});
```

Cada **fila** del array se convierte en un test independiente. Los `%s` del título se rellenan con los valores de la fila, así que en la salida verás cuatro líneas distintas: `para '123456' devuelve true`, `para '12345' devuelve false`… Si una falla, sabes exactamente **qué fila** fue.

Ejecútalo y mira cómo Vitest lista cada caso por separado, aunque escribiste el cuerpo una sola vez.

#### 4.2 Cuándo SÍ y cuándo NO

`test.each` brilla cuando los casos tienen **la misma forma** y solo cambian los datos (como aquí). Pero no fuerces todo a una tabla: si un caso prueba un **comportamiento distinto** (lanza un error, devuelve otra estructura), déjalo en su propio `it` con un nombre descriptivo. La tabla es para "lo mismo con datos distintos"; los `it` sueltos, para "comportamientos distintos".

> 💾 **Commit:** `test: parametrizar casos de contraseña con test.each`

---

## 🚀 Reto opcional

1. **Cubre `precioConDescuento` por completo.** Faltan fronteras: descuento de `0` (no descuenta nada), de `100` (queda gratis), y de `-1` (debería lanzar). Añade esos casos. ¿Pasan todos? Si `0` o `100` fallan, mira bien tu condición `< 0 || > 100`: ¿incluye o excluye los extremos? Es el típico bug de "≤ vs <".
2. **`test.each` con tabla de objetos.** Investiga la otra sintaxis de `test.each`, la que usa **plantillas con `$variable`** en el título en vez de `%s`, pasando un array de objetos `[{ password, esperado }]`. ¿Cuándo se lee mejor cada una? Pista: con muchas columnas, los nombres (`$password`) ganan a los `%s` posicionales.
