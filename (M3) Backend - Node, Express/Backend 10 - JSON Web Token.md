# Autenticación con JWT: firma y verificación de tokens

Ayer dejaste el login funcionando, pero con un cabo suelto: cuando el usuario inicia sesión correctamente, tu API le dice "login correcto"… y se olvida de él al instante. La siguiente petición vuelve a ser anónima. El servidor no recuerda quién es.

Hoy resolvemos eso. Al final del día, tu login devolverá un **token JWT** que identifica al usuario, y tendrás una forma de **verificar** ese token cuando el cliente lo presente de vuelta. Mañana usarás esa verificación para proteger rutas.

---

## Teoría

### El problema: HTTP no tiene memoria

HTTP es **sin estado** (_stateless_): cada petición es independiente y el servidor no recuerda nada de la anterior. Esto es bueno para escalar, pero plantea una pregunta: si cada petición empieza de cero, ¿cómo sabe la API que la persona que pide `GET /mis-datos` es la misma que hizo login hace un minuto?

La respuesta clásica es: el cliente tiene que **demostrar quién es en cada petición**. Necesita una especie de pulsera de un festival: te la ponen al entrar (login) y la enseñas cada vez que quieres pasar a una zona.

Esa pulsera es el **token**.

### Qué es un JWT

JWT son las siglas de **JSON Web Token**. Es una cadena de texto que el servidor genera al hacer login y entrega al cliente. El cliente la guarda y la envía en cada petición posterior. Cuando el servidor la recibe, comprueba que es auténtica y sabe quién eres.

Un JWT tiene tres partes separadas por puntos:

```
xxxxx.yyyyy.zzzzz
└─┬─┘ └─┬─┘ └─┬─┘
header payload firma
```

- **Header**: metadatos, como el algoritmo de firma usado.
- **Payload**: los datos que metes dentro (por ejemplo el id del usuario). Se llaman _claims_.
- **Firma**: la parte que garantiza que el token no ha sido manipulado.

### El detalle que todo el mundo malentiende: un JWT NO está cifrado

Esto es importante y casi nadie lo tiene claro al principio: **el payload de un JWT no es secreto.** Las dos primeras partes son simplemente texto codificado en Base64. Cualquiera puede copiar un token, pegarlo en [jwt.io](https://jwt.io) y leer lo que hay dentro.

Entonces, ¿de qué sirve? De la **firma**. El servidor firma el token con una clave secreta que solo él conoce. Si alguien modifica el payload (por ejemplo, cambia `id: 5` por `id: 1` para hacerse pasar por otro), la firma deja de cuadrar y el servidor rechaza el token.

La consecuencia práctica, que debes grabarte:

> **Un JWT garantiza que el contenido no ha sido alterado, pero NO lo oculta. Nunca metas información sensible (contraseñas, datos privados) en el payload.** Solo lo justo para identificar al usuario, normalmente su id.

### Firmar y verificar

El servidor hace dos operaciones, con la misma clave secreta:

- **Firmar (`sign`)**: en el login, genera el token metiendo el id del usuario y firmándolo con el secreto.
- **Verificar (`verify`)**: cuando llega un token, comprueba con el secreto que la firma es válida y, si lo es, recupera el payload.

```
LOGIN:        id del usuario + SECRETO  →  sign  →  token  →  se manda al cliente
PETICIÓN:     token + SECRETO           →  verify →  ¿firma válida? → sí → recuperas el id
```

Si el secreto del `verify` no coincide con el del `sign`, o si el token fue manipulado, `verify` lanza un error y el token se rechaza.

### El secreto y la expiración

Dos decisiones de configuración:

- **El secreto** es una cadena larga y aleatoria que **vive en tu `.env`**, nunca en el código ni en GitHub. Si tu secreto se filtra, cualquiera puede firmar tokens válidos y hacerse pasar por quien quiera. Es tan sensible como una contraseña de base de datos.
- **La expiración (`expiresIn`)**: un token no debería valer para siempre. Le pones una caducidad (por ejemplo, `"1h"` o `"7d"`). Cuando expira, el usuario tiene que volver a hacer login. Esto limita el daño si un token es robado.

### La librería: jsonwebtoken

Te hará falta una sola librería y dos funciones:

```js
const jwt = require("jsonwebtoken");

// Firmar (en el login):
const token = jwt.sign(
  { id: usuario._id }, // payload: solo lo necesario
  process.env.JWT_SECRET, // el secreto del .env
  { expiresIn: "1h" }, // opciones
);

// Verificar (cuando llega un token):
const datos = jwt.verify(token, process.env.JWT_SECRET);
// datos contiene el payload: datos.id es el id del usuario
// si el token es inválido o expiró, verify LANZA un error → try/catch
```

---

## Práctica

Trabaja sobre tu proyecto. Commit después de cada bloque.

### Bloque 1 — Instalar y configurar el secreto

#### 1.1 Instala la librería

```bash
npm install jsonwebtoken
```

#### 1.2 El secreto en el `.env`

Abre tu `.env` y añade una clave secreta. Que sea larga y aleatoria, no `"secreto123"`:

```
JWT_SECRET=una_cadena_larga_y_aleatoria_que_solo_conoce_el_servidor
```

> ⚠️ Comprueba que tu `.env` está en el `.gitignore`. Este secreto **nunca** se sube a GitHub. Si alguna vez se filtra, cámbialo: a partir de ese momento cualquiera podría firmar tokens válidos.

Para generar un secreto de verdad bueno, puedes usar en la terminal:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

> 💾 **Commit:** `chore: variable de entorno para JWT` _(recuerda: el `.env` NO se commitea, solo el código que lo usa)_

---

### Bloque 2 — Firmar el token en el login

Vuelve a tu controlador de login. Lo dejaste respondiendo "login correcto". Ahora, en ese mismo punto donde la contraseña ya cuadró, generas y devuelves el token.

#### 2.1 Modifica el login

```js
const jwt = require("jsonwebtoken");
// ...el resto de requires que ya tenías (bcrypt, Usuario)

const login = async (req, res) => {
  try {
    const { email, password } = req.body;

    const usuario = await Usuario.findOne({ email });
    if (!usuario) {
      return res.status(401).json({ mensaje: "Credenciales incorrectas" });
    }

    const coincide = await bcrypt.compare(password, usuario.password);
    if (!coincide) {
      return res.status(401).json({ mensaje: "Credenciales incorrectas" });
    }

    // NUEVO: la contraseña cuadra → firmamos un token con el id del usuario
    const token = jwt.sign({ id: usuario._id }, process.env.JWT_SECRET, {
      expiresIn: "1h",
    });

    res.status(200).json({
      mensaje: "Login correcto",
      token,
    });
  } catch (error) {
    res.status(500).json({ mensaje: "Error en el login" });
  }
};

module.exports = { registro, login };
```

Fíjate en qué metemos en el payload: **solo el id**. Ni el email, ni nada sensible. El servidor solo necesita saber _quién_ eres; el resto lo busca en la base de datos cuando haga falta.

#### 2.2 Pruébalo

- `POST /api/auth/login` con credenciales correctas → **200** con un campo `token` que es una cadena larga con dos puntos dentro.
- Copia ese token, pégalo en [jwt.io](https://jwt.io) y mira el payload. **Vas a ver tu id ahí dentro, en claro.** Eso confirma lo de la teoría: el token no está cifrado, solo firmado. Por eso solo metimos el id.

> 💾 **Commit:** `feat: el login devuelve un token JWT`

---

### Bloque 3 — Verificar un token

Ya entregas tokens. Ahora necesitas el otro lado: dado un token, comprobar que es válido y recuperar el id. Esta lógica será el corazón del middleware de protección de mañana, así que hoy la montamos y la probamos aislada.

#### 3.1 Una ruta de prueba temporal

Solo para hoy, crea una ruta que reciba un token y te diga quién es. Mañana esto se convertirá en un middleware reutilizable; hoy lo hacemos a mano para ver el mecanismo.

El cliente envía el token en la cabecera `Authorization`, con el formato estándar:

```
Authorization: Bearer xxxxx.yyyyy.zzzzz
```

La palabra `Bearer` ("portador") es la convención: indica que quien presenta el token tiene derecho a usarlo. Tú tendrás que separar esa palabra del token real.

```js
const jwt = require("jsonwebtoken");

const quienSoy = (req, res) => {
  try {
    // 1. Leer la cabecera Authorization
    const authHeader = req.headers.authorization;
    if (!authHeader) {
      return res.status(401).json({ mensaje: "No hay token" });
    }

    // 2. Separar "Bearer" del token
    const token = authHeader.split(" ")[1];

    // 3. Verificar el token (si es inválido o expiró, lanza error)
    const datos = jwt.verify(token, process.env.JWT_SECRET);

    // 4. datos.id es el id que metimos al firmar
    res.status(200).json({ mensaje: "Token válido", id: datos.id });
  } catch (error) {
    // verify lanzó: token manipulado, expirado o mal formado
    res.status(401).json({ mensaje: "Token inválido o expirado" });
  }
};

module.exports = { registro, login, quienSoy };
```

Añade la ruta en `auth.routes.js`:

```js
const { quienSoy } = require("../controllers/auth.controller");
router.get("/quien-soy", quienSoy);
```

#### 3.2 Pruébalo (esta es la parte importante del día)

1. Haz login y copia el `token` que te devuelve.
2. `GET /api/auth/quien-soy` añadiendo en Postman la cabecera:
   `Authorization: Bearer <pega-aquí-tu-token>`
   → **200** con tu id. **El servidor te ha reconocido sin que mandes email ni contraseña.** Eso es la pulsera del festival funcionando.
3. Ahora prueba a romperlo, que es donde se aprende:
   - Sin cabecera `Authorization` → **401** "No hay token".
   - Cambia un carácter del token a mano → **401** "Token inválido". La firma ya no cuadra.
   - Cambia el `JWT_SECRET` de tu `.env`, reinicia el servidor y reintenta con el token de antes → **401**. El token fue firmado con el secreto viejo. _(Acuérdate de dejar el secreto como estaba.)_

> 💾 **Commit:** `feat: verificación de token JWT`

---

## Cierre del día

Comprueba con Postman:

- [ ] El login devuelve un token al meter credenciales correctas.
- [ ] El token contiene el id (lo viste en jwt.io) y **nada sensible**.
- [ ] Con el token en la cabecera `Authorization: Bearer ...`, la API te reconoce.
- [ ] Sin token, token manipulado o secreto cambiado → 401.
- [ ] El `JWT_SECRET` está en el `.env` y el `.env` está en el `.gitignore`.

Hoy has montado las dos mitades de la autenticación con tokens: **firmar** en el login y **verificar** un token que llega. Pero esa verificación está suelta en una ruta de prueba. Mañana la conviertes en un **middleware de protección de rutas**: en lugar de comprobar el token a mano en cada controlador, lo pones como guardián delante de las rutas que quieras proteger, y de paso dejas el `req.usuario` disponible para el controlador.

---

## 🚀 Reto opcional

1. **¿Y si no hay segunda parte?** En el paso de `authHeader.split(" ")[1]`, ¿qué pasa si alguien manda `Authorization: Patata` sin la estructura `Bearer token`? Pruébalo y observa el comportamiento. ¿Cómo lo harías más robusto antes de llamar a `verify`?
2. **Expiración real.** Cambia el `expiresIn` a `"10s"`, haz login, espera quince segundos e intenta usar el token. ¿Qué error concreto lanza `verify`? Investiga la diferencia entre `TokenExpiredError` y `JsonWebTokenError`: ¿podrías devolver un mensaje distinto para "tu sesión ha caducado" frente a "token inválido"?
