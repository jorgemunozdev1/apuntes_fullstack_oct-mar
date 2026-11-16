# Introducción al testing: tu primer test automático

Llevas semanas probando tu API de una sola forma: a mano, con ThunderClient. Cambias algo, abres ThunderClient, lanzas la petición, miras si responde lo que esperabas. Funciona… hasta que tu proyecto tiene veinte endpoints. Entonces, cada cambio te obligaría a re-probar todo a mano para asegurarte de que no rompiste nada por otro lado. Nadie lo hace. Y por ahí se cuelan los errores: arreglas el login y, sin enterarte, rompes el registro. Lo descubres en producción, que es el peor sitio posible.

Hoy empiezas a resolver eso con **tests automáticos**: código que prueba tu código. Lo escribes una vez y se ejecuta entero en un segundo, cada vez que quieras, gritándote en cuanto algo que antes funcionaba deja de funcionar. Al final del día tendrás Vitest instalado, un comando `npm test`, y tu primer test en verde. Pequeño, pero es la base de todo lo que viene.

---

### El problema: probar a mano no escala y no se acuerda

Probar con ThunderClient está bien para ver una cosa puntual. Pero tiene dos defectos fatales como red de seguridad:

- **No escala.** Probar tres endpoints a mano es rápido. Probar treinta, después de cada cambio, es media hora de clicar. Así que no lo haces, y dejas de comprobar cosas.
- **No se acuerda.** ThunderClient no recuerda qué probaste ayer. Un test automático sí: queda escrito, y se vuelve a ejecutar siempre, igual, sin que se te olvide ningún caso.

A esto último se le llama una **regresión**: algo que funcionaba y, por un cambio en otra parte, se rompe sin que te des cuenta. Los tests automáticos son tu defensa contra eso. Son código que ejercita tu código y avisa en el instante en que aparece la regresión, no tres semanas después con un usuario enfadado.

### Qué es un test

Un test no es nada exótico. Es **un programita que prepara una situación, ejecuta tu código y comprueba que el resultado es el que esperabas.** Si la comprobación cuadra, el test pasa (verde). Si no cuadra, falla (rojo) y te dice qué esperaba y qué recibió.

Tener tests te da tres cosas:

1. **Confianza para refactorizar.** Puedes reescribir un trozo y, si los tests siguen verdes, sabes que no rompiste el comportamiento.
2. **Red contra regresiones.** El día que rompas algo sin querer, un test se pone rojo antes de que llegue a producción.
3. **Documentación viva.** Un test bien escrito dice, en código, qué se supone que hace una función. Y, a diferencia de un comentario, **no puede mentir**: si el código cambia y el test no, el test falla.

### La pirámide del testing

No todos los tests son iguales. Se organizan en tres niveles, y casi siempre se dibujan como una pirámide:

```
        /\        e2e        (toda la app, como un usuario — lentos, poquísimos)
       /--\
      /    \      integración (varias piezas juntas, ej. un endpoint + la BD)
     /------\
    /        \    unitarios   (una pieza aislada — rapidísimos, muchísimos)
   /----------\
```

- **Unitarios:** prueban **una sola pieza aislada** (una función). Son rápidos y baratos, así que tienes muchos. Es la base.
- **Integración:** prueban **varias piezas trabajando juntas** —por ejemplo, una petición que entra por una ruta, pasa por el controlador y toca la base de datos—. Dan más confianza, pero son más lentos; tienes menos.
- **e2e (end-to-end):** prueban **la aplicación entera** como lo haría un usuario real. Los más realistas y los más lentos; tienes muy pocos.

La forma de pirámide no es casual: muchos tests baratos abajo, poquitos caros arriba. **Hoy empiezas por la base** (un test unitario) y en las próximas clases vas subiendo.

### La anatomía de un test: AAA

Da igual el nivel: todo test tiene los mismos tres momentos. Se conocen como **AAA**, y verlos te ayuda a escribir tests ordenados:

- **Arrange** (preparar): montas el escenario, preparas los datos de entrada.
- **Act** (actuar): ejecutas exactamente lo que quieres probar.
- **Assert** (afirmar): compruebas que el resultado es el esperado.

El **Assert** es el corazón del test: una comparación entre **lo que obtuve** y **lo que esperaba**. Si no coinciden, el test falla. Todo lo demás está al servicio de esa comparación.

### La herramienta: Vitest

Para escribir y ejecutar tests necesitas un _framework de testing_. Te da tres cosas:

- un **comando** que encuentra y ejecuta todos tus tests,
- funciones para **agrupar y describir** los tests (`describe`, `it`),
- las **afirmaciones**: `expect(loQueObtuve).toBe(loQueEsperaba)`.

Usaremos **Vitest**. ¿Por qué Vitest y no Jest, que quizá hayas oído más? Porque no necesita configuración: lo instalas y funciona, tanto con `require` como con `import`. Además es la herramienta del mundo de React, así que en M3 testearás el frontend con esto mismo. Su API es idéntica a la de Jest: lo que aprendas hoy te sirve en los dos.

---

Trabaja sobre tu proyecto. Commit después de cada bloque.

### Bloque 1 — Instalar Vitest y escribir el primer test

#### 1.1 Instala Vitest

```bash
npm install -D vitest
```

Fíjate en la `-D` (de `--save-dev`): Vitest es una **dependencia de desarrollo**. Lo usas para construir y probar, pero **no forma parte de la app que despliegas**. Tus usuarios no ejecutan tests; tú sí. Por eso va separada en el `package.json`, en `devDependencies`.

#### 1.2 El script de test

En tu `package.json`, en `scripts`, añade:

```json
"scripts": {
  "dev": "node --watch index.js",
  "test": "vitest"
}
```

_(Deja el `dev` o el que ya tuvieras; solo añade la línea de `test`.)_

Ahora `npm test` arrancará Vitest.

#### 1.3 Algo que probar: una función pura

No se puede testear el vacío. Vamos a empezar por lo más fácil de testear que existe: una **función pura**, una función que, con la misma entrada, siempre devuelve la misma salida, y no toca nada de fuera (ni la BD, ni `req`, ni la hora). No necesita preparación ni limpieza: le das algo, te devuelve algo, y compruebas.

Crea `utils/password.js` con una regla que ya conoces de la clase de validación: una contraseña es "segura" si tiene al menos 6 caracteres.

```js
const esPasswordSegura = (password) => {
  return password.length >= 6;
};

module.exports = { esPasswordSegura };
```

#### 1.4 El primer test

Al lado, crea `utils/password.test.js`. La terminación **`.test.js`** es la convención: Vitest busca automáticamente todos los archivos que acaban así.

```js
const { describe, it, expect } = require("vitest");
const { esPasswordSegura } = require("./password");

describe("esPasswordSegura", () => {
  it("devuelve true si la contraseña tiene 6 caracteres o más", () => {
    // Arrange: preparo la entrada
    const password = "123456";

    // Act: ejecuto lo que quiero probar
    const resultado = esPasswordSegura(password);

    // Assert: compruebo que es lo que esperaba
    expect(resultado).toBe(true);
  });
});
```

Lee las piezas:

- **`describe("esPasswordSegura", ...)`** agrupa todos los tests de esa función. Es una etiqueta para organizar.
- **`it("devuelve true si...", ...)`** es **un caso concreto**. La frase se lee de corrido: "it (ella, la función) devuelve true si la contraseña tiene 6 caracteres o más". Describe el comportamiento esperado en español llano.
- Las tres líneas del cuerpo son el **AAA** de la teoría, una por momento.
- **`expect(resultado).toBe(true)`** es el assert: "espero que `resultado` sea exactamente `true`". `toBe` es el matcher más básico, comprueba igualdad estricta (como `===`).

#### 1.5 Ejecútalo

```bash
npm test
```

Deberías ver algo en verde, un `✓` junto a tu test y un `Test Files 1 passed`. **Acabas de automatizar una comprobación que antes harías a mano.** Cada vez que ejecutes `npm test`, esa garantía se vuelve a verificar sola.

> 💾 **Commit:** `test: configurar Vitest y primer test`

---

### Bloque 2 — Ver el rojo antes que el verde

Un test que siempre pasa no vale nada: podría no estar comprobando lo que crees. Antes de fiarte de un test, **tienes que verlo fallar**. Es la única forma de saber que de verdad mide algo.

#### 2.1 Rómpelo a propósito

Cambia un momento el valor esperado a `false`:

```js
expect(resultado).toBe(false); // mentira, a propósito
```

Ejecuta `npm test`. Ahora sale **rojo**, y Vitest te enseña exactamente el problema:

```
expected true to be false
```

"Esperaba `false`, pero recibí `true`." Ese mensaje es oro: cuando un test real falla, esto es lo que te dice qué se rompió. Lee bien el formato `expected X to be Y`. Ahora vuelve a poner `true` y comprueba que está verde otra vez.

> La regla: **un test en el que no has visto el rojo no te da confianza.** Si nunca falla, no sabes si comprueba algo.

#### 2.2 El segundo caso: el otro lado

Una función tiene más de un comportamiento. Acabas de probar que una contraseña larga da `true`; ahora prueba que una corta da `false`. Añade un segundo `it` dentro del mismo `describe`:

```js
it("devuelve false si la contraseña tiene menos de 6 caracteres", () => {
  expect(esPasswordSegura("123")).toBe(false);
});
```

Aquí he juntado el AAA en una sola línea: cuando el caso es trivial, se puede. La estructura sigue ahí mentalmente (preparo `"123"`, ejecuto, afirmo), solo que más compacta.

Ejecuta: **dos tests en verde**. Tu función ya está probada por sus dos lados.

> 💾 **Commit:** `test: caso de contraseña corta`

---

### Bloque 3 — `toBe` vs `toEqual`: la trampa de los objetos

`toBe` te ha servido para `true`/`false` y números. Pero en cuanto pruebes objetos o arrays, te vas a estrellar con una trampa que pilla a todo el mundo una vez. Mejor verla hoy, controlada.

#### 3.1 Una función que devuelve un objeto

Crea en `utils/usuario.js` una función pura que recoge una idea que ya aplicaste en el login y el perfil: nunca devolver la contraseña. Esta función toma un usuario y devuelve solo su parte pública.

```js
const usuarioPublico = (usuario) => {
  return { id: usuario.id, email: usuario.email };
};

module.exports = { usuarioPublico };
```

#### 3.2 El test… que falla con `toBe`

Crea `utils/usuario.test.js`:

```js
const { describe, it, expect } = require("vitest");
const { usuarioPublico } = require("./usuario");

describe("usuarioPublico", () => {
  it("devuelve solo id y email, sin la contraseña", () => {
    const usuario = { id: "abc", email: "a@a.com", password: "hash_secreto" };

    const resultado = usuarioPublico(usuario);

    expect(resultado).toBe({ id: "abc", email: "a@a.com" }); // ⚠️ esto va a fallar
  });
});
```

Ejecútalo. **Falla**, aunque el objeto parece idéntico. El mensaje habla de identidad de referencia, no de contenido. ¿Por qué?

Porque `toBe` comprueba si son **el mismo objeto en memoria** (como `===`). Y dos objetos escritos por separado, aunque tengan el mismo contenido, **no son el mismo objeto**: viven en sitios distintos de la memoria. Para objetos, `toBe` casi nunca es lo que quieres.

#### 3.3 La solución: `toEqual`

Cambia `toBe` por `toEqual`:

```js
expect(resultado).toEqual({ id: "abc", email: "a@a.com" });
```

Ahora pasa. **`toEqual` compara el contenido en profundidad**, campo por campo, sin importar si es el mismo objeto. Graba la regla:

> **Primitivos** (números, strings, booleanos) → `toBe`. **Objetos y arrays** → `toEqual`.

Y fíjate en lo que el test deja claro de paso: en el resultado **no aparece `password`**. Es la misma idea de seguridad de la clase del login, ahora **blindada por un test**: si alguien tocara `usuarioPublico` y colara la contraseña, este test se pondría rojo al instante.

> 💾 **Commit:** `test: usuarioPublico oculta la contraseña`
