# Hashing con bcrypt, modelo Usuario y registro/login

Ayer pusiste un portero en la puerta: tu API ya no se traga datos basura. Hoy abres la parte de **autenticación**. El primer paso no es todavía iniciar sesión con tokens (eso viene después): es algo más básico y más importante: **guardar contraseñas sin guardarlas en claro**.

Al final del día tendrás un modelo `Usuario`, un endpoint de **registro** que guarda la contraseña cifrada, y un endpoint de **login** que comprueba la contraseña sin haberla guardado nunca en texto plano.

---

### Por qué nunca se guarda una contraseña tal cual

Imagina que guardas en tu base de datos `password: "1234"`. El día que alguien acceda a esa base de datos —un atacante, un empleado, una fuga— tiene las contraseñas de todos tus usuarios. Y como la gente reutiliza contraseñas, no solo has comprometido tu app: has comprometido su correo, su banco, todo.

La regla es absoluta: **una contraseña en claro nunca toca tu base de datos.** Ni en logs, ni en respuestas de la API, ni en ningún sitio.

### Hashing no es cifrado

Es la distinción clave del día. Mucha gente los confunde.

- **Cifrar** es reversible: con la clave correcta, recuperas el texto original. Sirve para datos que necesitas volver a leer.
- **Hashear** es de **una sola dirección**: conviertes el texto en una huella (un _hash_) de la que **no puedes volver atrás**. No existe el "deshasheo".

Una contraseña se **hashea**, no se cifra. Y entonces surge la pregunta lógica: si no puedo recuperar la contraseña original, ¿cómo compruebo en el login que el usuario ha metido la correcta?

La respuesta es elegante: cuando el usuario hace login, **hasheas lo que acaba de escribir y comparas los dos hashes**. Si coinciden, la contraseña es correcta. Nunca necesitaste conocer la original.

```
REGISTRO:  "1234"  →  hash  →  se guarda el hash en la BD
LOGIN:     "1234"  →  hash  →  ¿coincide con el hash guardado? → sí → acceso
```

### Qué aporta bcrypt: salt y coste

Un hash "a secas" tiene dos problemas, y bcrypt resuelve los dos:

**1. El salt.** Si dos usuarios tienen la contraseña `"1234"`, un hash normal daría el mismo resultado para ambos. Un atacante con una tabla de hashes precalculados (un _rainbow table_) los rompería de golpe. bcrypt añade a cada contraseña un **salt**: un valor aleatorio único. Así, dos contraseñas iguales producen hashes distintos, y las tablas precalculadas dejan de servir. Lo mejor: bcrypt **guarda el salt dentro del propio hash**, no tienes que gestionarlo tú.

**2. El coste (salt rounds).** bcrypt es **deliberadamente lento**. Cuando lo configuras con un coste de, por ejemplo, `10`, le estás diciendo cuántas veces repetir el proceso internamente. Para tu servidor, hashear una contraseña tarda una fracción de segundo: irrelevante. Para un atacante que quiere probar millones de contraseñas por segundo, esa lentitud lo hace inviable. Un valor de `10` es un buen estándar.

### Lo que necesitas de bcrypt

Solo dos funciones, y las dos son asíncronas (devuelven promesas, así que `await`):

```js
const bcrypt = require("bcrypt");

// Hashear (en el registro):
const hash = await bcrypt.hash(passwordEnClaro, 10);

// Comparar (en el login):
const coincide = await bcrypt.compare(passwordEnClaro, hashGuardado);
// coincide es true o false
```

`bcrypt.compare` hace por ti todo el trabajo: extrae el salt del hash guardado, hashea lo que le pasas con ese mismo salt, y compara. Tú solo recibes un `true` o `false`.

> Nota: existen dos paquetes, `bcrypt` y `bcryptjs`. El primero es más rápido pero compila código nativo; el segundo es JavaScript puro y no da problemas de instalación. Para clase usaremos `bcrypt`; si en tu máquina la instalación falla, `bcryptjs` tiene la misma API y solo cambias el `require`.

---

## Vamos a llevarlo al proyecto

Trabaja sobre tu proyecto. Commit después de cada bloque.

### Bloque 1 — Instalar y crear el modelo Usuario

#### 1.1 Instala bcrypt

```bash
npm install bcrypt
```

#### 1.2 El modelo

Crea `models/usuario.model.js` siguiendo el mismo patrón que tus otros modelos de Mongoose:

```js
const mongoose = require("mongoose");

const usuarioSchema = new mongoose.Schema(
  {
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    password: {
      type: String,
      required: true,
    },
  },
  { timestamps: true },
);

module.exports = mongoose.model("Usuario", usuarioSchema);
```

Fíjate en dos detalles:

- **`unique: true`** en el email: no queremos dos usuarios con el mismo correo. (Ojo: `unique` es un índice, no un validador; si ya tenías la colección creada, asegúrate de que el índice se aplica.)
- **El campo `password` guardará el hash, no la contraseña.** El nombre engaña: dentro nunca habrá una contraseña en claro.

> 💾 **Commit:** `feat: modelo Usuario`

---

### Bloque 2 — Registro: guardar la contraseña hasheada

#### 2.1 El controlador de registro

Crea `controllers/auth.controller.js` (o añádelo donde tengas tus controladores). Empezamos por el registro:

```js
const bcrypt = require("bcrypt");
const Usuario = require("../models/usuario.model");

const registro = async (req, res) => {
  try {
    const { email, password } = req.body;

    // 1. ¿Ya existe un usuario con ese email?
    const existe = await Usuario.findOne({ email });
    if (existe) {
      return res.status(409).json({ mensaje: "Ese email ya está registrado" });
    }

    // 2. Hashear la contraseña ANTES de guardar
    const hash = await bcrypt.hash(password, 10);

    // 3. Guardar el usuario con el hash, nunca con la contraseña en claro
    const nuevoUsuario = await Usuario.create({ email, password: hash });

    // 4. Responder SIN devolver el hash
    res.status(201).json({
      id: nuevoUsuario._id,
      email: nuevoUsuario.email,
    });
  } catch (error) {
    res.status(500).json({ mensaje: "Error en el registro" });
  }
};

module.exports = { registro };
```

Lee los cuatro pasos. El orden importa: **compruebas, hasheas, guardas, respondes.** Y en la respuesta **no devuelves el password**, ni siquiera hasheado: el cliente no tiene nada que hacer con él.

> ⚠️ El `409 Conflict` es el código correcto para "este recurso ya existe". No es un error de validación (no es un 422): los datos eran válidos, el problema es que el email ya está cogido.

#### 2.2 La ruta

Crea `routes/auth.routes.js`:

```js
const { Router } = require("express");
const { registro } = require("../controllers/auth.controller");

const router = Router();

router.post("/registro", registro);

module.exports = router;
```

Y móntala en tu app principal, con el prefijo que prefieras:

```js
const authRoutes = require("./routes/auth.routes");
app.use("/api/auth", authRoutes);
```

#### 2.3 Pruébalo

- `POST /api/auth/registro` con `{ "email": "test@test.com", "password": "1234" }` → **201**, te devuelve id y email.
- Repite la misma petición → **409**, email ya registrado.
- **Lo más importante:** abre tu base de datos (Atlas o Compass) y mira el documento. El campo `password` debe ser una cadena larga e ilegible tipo `$2b$10$...`, **no** `1234`. Si ves `1234`, algo va mal.

> 💾 **Commit:** `feat: endpoint de registro con hashing`

---

### Bloque 3 — Validar la entrada del registro

El registro recibe datos del usuario, así que merece validación, igual que hiciste ayer. Reutiliza el patrón del Día 27.

#### 3.1 Validadores de auth

Crea `validators/auth.validator.js`:

```js
const { body } = require("express-validator");

const reglasRegistro = [
  body("email").isEmail().withMessage("El email no es válido").normalizeEmail(),
  body("password")
    .isLength({ min: 6 })
    .withMessage("La contraseña debe tener al menos 6 caracteres"),
];

module.exports = { reglasRegistro };
```

#### 3.2 Conéctalos a la ruta

```js
const { reglasRegistro } = require("../validators/auth.validator");
const { validar } = require("../middlewares/validar");

router.post("/registro", reglasRegistro, validar, registro);
```

#### 3.3 Pruébalo

- Registro con email inválido o password de 3 caracteres → **422** con tus mensajes.
- Registro válido → sigue funcionando.

> 💾 **Commit:** `feat: validación del registro`

---

### Bloque 4 — Login: comprobar la contraseña

Ahora el otro lado. El login recibe email y password, busca al usuario, y compara la contraseña con `bcrypt.compare`.

#### 4.1 El controlador de login

En `auth.controller.js`:

```js
const login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // 1. Buscar al usuario por email
    const usuario = await Usuario.findOne({ email });
    if (!usuario) {
      return res.status(401).json({ mensaje: "Credenciales incorrectas" });
    }

    // 2. Comparar la contraseña recibida con el hash guardado
    const coincide = await bcrypt.compare(password, usuario.password);
    if (!coincide) {
      return res.status(401).json({ mensaje: "Credenciales incorrectas" });
    }

    // 3. Por ahora, login correcto. (Mañana: aquí devolveremos un token JWT.)
    res.status(200).json({
      mensaje: "Login correcto",
      id: usuario._id,
      email: usuario.email,
    });
  } catch (error) {
    res.status(500).json({ mensaje: "Error en el login" });
  }
};

module.exports = { registro, login };
```

#### 4.2 Un detalle de seguridad: el mismo mensaje para los dos casos

Fíjate en que respondemos **exactamente lo mismo** tanto si el email no existe como si la contraseña no coincide: `"Credenciales incorrectas"` con un **401**.

Esto es intencionado. Si dijeras "ese email no existe" en un caso y "contraseña incorrecta" en el otro, le estarías regalando a un atacante una forma de averiguar qué emails están registrados en tu sistema (eso se llama _enumeración de usuarios_). Mismo mensaje, mismo código, los dos casos. Un detalle pequeño que marca la diferencia entre un junior y alguien que piensa en seguridad.

#### 4.3 Ruta y validación

Añade las reglas de login (más simples: solo que vengan los campos) y la ruta:

```js
// en auth.validator.js
const reglasLogin = [
  body("email").isEmail().withMessage("Email no válido").normalizeEmail(),
  body("password").notEmpty().withMessage("La contraseña es obligatoria"),
];

module.exports = { reglasRegistro, reglasLogin };
```

```js
// en auth.routes.js
const { login } = require("../controllers/auth.controller");
const { reglasLogin } = require("../validators/auth.validator");

router.post("/login", reglasLogin, validar, login);
```

#### 4.4 Pruébalo

- Login con email y contraseña correctos → **200**, "Login correcto".
- Login con contraseña incorrecta → **401**, "Credenciales incorrectas".
- Login con un email que no existe → **401**, mismo mensaje exacto.

> 💾 **Commit:** `feat: endpoint de login con bcrypt.compare`

---

## Cierre del día

Comprueba con Postman:

- [ ] Registro guarda la contraseña **hasheada** (lo verificaste mirando la BD, no solo confiando).
- [ ] Registro de un email repetido → 409.
- [ ] Login correcto → 200.
- [ ] Login incorrecto (contraseña mal o email inexistente) → 401 con el **mismo** mensaje.
- [ ] En ninguna respuesta de la API aparece el password, ni en claro ni hasheado.

Hoy tienes registro y login funcionando, pero el login todavía no "recuerda" nada: cada petición es anónima. Mañana resolvemos eso con **JWT**: el login te devolverá un token que el cliente guardará y enviará en cada petición para demostrar quién es.

---

## 🚀 Reto opcional

1. **Hashear desde el modelo, no desde el controlador.** Investiga los _middleware_ de Mongoose (`usuarioSchema.pre("save", ...)`). ¿Podrías hacer que el hasheo ocurra automáticamente justo antes de guardar, de modo que el controlador ni se entere? ¿Qué ventaja tendría? ¿Qué cuidado hay que tener para no re-hashear un hash ya hasheado al actualizar?
2. **Ocultar el password por defecto.** Busca la opción `select: false` en el schema. Si la pones en el campo `password`, deja de venir en las consultas normales. ¿Qué tendrías que cambiar entonces en el login para poder comparar la contraseña?

---

## ✅ Soluciones del reto

### 1. Hashear desde el modelo con `pre("save")`

Un **middleware de Mongoose** es una función que se engancha a un evento del ciclo de vida del documento. `pre("save")` se ejecuta justo **antes** de escribir en Mongo, y dentro de él `this` es el documento que está a punto de guardarse, así que puedes reescribir sus campos:

```js
// usuario.model.js
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");

const usuarioSchema = new mongoose.Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true },
  },
  { timestamps: true },
);

usuarioSchema.pre("save", async function (next) {
  // Si el password NO ha cambiado en esta operación, ya está hasheado: no tocar.
  if (!this.isModified("password")) return next();

  try {
    this.password = await bcrypt.hash(this.password, 10);
    next();
  } catch (error) {
    next(error); // pasamos el error a Mongoose -> lo captura el catch del controlador
  }
});

module.exports = mongoose.model("Usuario", usuarioSchema);
```

> ⚠️ Fíjate en que es `function` y **no** una arrow function. Una arrow no tiene su propio `this`, así que `this.password` no apuntaría al documento.

Y el controlador se queda así: ya no importa `bcrypt` siquiera.

```js
// auth.controller.js (registro)
const nuevoUsuario = await Usuario.create({ email, password });
```

#### ¿Qué ventaja tiene?

- **Es imposible olvidarse.** Da igual desde dónde se cree un usuario (el controlador de registro, un script de seed, un test, un panel de admin que hagas en tres meses): el password **siempre** se guarda hasheado. Con el hash en el controlador, cada sitio nuevo tiene que acordarse de llamar a bcrypt, y el día que alguien se olvide tienes contraseñas en claro en la base de datos.
- **Una sola fuente de verdad.** Subir el coste de bcrypt de 10 a 12 es cambiar una línea en un archivo, no buscar `bcrypt.hash` por todo el proyecto.
- **El controlador recupera su trabajo real:** recibir datos, validar, guardar, responder. El *cómo* se protege una contraseña es un detalle del modelo, no del HTTP.

#### El cuidado: no re-hashear un hash

El hook se dispara en **cada** `save()`, no solo al crear. Si mañana haces esto:

```js
const usuario = await Usuario.findById(id);
usuario.email = "nuevo@mail.com";
await usuario.save(); // <- el hook corre otra vez
```

…sin la guarda, hashearías el hash que ya estaba guardado. El resultado es un hash válido de un string que nadie conoce: `bcrypt.compare` devolvería `false` para siempre y ese usuario **no podría volver a entrar nunca**. Y lo peor es que no verías ningún error: la operación tiene éxito, simplemente el login deja de funcionar.

La guarda lo evita:

```js
if (!this.isModified("password")) return next();
```

Mongoose lleva la cuenta de qué campos han cambiado en esta operación. Si `password` no está entre ellos, salimos del hook sin tocar nada.

> 🕳️ **Segunda trampa, importante:** `pre("save")` solo corre con `save()` y `create()`. `findOneAndUpdate`, `updateOne` y `updateMany` hablan directamente con la base de datos **sin instanciar un documento**, así que **no disparan el hook**. Si algún día haces un "cambiar contraseña", cárgalo y usa `save()`:
>
> ```js
> const usuario = await Usuario.findById(id);
> usuario.password = nuevaPassword;
> await usuario.save(); // sí hashea
> // await Usuario.findByIdAndUpdate(id, { password: nuevaPassword }); // ❌ guardaría en claro
> ```

### 2. Ocultar el password con `select: false`

```js
password: {
  type: String,
  required: true,
  select: false, // no viene en find / findOne / findById
},
```

A partir de ahí, las consultas normales devuelven el documento **sin** el campo `password`. Esto te protege del fallo más típico y más caro: hacer `res.json(usuario)` y filtrar el hash sin darte cuenta. Con `select: false`, aunque te equivoques, no hay nada que filtrar.

#### ¿Qué hay que cambiar en el login?

Si el campo no viene, `usuario.password` es `undefined` y la comparación ya no puede funcionar. Hay que **pedirlo explícitamente** con `+`:

```js
const usuario = await Usuario.findOne({ email }).select("+password");
```

El `+` significa "además de lo normal, tráeme también este campo". Y eso es precisamente lo bueno del patrón: el acceso al hash queda reducido a **una línea localizable** en todo el proyecto. Si alguien quiere saber dónde se lee la contraseña, lo encuentra con un `grep`.

#### Bonus: encapsular la comparación en el modelo

Si ya has movido el hasheo al modelo, tiene sentido mover también la comparación. Un **método de instancia**:

```js
// usuario.model.js
usuarioSchema.methods.compararPassword = function (passwordEnClaro) {
  return bcrypt.compare(passwordEnClaro, this.password);
};
```

```js
// auth.controller.js (login)
const usuario = await Usuario.findOne({ email }).select("+password");
if (!usuario) return res.status(401).json({ mensaje: "Credenciales incorrectas" });

const coincide = await usuario.compararPassword(password);
if (!coincide) return res.status(401).json({ mensaje: "Credenciales incorrectas" });
```

Ahora el controlador no sabe que existe bcrypt: solo pregunta "¿coincide esta contraseña?". Si mañana cambias bcrypt por argon2, el controlador no se entera.

> 💡 Los usuarios que ya tuvieras creados siguen funcionando igual: el hash guardado no cambia. Lo único que notarás es que si haces un `console.log(usuario)` el password aparece como `undefined` — eso es `select: false` haciendo su trabajo, no un bug.
