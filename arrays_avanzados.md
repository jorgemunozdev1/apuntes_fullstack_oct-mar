# Repaso de JavaScript: Arrays y métodos de búsqueda y manipulación

## 1. Declaración de arrays de objetos

Un array permite almacenar varios elementos. En JavaScript podemos guardar objetos dentro de un array, de forma que cada objeto contenga diferentes propiedades.

### Array de personas

Cada persona tiene un nombre y una nota.

```javascript
const arrayPersonas = [
  { nombre: "Jorge", nota: 5.5 },
  { nombre: "Antonio", nota: 7.4 },
  { nombre: "Rafa", nota: 2.45 },
  { nombre: "Jose", nota: 7.8 },
  { nombre: "Álvaro", nota: 5.6 },
];
```

### Array de productos

Cada producto tiene unas propiedades comunes, como `nombre`, `precio`, `stock` (unidades disponibles) y `tipo`, y otras que pueden variar según el producto.

```javascript
const productos = [
  {
    nombre: "Camiseta",
    precio: 18.99,
    stock: 23,
    tipo: "ropa",
    talla: "M",
    color: "azul",
  },
  {
    nombre: "Pantalón",
    precio: 34.5,
    stock: 25,
    tipo: "ropa",
    talla: "L",
    marca: "DenimCo",
  },
  {
    nombre: "Auriculares",
    precio: 49.99,
    stock: 64,
    tipo: "tecnología",
    marca: "SoundMax",
    inalámbricos: true,
  },
  {
    nombre: "Ratón",
    precio: 15.75,
    stock: 3,
    tipo: "tecnología",
    marca: "Click",
    inalámbricos: false,
  },
  {
    nombre: "Manzana",
    precio: 0.8,
    stock: 170,
    tipo: "alimentación",
    origen: "España",
    ecológico: true,
  },
  {
    nombre: "Pan",
    precio: 1.2,
    stock: 10,
    tipo: "alimentación",
    peso: "500 g",
  },
];

// Mostramos el array completo en la consola.
console.log(productos);
```

**Importante:** no todos los objetos tienen que tener exactamente las mismas propiedades. Por ejemplo, el objeto `Camiseta` tiene `color`, mientras que el objeto `Pan` tiene `peso`.

---

## 2. Métodos `some()` y `every()`

Estos métodos permiten comprobar si se cumplen determinadas condiciones en los elementos de un array.

### `some()`

Devuelve `true` si **al menos un elemento** cumple la condición. Si ninguno la cumple, devuelve `false`.

```javascript
const hayAuriculares = productos.some((p) => p.nombre == "Auriculares");
```

En este caso, devuelve `true` porque existe un producto cuyo nombre es `"Auriculares"`.

### `every()`

Devuelve `true` si **todos los elementos** cumplen la condición. Si alguno no la cumple, devuelve `false`.

```javascript
const todosSonAuriculares = productos.every((p) => p.nombre == "Auriculares");
```

En este caso, devuelve `false` porque no todos los productos son auriculares.

**Diferencia principal:**

- `some()`: ¿hay al menos uno que cumpla la condición?
- `every()`: ¿cumplen todos la condición?

---

## 3. Métodos `find()` y `findIndex()`

Permiten buscar elementos dentro de un array utilizando una condición.

### `find()`

Devuelve el **primer elemento que cumple la condición**. Si no encuentra ninguno, devuelve `undefined`.

```javascript
const auriculares = productos.find((p) => p.nombre == "Auriculares");
```

La variable `auriculares` contiene el objeto completo correspondiente a los auriculares, no solamente su nombre.

### `findIndex()`

Devuelve el índice o posición del primer elemento que cumple la condición. Si no encuentra ninguno, devuelve `-1`.

```javascript
const posicionAuriculares = productos.findIndex(
  (p) => p.nombre == "Auriculares",
);
```

En este caso, devuelve `2`, ya que los índices de los arrays empiezan en cero:

| Índice | Producto    |
| -----: | ----------- |
|      0 | Camiseta    |
|      1 | Pantalón    |
|      2 | Auriculares |
|      3 | Ratón       |
|      4 | Manzana     |
|      5 | Pan         |

**Recuerda:** `find()` devuelve el elemento; `findIndex()` devuelve su posición.

---

## 4. Métodos `splice()` y `toSpliced()`

Ambos permiten eliminar elementos de un array indicando su posición y el número de elementos que queremos eliminar, pero existe una diferencia fundamental.

### `splice()`: modifica el array original

```javascript
productos.splice(posicionAuriculares, 1);
```

- El primer argumento indica la posición desde la que comenzamos.
- El segundo argumento indica cuántos elementos queremos eliminar.
- Modifica directamente el array original.
- Devuelve un array con los elementos eliminados.

En nuestro ejemplo, se elimina el producto situado en la posición `2`, que corresponde a los auriculares.

### `toSpliced()`: devuelve un nuevo array

```javascript
const productosSinAuriculares = productos.toSpliced(posicionAuriculares, 1);
```

- Recibe los mismos tipos de argumentos.
- No modifica el array original.
- Devuelve un nuevo array con los cambios aplicados.

**Importante:** en el código de este ejemplo, `splice()` se ejecuta antes que `toSpliced()`. Por tanto, cuando se ejecuta `toSpliced()`, los auriculares ya han sido eliminados del array original.

Para comprobar los resultados:

```javascript
// Mostramos el array original, que ya ha sido modificado por splice().
console.log(productos);

// Mostramos el nuevo array devuelto por toSpliced().
console.log(productosSinAuriculares);
```

Como el índice `2` ya no contiene los auriculares después de ejecutar `splice()`, `toSpliced(2, 1)` eliminará el siguiente producto que ocupe esa posición, que será el ratón.

**Conclusión:** si queremos comparar ambos métodos partiendo del mismo array, debemos hacerlo antes de modificarlo con `splice()`.

---

## 5. Filtrar productos con `filter()`

El método `filter()` devuelve un nuevo array con los elementos que cumplen una condición.

```javascript
// Obtenemos los productos cuyo precio es inferior a 30.
const baratos = productos.filter((producto) => producto.precio < 30);
```

El array `baratos` contendrá los productos que tengan un precio inferior a `30`.

### Filtrar por varias condiciones

Podemos combinar condiciones utilizando operadores lógicos.

```javascript
const baratos = productos.filter(
  (producto) => producto.precio < 30 && producto.tipo == "ropa",
);
```

En este caso, solo se incluyen los productos que cumplan ambas condiciones:

- Su precio es inferior a `30`.
- Su tipo es `"ropa"`.

El operador `&&` significa que ambas condiciones deben ser verdaderas.

**Nota:** estos ejemplos están comentados en el código original porque utilizan el mismo nombre de variable, `baratos`. Para ejecutarlos juntos, habría que utilizar nombres diferentes o descomentar únicamente uno de ellos.

---

## 6. Crear una función para buscar productos por tipo

Podemos crear nuestras propias funciones para reutilizar operaciones sobre arrays.

```javascript
function buscarPorTipo(arrayProductos, tipo) {
  // Convertimos el texto introducido a minúsculas.
  let tipoMinuscula = tipo.toLowerCase();

  // Convertimos la primera letra a mayúscula.
  let primeraMayuscula =
    tipoMinuscula.charAt(0).toUpperCase() + tipoMinuscula.slice(1);

  // Filtramos los productos cuyo tipo contiene el texto indicado.
  return arrayProductos.filter((producto) =>
    producto.tipo.includes(tipoMinuscula),
  );
}
```

### ¿Qué hace cada método?

- `toLowerCase()`: convierte todo el texto a minúsculas.
- `charAt(0)`: obtiene el primer carácter de una cadena.
- `toUpperCase()`: convierte el carácter a mayúscula.
- `slice(1)`: devuelve la cadena desde el índice `1` hasta el final.
- `includes()`: comprueba si una cadena contiene otra cadena.
- `filter()`: devuelve un nuevo array con los elementos que cumplen la condición.
- `return`: devuelve el resultado de la función.

La variable `primeraMayuscula` se calcula, pero en esta versión no se utiliza en el filtrado. Se mantiene para respetar el código trabajado en clase.

### Ejemplo de uso

```javascript
const baratos = productos.filter((producto) => producto.precio < 30);

console.log(buscarPorTipo(baratos, ""));
```

En este ejemplo, el texto de búsqueda está vacío (`""`). Como todas las cadenas contienen la cadena vacía, se devolverán todos los elementos del array `baratos`.

Si queremos buscar productos de ropa:

```javascript
console.log(buscarPorTipo(baratos, "ropa"));
```

---

## 7. Transformar elementos con `map()`

El método `map()` recorre los elementos de un array y devuelve un nuevo array con los resultados de aplicar una función a cada elemento.

No modifica el array original.

```javascript
const nuevoArray = arrayPersonas.map((persona) => {
  let nuevaNota = Number((Math.random(persona.nota) * 100).toFixed(2));

  return {
    nombre: persona.nombre,
    nota: nuevaNota,
  };
});

const baratos = productos.filter((p) => p.precio < 20);

console.log(nuevoArray);
```

### ¿Qué sucede en este ejemplo?

1. `map()` recorre cada persona del array `arrayPersonas`.
2. Se calcula una nueva nota mediante `Math.random()`.
3. `toFixed(2)` limita el resultado a dos decimales y lo convierte en una cadena.
4. `Number()` convierte ese resultado nuevamente a número.
5. `return` crea un nuevo objeto con el nombre original y la nueva nota.

**Importante:** `Math.random()` no utiliza argumentos para establecer un valor máximo. Por tanto, `Math.random(persona.nota)` genera un número aleatorio entre `0` (incluido) y `1` (excluido), independientemente del argumento recibido. Al multiplicarlo por `100`, se obtiene una cifra entre `0` y `100`, antes de redondearla a dos decimales.

---

## 8. Métodos `includes()` e `indexOf()`

Estos métodos permiten buscar valores concretos dentro de un array.

### `includes()`

Devuelve `true` si el array contiene el elemento indicado y `false` si no lo contiene.

```javascript
console.log("Está Pepe? " + arrayPersonas.includes("Pepito"));
```

En este ejemplo, el resultado es `false`, porque `arrayPersonas` contiene objetos, no una cadena con el valor `"Pepito"`.

Para buscar nombres dentro de un array de objetos, podemos utilizar `some()`:

```javascript
console.log(arrayPersonas.some((persona) => persona.nombre == "Jorge"));
```

### `indexOf()`

Devuelve el índice de la primera aparición del valor indicado. Si no lo encuentra, devuelve `-1`.

```javascript
console.log("Posicion de Pepe: " + arrayPersonas.indexOf("Pepe"));
```

En este caso devuelve `-1`, porque no hay ningún elemento que sea exactamente la cadena `"Pepe"`.

Con un array de valores simples (cadenas o números) sí encuentra el elemento:

```javascript
const frutas = ["manzana", "pera", "uva", "kiwi"];

console.log(frutas.indexOf("uva")); // 2
console.log(frutas.indexOf("melón")); // -1 (no existe)
```

### Combinar `indexOf()` con `splice()` para eliminar un elemento

Si sabemos el **valor** pero no la posición, primero buscamos el índice con `indexOf()` y después lo sacamos con `splice(posicion, 1)`, que modifica el array original:

```javascript
const posicion = frutas.indexOf("uva");

// Comprobamos que existe antes de eliminar: splice(-1, 1) borraría el último elemento.
if (posicion !== -1) {
  const eliminado = frutas.splice(posicion, 1)[0];
  console.log(eliminado); // "uva"
}

console.log(frutas); // ["manzana", "pera", "kiwi"]
```

Para un array de objetos, el equivalente es `findIndex()` con una condición:

```javascript
const posicionRafa = arrayPersonas.findIndex((p) => p.nombre == "Rafa");
```

**Importante:** `includes()` e `indexOf()` buscan valores concretos. Para buscar una propiedad dentro de objetos, normalmente utilizaremos métodos como `find()`, `findIndex()` o `some()`.

---

## 9. Recorrer un array con `for...of`

Otra forma de recorrer un array es mediante un bucle `for...of`.

```javascript
const arrayPersonasv2 = [];

for (const persona of arrayPersonas) {
  if (persona != "Rafa") {
    arrayPersonasv2.push(persona);
  }
}
```

En cada iteración, `persona` representa el elemento actual del array.

- `for...of`: recorre los elementos del array.
- `if`: permite ejecutar el bloque únicamente si se cumple una condición.
- `push()`: añade un elemento al final del array.

**Observación:** como `arrayPersonas` contiene objetos, la condición `persona != "Rafa"` no elimina a Rafa. Para excluirlo correctamente, debemos comparar su propiedad `nombre`:

```javascript
// Ejemplo corregido para practicar la comparación de propiedades.
const arrayPersonasv2 = [];

for (const persona of arrayPersonas) {
  if (persona.nombre != "Rafa") {
    arrayPersonasv2.push(persona);
  }
}
```

Esta segunda versión sí excluye a la persona cuyo nombre es `"Rafa"`.

---

## 10. Añadir elementos: `unshift()` y `push()`

Podemos añadir elementos al principio o al final de un array.

```javascript
// Añade un elemento al principio del array.
arrayPersonasv2.unshift("Jose");

// Añade un elemento al final del array.
arrayPersonasv2.push("Alvaro");

console.log(arrayPersonasv2);
```

- `unshift()`: añade uno o varios elementos al principio y modifica el array original.
- `push()`: añade uno o varios elementos al final y modifica el array original.

Ambos métodos devuelven la nueva longitud del array.

---

## 11. Eliminar elementos: `shift()` y `pop()`

Estos métodos eliminan elementos del array y devuelven el elemento eliminado.

```javascript
// Devuelve el primer elemento del array y lo elimina.
const nombrePrimerElemento = arrayPersonasv2.shift();

// Devuelve el último elemento del array y lo elimina.
const nombreUltimoElemento = arrayPersonasv2.pop();
```

- `shift()`: elimina el primer elemento.
- `pop()`: elimina el último elemento.

Ambos métodos modifican el array original.

---

## 12. Dividir y eliminar partes de un array con `splice()`

El método `splice()` permite eliminar elementos indicando una posición y, opcionalmente, un número de elementos.

### Eliminar desde una posición hasta el final

```javascript
const arrayPartido = arrayPersonasv2.splice(5 + 1);
```

Como `5 + 1` equivale a `6`, se eliminan todos los elementos desde el índice `6` hasta el final del array.

El resultado de `splice()` es un nuevo array que contiene los elementos eliminados.

### Eliminar un número concreto de elementos

```javascript
const arrayPartidov2 = arrayPersonasv2.splice(5, 2);
```

En este caso:

- `5`: índice desde el que comenzamos.
- `2`: número de elementos que queremos eliminar.

Se eliminan hasta dos elementos desde el índice indicado, siempre que existan suficientes elementos.

Para mostrar los resultados:

```javascript
console.log(arrayPartido);
console.log(arrayPartidov2);
```

**Importante:** si ejecutamos los dos ejemplos seguidos, el segundo `splice()` actuará sobre el array que ya ha sido modificado por el primero.

---

## 13. Crear objetos nuevos con `map()` (seleccionar propiedades)

`map()` también sirve para quedarnos solo con algunas propiedades de cada objeto.

```javascript
const nuevosProductos = productos.map((producto) => {
  return {
    nombre: producto.nombre,
    precio: producto.precio,
    tipo: producto.tipo,
    talla: producto.talla,
    color: producto.color,
  };
});
```

Si un producto no tiene alguna de esas propiedades (por ejemplo, `Pan` no tiene `talla`), su valor en el nuevo objeto será `undefined`.

Otra forma de obtener un array nuevo (una copia superficial) es con `filter()` usando una condición que cumplan todos:

```javascript
const nuevosProductos = productos.filter(
  (producto) => producto.nombre.length > 0,
);
```

---

## 14. Ordenar arrays con `sort()`

`sort()` ordena el array **modificando el original** y recibe una función de comparación `(a, b)`:

- Si devuelve un número negativo, `a` va antes que `b`.
- Si devuelve un número positivo, `b` va antes que `a`.
- Si devuelve `0`, se mantiene el orden.

### Ordenar por número (precio de menor a mayor)

```javascript
nuevosProductos.sort((n1, n2) => n1.precio - n2.precio);
```

Para ordenar de mayor a menor, invertimos la resta: `n2.precio - n1.precio`.

Como `nuevosProductos` es un array nuevo, el array `productos` mantiene su orden. Aun así, los objetos de dentro son **los mismos** (copia superficial): si modificamos una propiedad de un objeto en uno de los arrays, el cambio se ve en ambos.

### Ordenar por texto (orden alfabético)

```javascript
productos.sort((n1, n2) => n1.nombre.localeCompare(n2.nombre));
console.log(productos);
```

`localeCompare()` compara cadenas teniendo en cuenta el idioma (tildes, `ñ`...) y devuelve un número negativo, positivo o `0`, justo lo que necesita `sort()`.

**Truco:** si no queremos modificar el original, podemos usar `toSorted()`, que devuelve un array nuevo ordenado.

---

## 15. Reducir un array a un solo valor con `reduce()`

`reduce()` recorre el array y va acumulando un resultado. Recibe:

1. Una función `(acumulador, elementoActual) => nuevoAcumulador`.
2. Opcionalmente, el valor inicial del acumulador.

### Media de un array de números

```javascript
const numeros = [5, 10, 15];

const media =
  numeros.reduce((acumulado, n) => acumulado + n, 0) / numeros.length;
console.log(media); // 10
```

### Total de unidades en stock

```javascript
const totalUnidades = productos.reduce((acc, p) => acc + p.stock, 0);
console.log(totalUnidades); // 295
```

### Valor total del inventario (precio × stock)

```javascript
const valorTodosLosProductos = productos.reduce(
  (acc, p) => acc + p.precio * p.stock,
  0,
);
console.log(valorTodosLosProductos);
```

### Producto más caro

Sin valor inicial, el acumulador empieza siendo el primer elemento. En cada vuelta nos quedamos con el más caro de los dos:

```javascript
const masCaro = productos.reduce((masCaro, producto) =>
  masCaro.precio > producto.precio ? masCaro : producto,
);
// { nombre: "Auriculares", precio: 49.99, ... }
```

### Alternativa: `Math.max()` con el operador spread

Si solo queremos el **precio** máximo (no el objeto entero):

```javascript
const precioMax = Math.max(...productos.map((p) => p.precio)); // 49.99
```

- `map()` crea un array solo con los precios.
- `...` (spread) "despliega" ese array en argumentos sueltos, porque `Math.max()` no acepta un array directamente.

**Importante:** pon siempre el valor inicial (`0`) cuando sumes; si el array está vacío y no hay valor inicial, `reduce()` lanza un error.

---

## 16. Copia profunda con `structuredClone()`

Copiar con `map()`, `filter()` o el spread (`[...array]`) crea una **copia superficial**: el array es nuevo, pero los objetos y arrays de dentro se comparten.

`structuredClone()` crea una **copia profunda**: todo es independiente.

```javascript
const arrayOriginal = [1, [2, 3], { nombre: "Ana" }];

// Crear la copia profunda
const copiaProfunda = structuredClone(arrayOriginal);

// Modificamos la copia para demostrar que son independientes
copiaProfunda[1][0] = 99;
copiaProfunda[2].nombre = "Carlos";

console.log(arrayOriginal); // [1, [2, 3], { nombre: "Ana" }] <- ¡El original no cambia!
console.log(copiaProfunda); // [1, [99, 3], { nombre: "Carlos" }]
```

---

## 17. JSON: `JSON.stringify()` y `JSON.parse()`

JSON es un formato de **texto** para guardar o enviar datos (por ejemplo, a un servidor o a `localStorage`).

```javascript
const producto = { nombre: "Taza", precio: 8, colores: ["blanco", "negro"] };

// Objeto → texto
const texto = JSON.stringify(producto);
console.log(texto); // '{"nombre":"Taza","precio":8,"colores":["blanco","negro"]}'
console.log(typeof texto); // "string"
console.log(texto.nombre); // undefined ← es texto, no tiene propiedades

// Bonito (con sangría de 2 espacios), útil para ver o guardar archivos
console.log(JSON.stringify(producto, null, 2));

// Texto → objeto
const deVuelta = JSON.parse(texto);
console.log(deVuelta.nombre); // "Taza"
```

- `JSON.stringify(valor)`: convierte un objeto/array en una cadena JSON.
- `JSON.stringify(valor, null, 2)`: igual, pero con saltos de línea y sangría.
- `JSON.parse(texto)`: convierte una cadena JSON de nuevo en objeto/array.

**Curiosidad:** `JSON.parse(JSON.stringify(obj))` también hace una copia profunda, aunque `structuredClone()` es la forma moderna y más fiable.

---

## Resumen de los métodos principales

| Método        | ¿Qué hace?                                                         | ¿Modifica el array original? |
| ------------- | ------------------------------------------------------------------ | ---------------------------- |
| `some()`      | Comprueba si al menos un elemento cumple una condición.            | No                           |
| `every()`     | Comprueba si todos los elementos cumplen una condición.            | No                           |
| `find()`      | Devuelve el primer elemento que cumple una condición.              | No                           |
| `findIndex()` | Devuelve el índice del primer elemento que cumple una condición.   | No                           |
| `filter()`    | Devuelve un array con los elementos que cumplen una condición.     | No                           |
| `map()`       | Devuelve un array con los resultados de transformar los elementos. | No                           |
| `includes()`  | Comprueba si existe un valor concreto.                             | No                           |
| `indexOf()`   | Devuelve el índice de un valor concreto o `-1`.                    | No                           |
| `splice()`    | Elimina, sustituye o añade elementos.                              | Sí                           |
| `toSpliced()` | Devuelve un array con los cambios solicitados.                     | No                           |
| `push()`      | Añade elementos al final.                                          | Sí                           |
| `unshift()`   | Añade elementos al principio.                                      | Sí                           |
| `pop()`       | Elimina y devuelve el último elemento.                             | Sí                           |
| `shift()`     | Elimina y devuelve el primer elemento.                             | Sí                           |
| `sort()`      | Ordena los elementos según una función de comparación.             | Sí                           |
| `toSorted()`  | Devuelve un array nuevo ordenado.                                  | No                           |
| `reduce()`    | Acumula todos los elementos en un único valor.                     | No                           |

**Otras herramientas vistas:** `structuredClone()` (copia profunda), `JSON.stringify()` / `JSON.parse()` (objeto ↔ texto) y `Math.max(...array)` (máximo de un array de números).

### Recomendación para estudiar

Para cada método, los alumnos deberían saber responder a tres preguntas:

1. ¿Qué hace?
2. ¿Qué devuelve?
3. ¿Modifica el array original?

Comprender estas tres diferencias ayuda a elegir el método adecuado en cada ejercicio de JavaScript.
