# Express-validator

## Por qué validamos en el backend

Hasta ahora tu API acepta lo que le mandes. Si haces un `POST` con el body vacío, con un campo de más, o con un email que es `"patata"`, tu servidor lo coge sin rechistar y lo intenta guardar. Eso es una puerta abierta.

La regla que no se te puede olvidar: **nunca confíes en el cliente.** Aunque el formulario del frontend valide, ese formulario lo controla el usuario. Puede desactivar el JavaScript, puede mandarte peticiones con Postman, puede tener un bug. La validación del frontend es comodidad para el usuario; la validación del backend es seguridad para tu sistema. Las dos existen y no se sustituyen.

## ¿Y Mongoose no validaba ya?

Sí, tu schema de Mongoose tiene `required`, tipos, a veces `min`/`max`. Entonces, ¿por qué validar otra vez antes?

Por dos motivos:

1. **Control de los errores.** Cuando Mongoose rechaza algo, lanza un error con un formato suyo, en el momento de tocar la base de datos. Si validas antes, en la capa HTTP, decides tú el formato del error, el código de estado y el mensaje, y lo haces _antes_ de gastar una llamada a la base de datos.
2. **Separación de responsabilidades.** Mongoose protege la **integridad de los datos** (que lo que se guarde tenga sentido). express-validator protege la **entrada de la petición** (que lo que llega por HTTP tenga la forma que esperas). Son dos murallas distintas, y tener las dos no es redundancia inútil: es defensa en capas.

Piénsalo así: Mongoose es el portero de la base de datos. express-validator es el portero de la puerta de la calle. Que haya portero dentro no significa que dejes la puerta de la calle abierta.

## Cómo funciona express-validator

express-validator se apoya en una idea que ya conoces: **el middleware**. Recuerda lo que vimos: un middleware es una función que se ejecuta entre que llega la petición y que se ejecuta tu controlador, y tiene acceso a `req`, `res` y `next`.

express-validator funciona en dos tiempos:

### 1. Las reglas (validation chains)

Para cada campo defines una cadena de reglas. Se leen casi como una frase:

```js
body("email").isEmail().withMessage("El email no es válido");
```

Esto dice: "del body, coge el campo `email`, comprueba que es un email, y si no lo es, usa este mensaje". Cada regla es un middleware que se acumula en la petición, pero **todavía no corta nada**: solo va apuntando los problemas que encuentra.

Validadores que vas a usar constantemente:

| Validador                 | Qué comprueba                                             |
| ------------------------- | --------------------------------------------------------- |
| `.notEmpty()`             | que el campo no venga vacío                               |
| `.isLength({ min, max })` | longitud de un texto                                      |
| `.isEmail()`              | formato de email                                          |
| `.isInt()` / `.isFloat()` | que sea número entero / decimal                           |
| `.isBoolean()`            | que sea `true`/`false`                                    |
| `.optional()`             | que el campo puede no venir (y si no viene, no se valida) |

Y un par de funciones de la familia (`body`, `param`, `query`) según de dónde saques el dato:

- `body("campo")` → del cuerpo de la petición (lo normal en POST y PUT).
- `param("id")` → de la URL (`/usuarios/:id`).
- `query("page")` → de la query string (`?page=2`).

### 2. La comprobación del resultado

Las reglas solo _apuntan_ los errores. Para _actuar_ necesitas leer el resultado con `validationResult(req)`. Esto va en un middleware propio, después de las reglas:

```js
const { validationResult } = require("express-validator");

const validar = (req, res, next) => {
  const errores = validationResult(req);
  if (!errores.isEmpty()) {
    return res.status(422).json({ errores: errores.array() });
  }
  next();
};

module.exports = { validar };
```

Si hay errores, cortas la petición con un **422** y devuelves la lista. Si no los hay, llamas a `next()` y la petición sigue su camino hacia el controlador.

## ¿Por qué un 422 y no un 400?

Los dos son errores del cliente (familia 4xx) y los dos son aceptables. La diferencia fina:

- **400 Bad Request**: la petición está mal formada, el servidor no la entiende.
- **422 Unprocessable Entity**: el servidor entiende perfectamente la petición, pero los datos no pasan tus reglas de negocio.

Cuando el body es un JSON correcto pero el email no es válido, el servidor _entendió_ la petición; simplemente no le gustan los datos. Eso es un 422. Es el código más preciso para validación, y usarlo bien es un detalle que distingue a un junior que sabe lo que hace. Si en algún sitio ves usar 400 para esto, tampoco es un error grave; es una convención.

## Dónde colocamos las reglas

Tienes dos opciones:

1. Meter las reglas directamente en el router, pegadas a cada ruta.
2. Sacarlas a un archivo de validadores por recurso (`validators/usuario.validator.js`) y que el router solo las importe.

Vamos a hacer la **segunda**. En la práctica de hoy verás por qué: el router queda limpio y legible, las reglas se reutilizan, y cuando crezca el proyecto sabes exactamente dónde está cada cosa. Es el mismo principio que ya aplicaste al separar el modelo y el controlador en el MVC: **cada cosa en su sitio.**

## Resumen

- Valida siempre en el backend, aunque el frontend ya valide. Nunca confíes en el cliente.
- Mongoose y express-validator no se pisan: protegen capas distintas (la base de datos y la entrada HTTP).
- express-validator trabaja en dos tiempos: primero las **reglas** (que solo apuntan), después un middleware con **`validationResult`** (que actúa).
- Errores de validación → **422** con la lista de problemas.
- Las reglas van en **archivos de validadores por recurso**, no sueltas en el router.

# Parte Práctica

Hoy vas a coger la API que ya tienes (CRUD + MVC + Mongoose) y le vas a poner un portero en la puerta. Al final del día, tu API rechazará datos inválidos con mensajes claros y un código de estado correcto, sin llegar a tocar la base de datos con basura.

Trabaja sobre tu proyecto actual. Cada bloque termina con un commit: no esperes al final para guardar.

---

## Punto de partida

Antes de empezar, comprueba que tu API funciona como la dejaste:

- Levanta el servidor (`npm run dev` o el script que tengas).
- Haz un `POST` con Postman creando un recurso válido. Debe funcionar.
- Ahora haz un `POST` con el **body vacío**. Fíjate en qué pasa: probablemente te dé un error feo de Mongoose, o guarde algo a medias. **Ese es el problema que vamos a resolver hoy.**

---

## Bloque 1 — Instalar y primer validador inline

### 1.1 Instala el paquete

```bash
npm install express-validator
```

### 1.2 Un validador rápido, directo en el router

Solo para ver el mecanismo funcionando, vamos a poner una regla suelta en tu ruta `POST`. Abre tu archivo de rutas e importa lo que necesitas:

```js
const { body, validationResult } = require("express-validator");
```

Añade una regla a tu ruta de crear (ajusta el nombre del campo a tu recurso real):

```js
router.post(
  "/",
  body("nombre").notEmpty().withMessage("El nombre es obligatorio"),
  (req, res) => {
    const errores = validationResult(req);
    if (!errores.isEmpty()) {
      return res.status(422).json({ errores: errores.array() });
    }
    // ... aquí seguía tu controlador
  },
);
```

### 1.3 Pruébalo

- `POST` con `nombre` relleno → debe seguir funcionando.
- `POST` con el `nombre` vacío o sin enviarlo → debes recibir un **422** con tu mensaje.

**Checkpoint:** consigues un 422 con el mensaje de error. Ya tienes el mecanismo en marcha.

> **Commit:** `feat: primer validador con express-validator en POST`

---

## Bloque 2 — Sacar la lógica de errores a un middleware

Ese bloque de `validationResult` no puede vivir dentro de cada controlador: lo repetirías en todas las rutas. Lo sacamos a un middleware reutilizable.

### 2.1 Crea el middleware

Crea `middlewares/validar.js` (o el nombre/carpeta que uses para middlewares):

```js
const { validationResult } = require("express-validator");

const validar = (req, res, next) => {
  const errores = validationResult(req);
  if (!errores.isEmpty()) {
    return res.status(422).json({ errores: errores.array() });
  }
  next();
};

module.exports = { validar };
```

### 2.2 Úsalo en la ruta

Ahora tu controlador vuelve a estar limpio, y la comprobación de errores es un middleware más en la cadena:

```js
const { validar } = require("../middlewares/validar");

router.post(
  "/",
  body("nombre").notEmpty().withMessage("El nombre es obligatorio"),
  validar, // <- middleware: corta aquí si hay errores
  crearRecurso, // <- tu controlador, ya sin lógica de validación
);
```

### 2.3 Pruébalo otra vez

Mismo comportamiento que antes, pero ahora tu controlador no sabe nada de validación. Comprueba que el 422 sigue saliendo.

**Checkpoint:** el controlador ya no contiene `validationResult`. La lógica de error está en un único sitio.

> **Commit:** `refactor: extraer manejo de errores de validación a middleware`

---

## Bloque 3 — Validadores por recurso (el patrón profesional)

Las reglas tampoco deberían vivir sueltas en el router. Las agrupamos por recurso en su propio archivo, igual que separaste modelo y controlador.

### 3.1 Crea el archivo de validadores

Crea `validators/recurso.validator.js` (sustituye `recurso` por tu entidad real: `usuario`, `producto`, `tarea`...). Exporta un **array de reglas**:

```js
const { body } = require("express-validator");

const reglasCrear = [
  body("nombre")
    .notEmpty()
    .withMessage("El nombre es obligatorio")
    .isLength({ min: 3 })
    .withMessage("Mínimo 3 caracteres"),

  // Añade aquí una regla por cada campo que tu recurso necesite.
  // Ejemplos según tus campos reales:
  // body("email").isEmail().withMessage("Email no válido"),
  // body("precio").isFloat({ min: 0 }).withMessage("El precio debe ser un número positivo"),
  // body("stock").isInt({ min: 0 }).withMessage("El stock debe ser un entero >= 0"),
];

module.exports = { reglasCrear };
```

### 3.2 El router queda limpio

```js
const { reglasCrear } = require("../validators/recurso.validator");
const { validar } = require("../middlewares/validar");

router.post("/", reglasCrear, validar, crearRecurso);
```

Lee esa línea en voz alta: "para crear, aplica las reglas de crear, valida, y si todo va bien crea el recurso". Se entiende sola. **Eso es código limpio.**

### 3.3 Define las reglas de TODOS los campos de tu recurso

Mira tu schema de Mongoose y, para cada campo importante, escribe su regla de validación. Piensa qué hace que un dato sea válido en tu dominio: ¿qué campos son obligatorios?, ¿qué longitudes tienen sentido?, ¿qué debe ser número y no texto?

**Checkpoint:** tu `POST` valida todos los campos relevantes del recurso, y el router está limpio.

> **Commit:** `feat: validadores por recurso para la creación`

---

## Bloque 4 — Validar el PUT y el :id

Crear no es lo único que recibe datos. El `PUT` también, y además trabaja con un `id` en la URL que conviene comprobar.

### 4.1 Reglas para actualizar

En tu `recurso.validator.js`, añade un segundo array. Al actualizar, a veces el usuario manda solo algunos campos, así que aquí `.optional()` es tu amigo:

```js
const { body, param } = require("express-validator");

const reglasActualizar = [
  param("id").isMongoId().withMessage("El id no es válido"),

  body("nombre")
    .optional()
    .notEmpty()
    .withMessage("El nombre no puede ir vacío")
    .isLength({ min: 3 })
    .withMessage("Mínimo 3 caracteres"),

  // ...el resto de campos, todos con .optional() en un PUT parcial
];

module.exports = { reglasCrear, reglasActualizar };
```

Fíjate en `param("id").isMongoId()`: como ahora usas MongoDB, los id son strings con un formato concreto. Si alguien pide `PUT /recurso/patata`, lo cazas **antes** de ir a la base de datos, en lugar de esperar a que Mongoose explote.

### 4.2 Conéctalo

```js
const { reglasActualizar } = require("../validators/recurso.validator");

router.put("/:id", reglasActualizar, validar, actualizarRecurso);
```

### 4.3 Pruébalo

- `PUT` con un id mal formado (`/recurso/123`) → **422**, sin tocar la BD.
- `PUT` con un id bien formado pero inexistente → aquí express-validator ya no llega: el formato es correcto, así que pasa, y es tu **controlador** quien debe responder 404 si no encuentra el documento. Comprueba que ese 404 sigue funcionando como lo tenías.

> **Trampa importante:** que un id tenga el formato correcto **no** significa que exista en la base de datos. express-validator valida la _forma_; la _existencia_ la comprueba tu controlador con un 404. No mezcles las dos cosas.

**Checkpoint:** `PUT` valida el id y los campos opcionales; el 404 de "no encontrado" lo sigue dando el controlador.

> **Commit:** `feat: validación de PUT y del parámetro id`

---

## Bloque 5 — Pulir los mensajes de error

Haz un `POST` con varios campos mal a la vez y mira la respuesta. Vas a ver que `errores.array()` devuelve bastante información por cada error (campo, valor recibido, ubicación...).

Para un frontend, a veces interesa una forma más limpia. Mejora tu middleware `validar.js` para devolver solo lo útil:

```js
const validar = (req, res, next) => {
  const errores = validationResult(req);
  if (!errores.isEmpty()) {
    return res.status(422).json({
      errores: errores.array().map((e) => ({
        campo: e.path,
        mensaje: e.msg,
      })),
    });
  }
  next();
};

module.exports = { validar };
```

Ahora la respuesta es un array de `{ campo, mensaje }`, perfecto para que un frontend pinte los errores junto a cada input. Pruébalo con varios errores a la vez.

**Checkpoint:** tus errores salen como una lista limpia de campo + mensaje.

> **Commit:** `refactor: formato limpio de errores de validación`

---

## Reto opcional

1. **Sanitización**, no solo validación. Investiga `.trim()` y `.normalizeEmail()` y añádelos a tus reglas. ¿Qué diferencia hay entre _validar_ (comprobar) y _sanitizar_ (limpiar/transformar) un dato? Pista: la sanitización modifica el valor antes de que llegue al controlador.
2. **Validación personalizada** con `.custom()`. Escribe una regla que no exista de serie: por ejemplo, que un campo `password` y un campo `repetirPassword` coincidan. Busca en la documentación cómo recibe `.custom()` el resto del body.
