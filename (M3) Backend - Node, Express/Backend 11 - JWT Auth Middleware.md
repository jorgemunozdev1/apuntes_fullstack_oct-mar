# Proteger rutas con un middleware de autenticación

Ayer montaste las dos mitades del JWT: **firmar** en el login y **verificar** un token que llega. Pero esa verificación quedó suelta, copiada a mano dentro de una ruta de prueba (`quien-soy`). Si ahora quisieras proteger diez rutas, tendrías que copiar ese mismo bloque diez veces. Eso no escala, y lo sabes desde que sacaste la lógica de validación a un middleware.

Hoy conviertes esa verificación en un **middleware reutilizable**: un guardián que pones delante de cualquier ruta que quieras proteger. Comprueba el token una sola vez, en un único sitio, y de paso deja `req.usuario` listo para que el controlador sepa quién está pidiendo. Al final del día tendrás una ruta `GET /perfil` que solo responde si presentas un token válido, y las operaciones de escritura de tu CRUD cerradas a quien no haya iniciado sesión.

---

## Teoría

### El problema: no repitas el portero en cada controlador

Ayer, para saber quién pedía, leías la cabecera, separabas el `Bearer`, llamabas a `verify` y respondías. Todo dentro de un controlador. Si metieras ese mismo bloque en cada ruta protegida, tendrías el mismo código repetido por todas partes: el día que cambies algo (el mensaje de error, el nombre de la cabecera) tendrías que tocarlo en veinte sitios.

Ya resolviste exactamente este problema con la validación: la lógica de `validationResult` no vivía en cada controlador, la sacaste a `middlewares/validar.js`. Hoy haces lo mismo con la autenticación. **Mismo patrón, distinto portero.**

### Un middleware es un guardián delante del controlador

Recuerda lo que es un middleware: una función que se ejecuta **entre** que llega la petición y que corre tu controlador, con acceso a `req`, `res` y `next`. Ya conoces dos porteros:

- `validar` → el portero que comprueba que los **datos** tienen buena forma.
- Hoy: `verificarToken` → el portero que comprueba **quién eres** antes de dejarte pasar.

La diferencia con el `quienSoy` de ayer es solo el final de la función. Ayer, si el token era válido, **respondías** (`res.json`). Un middleware no responde: si todo va bien, **da paso** llamando a `next()`, y la petición continúa hacia el controlador. Si algo falla, **corta** la cadena con un 401 y no llama a `next()` — el controlador protegido no llega a ejecutarse nunca.

```
PETICIÓN → [ verificarToken ] → ¿token válido?
                                  │
                    sí → next() → [ controlador ] → respuesta
                    no → res.status(401)  ✗ (la cadena se corta aquí)
```

### La idea nueva del día: enriquecer `req`

Hasta ahora un middleware solo decidía si la petición pasaba o no. Hoy hace algo más: **deja información preparada para el siguiente eslabón.**

Cuando `verify` te devuelve el payload (que contiene el `id` del usuario), lo guardas en la propia petición:

```js
req.usuario = datos; // datos.id es el id que metiste al firmar
```

A partir de ese momento, cualquier controlador que venga **después** del middleware tiene `req.usuario` disponible. El controlador ya no pregunta "¿quién eres?": el guardián de la puerta ya lo averiguó y lo dejó apuntado. Esto es un patrón que vas a ver en todos los backends del mundo: el middleware de auth resuelve la identidad y la cuelga de `req` para que el resto la use.

### Autenticación no es lo mismo que autorización

Dos palabras que se parecen y se confunden constantemente:

- **Autenticación** (_authentication_): ¿**quién eres**? Es lo que hace el middleware de hoy: comprobar que presentas un token válido. Si no, **401 Unauthorized**.
- **Autorización** (_authorization_): vale, ya sé quién eres, pero ¿**puedes hacer esto**? Por ejemplo, ¿eres administrador para borrar usuarios? Si estás identificado pero no tienes permiso, el código correcto es **403 Forbidden**, no 401.

Hoy montas la autenticación. La autorización (roles y permisos) la dejamos apuntada como reto, pero quédate con la distinción: **401 = no sé quién eres; 403 = sé quién eres, pero no puedes.**

---

## Práctica

Trabaja sobre tu proyecto. Commit después de cada bloque.

### Bloque 1 — Crear el middleware `verificarToken`

Vas a coger la lógica de `quienSoy` de ayer y convertirla en un middleware. Es casi el mismo código; cambia solo el final.

#### 1.1 El archivo del middleware

Crea `middlewares/verificarToken.js`, al lado de tu `validar.js`:

```js
const jwt = require("jsonwebtoken");

const verificarToken = (req, res, next) => {
  try {
    // 1. Leer la cabecera Authorization
    const authHeader = req.headers.authorization;
    if (!authHeader) {
      return res.status(401).json({ mensaje: "No hay token" });
    }

    // 2. Separar "Bearer" del token
    const token = authHeader.split(" ")[1];

    // 3. Verificar (si es inválido o expiró, lanza error → catch)
    const datos = jwt.verify(token, process.env.JWT_SECRET);

    // 4. Dejar el usuario disponible para el controlador
    req.usuario = datos;

    // 5. Dar paso al siguiente eslabón de la cadena
    next();
  } catch (error) {
    return res.status(401).json({ mensaje: "Token inválido o expirado" });
  }
};

module.exports = { verificarToken };
```

Compáralo línea a línea con el `quienSoy` de ayer. Es idéntico hasta el paso 3. La diferencia está en el final:

- Donde ayer hacías `res.status(200).json({ ... })`, hoy haces `req.usuario = datos; next()`.
- Un middleware **no responde** cuando todo va bien: enriquece `req` y da paso.
- Fíjate en los `return` de los caminos de error: cortan la función ahí mismo, sin llegar a `next()`. Sin `next()`, el controlador protegido nunca se ejecuta. **Eso es el portero cerrando la puerta.**

> 💾 **Commit:** `feat: middleware de verificación de token`

---

### Bloque 2 — Proteger tu primera ruta: `GET /perfil`

Un middleware solo se entiende cuando lo pones a trabajar. Vamos a crear una ruta protegida de verdad: "dame los datos del usuario que ha iniciado sesión". Es el caso perfecto, porque el controlador necesita saber **quién** pide… y eso ya lo dejó preparado el middleware en `req.usuario`.

#### 2.1 El controlador del perfil

En `auth.controller.js`:

```js
const obtenerPerfil = async (req, res) => {
  try {
    // req.usuario lo dejó el middleware. Aquí ya sabemos quién pide,
    // sin que el cliente haya mandado email ni nada: solo su token.
    const usuario = await Usuario.findById(req.usuario.id).select("-password");

    if (!usuario) {
      return res.status(404).json({ mensaje: "Usuario no encontrado" });
    }

    res.status(200).json(usuario);
  } catch (error) {
    res.status(500).json({ mensaje: "Error al obtener el perfil" });
  }
};

module.exports = { obtenerPerfil };
```

Dos detalles importantes:

- El controlador **no toca la cabecera ni el token**. Esa parte es problema del middleware. Aquí solo lees `req.usuario.id` y vas a la base de datos. Cada cosa en su sitio.
- `.select("-password")` excluye el campo `password` de la respuesta. Nunca devuelves el hash de la contraseña, ni aunque esté hasheado: el cliente no tiene nada que hacer con él.

#### 2.2 La ruta protegida

En `auth.routes.js`, el guardián va **antes** del controlador:

```js
const { verificarToken } = require("../middlewares/verificarToken");
const { obtenerPerfil } = require("../controllers/auth.controller");

router.get("/perfil", verificarToken, obtenerPerfil);
```

Léelo en voz alta: "para ver el perfil, primero verifica el token, y si pasa, obtén el perfil". El orden no es decorativo: si `verificarToken` corta con un 401, `obtenerPerfil` no llega a ejecutarse.

#### 2.3 Pruébalo

1. Haz login y copia el `token`.
2. `GET /api/auth/perfil` con la cabecera `Authorization: Bearer <tu-token>` → **200** con tus datos (email, fechas… **sin** el password).
3. Ahora rómpelo:
   - Sin la cabecera `Authorization` → **401** "No hay token". El controlador no se ejecutó.
   - Cambia un carácter del token → **401** "Token inválido o expirado".

Has reconocido a un usuario sin que mande email ni contraseña: solo la pulsera del festival. Y el controlador no ha tenido que saber nada de tokens.

> 💾 **Commit:** `feat: ruta protegida GET /perfil`

---

### Bloque 3 — Reutilizar el guardián en tu CRUD

Aquí es donde el trabajo de los bloques anteriores se paga: un middleware se escribe una vez y se reutiliza en cuantas rutas quieras. Vamos a cerrar las operaciones de **escritura** de tu CRUD para que solo un usuario identificado pueda crear, editar o borrar.

#### 3.1 Decide qué proteger

Una decisión de diseño muy común: dejar las **lecturas públicas** y proteger las **escrituras**. Cualquiera puede ver el catálogo, pero solo un usuario logueado puede modificarlo. Tú ajústalo a tu proyecto, pero el patrón típico es este.

En el router de tu recurso (`recurso.routes.js`), importa el middleware y ponlo delante de `POST`, `PUT` y `DELETE`:

```js
const { verificarToken } = require("../middlewares/verificarToken");

// Lectura: abierta
router.get("/", obtenerTodos);
router.get("/:id", obtenerUno);

// Escritura: protegida (el guardián va el primero de la cadena)
router.post("/", verificarToken, reglasCrear, validar, crearRecurso);
router.put("/:id", verificarToken, reglasActualizar, validar, actualizarRecurso);
router.delete("/:id", verificarToken, eliminarRecurso);
```

Fíjate en la cadena del `POST`: **verifica quién eres → valida los datos → crea**. Tres porteros en fila, cada uno con su trabajo, y el controlador al final ya limpio de todo eso. Esto es exactamente lo que buscabas: identidad, datos y lógica de negocio, separados en capas.

#### 3.2 Pruébalo

- `POST` / `PUT` / `DELETE` **sin** token → **401**. No se toca la base de datos.
- Las mismas operaciones **con** un token válido en la cabecera → funcionan como antes.
- `GET` sigue funcionando sin token (lo dejaste público).

> 💾 **Commit:** `feat: proteger las rutas de escritura del CRUD`

---

### Bloque 4 — Hacer el middleware robusto

Tu middleware funciona con tokens buenos y con tokens rotos. Pero hay un par de casos feos que conviene cubrir, y de paso afinamos los mensajes. Esto recoge los dos retos que te dejaste ayer.

#### 4.1 El caso `Authorization: Patata`

¿Qué pasa si alguien manda la cabecera pero sin la estructura `Bearer token`? Con `authHeader.split(" ")[1]`, el token sale `undefined`, y `verify` revienta. Acaba en el `catch` y devuelve 401, así que **no es un fallo grave**… pero es más limpio cazar el formato incorrecto antes de intentar verificar nada.

#### 4.2 Distinguir "token inválido" de "sesión caducada"

Cuando `verify` falla, no siempre es por lo mismo. La librería lanza errores con `name` distinto:

- `TokenExpiredError` → el token era válido pero **caducó** (pasó el `expiresIn`).
- `JsonWebTokenError` → el token está **manipulado o mal formado**.

Para el usuario no es lo mismo "tu sesión ha caducado, vuelve a entrar" que "token inválido". Aprovechemos esa diferencia. La versión robusta del middleware:

```js
const jwt = require("jsonwebtoken");

const verificarToken = (req, res, next) => {
  const authHeader = req.headers.authorization;

  // 1. ¿Viene la cabecera y tiene el formato "Bearer token"?
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res
      .status(401)
      .json({ mensaje: "No hay token o el formato es incorrecto" });
  }

  // 2. Extraer el token
  const token = authHeader.split(" ")[1];

  // 3. Verificar, distinguiendo por qué falla
  try {
    const datos = jwt.verify(token, process.env.JWT_SECRET);
    req.usuario = datos;
    next();
  } catch (error) {
    if (error.name === "TokenExpiredError") {
      return res
        .status(401)
        .json({ mensaje: "Tu sesión ha caducado, vuelve a iniciar sesión" });
    }
    return res.status(401).json({ mensaje: "Token inválido" });
  }
};

module.exports = { verificarToken };
```

#### 4.3 Pruébalo

- `Authorization: Patata` (sin `Bearer`) → **401** "formato incorrecto", sin pasar por `verify`.
- Token manipulado → **401** "Token inválido".
- Caducidad real: cambia el `expiresIn` del login a `"10s"`, haz login, espera quince segundos y reintenta → **401** "Tu sesión ha caducado". _(Acuérdate de volver a dejar el `expiresIn` como lo tenías.)_

> 💾 **Commit:** `refactor: middleware de token más robusto`

---

### Limpieza final

La ruta de prueba `quien-soy` de ayer ya no tiene sentido: era el andamio para entender el mecanismo, y ese mecanismo ahora vive en el middleware. Bórrala (la ruta en `auth.routes.js` y el controlador `quienSoy` en `auth.controller.js`).

> 💾 **Commit:** `chore: eliminar ruta de prueba quien-soy`

---

## Cierre del día

Comprueba con Postman:

- [ ] `GET /perfil` con token válido → 200 con tus datos, **sin** el password.
- [ ] `GET /perfil` sin token o con token roto → 401, y el controlador no se ejecuta.
- [ ] Las escrituras de tu CRUD piden token; las lecturas siguen abiertas.
- [ ] El middleware deja `req.usuario` disponible y el controlador lo usa (`req.usuario.id`).
- [ ] Token caducado y token inválido dan mensajes distintos.

Hoy has cerrado el arco de autenticación: registro (bcrypt), login (token), y protección de rutas (este middleware). Tu API ya distingue entre quién puede pasar y quién no, y lo hace en un único sitio reutilizable. Lo que falta para una auth completa es la otra mitad de la moneda: la **autorización** —no solo "quién eres", sino "qué tienes permiso de hacer"—, y ahí es donde entran los roles.

---

## 🚀 Reto opcional

1. **Roles y autorización (el 403).** Añade un campo `rol` al modelo `Usuario` (por ejemplo `"usuario"` o `"admin"`, con un `default`). Inclúyelo en el payload al firmar el token. Después escribe un **segundo** middleware, `verificarRol("admin")`, que se ejecute _después_ de `verificarToken` y mire `req.usuario.rol`: si no es el rol exigido, responde **403 Forbidden** (no 401: el usuario está identificado, simplemente no tiene permiso). Protege con él alguna ruta sensible, como el `DELETE`. Pista: `verificarRol` no es un middleware directo, es una función que **devuelve** un middleware.
2. **¿Y si el usuario ya no existe?** Un token puede estar perfectamente firmado y sin caducar, pero apuntar a un usuario que fue borrado de la base de datos. Ahora mismo tu middleware confiaría en él. Haz que `verificarToken` cargue el usuario real (`Usuario.findById(req.usuario.id)`) y, si no existe, responda 401. Engancha el documento entero en `req.usuario`. ¿Qué ganas con esto? ¿Qué pagas? (Pista: una consulta a la base de datos en **cada** petición protegida. ¿Compensa siempre?)
