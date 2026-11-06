# MongoDB + Mongoose: la base de datos de verdad

Bienvenido al sprint 2. Has cerrado la semana pasada con una API REST con CRUD completo, persistencia y arquitectura MVC limpia. Esta semana le añadimos las dos capas que la convierten en una API profesional: **base de datos de verdad** y **autenticación**.

Empezamos por la pieza que llevas esperando desde el primer día: dejar atrás el fichero JSON y conectar tu API a **MongoDB**, una base de datos de verdad. El viernes viste SQL. Hoy entras en el "otro mundo": el de las bases de datos de documentos.

Y vas a comprobar algo importante: el refactor a MVC del jueves no fue por capricho. Como dejaste la lógica de datos aislada en el modelo, hoy vas a cambiar tu fichero JSON por una base de datos real **tocando casi solo un archivo**. Los controladores y las rutas apenas se enteran. Eso es lo que se llama una **arquitectura bien diseñada**.

---

## Warm-up: kata

```javascript
// Tienes documentos como los que Mongo te devolverá:
const posts = [
  {
    _id: "65f1a2b3c4d5e6f7g8h9i0j1",
    titulo: "Hola mundo",
    autor: { nombre: "Ana", email: "ana@example.com" },
    tags: ["personal", "intro"],
    likes: 23,
  },
  {
    _id: "65f1a2b3c4d5e6f7g8h9i0j2",
    titulo: "Express mola",
    autor: { nombre: "Luis", email: "luis@example.com" },
    tags: ["backend", "tutorial"],
    likes: 41,
  },
  {
    _id: "65f1a2b3c4d5e6f7g8h9i0j3",
    titulo: "Async fácil",
    autor: { nombre: "Ana", email: "ana@example.com" },
    tags: ["javascript", "tutorial"],
    likes: 17,
  },
];

// 1. Devuelve los títulos de posts que tengan el tag "tutorial"
// 2. Devuelve los emails únicos de autores
// 3. Devuelve los posts ordenados por likes (desc)
```

> Fíjate en una cosa: este array de "documentos" tiene cosas que en SQL serían imposibles: el campo `autor` es un objeto anidado, y `tags` es un array. **Eso es Mongo**. Y todavía no hemos empezado.

---

## ¿Qué es MongoDB?

MongoDB es una **base de datos de documentos**. En vez de tablas con filas y columnas (como SQL), guarda **documentos** en **colecciones**.

Un **documento** es prácticamente un objeto JSON. De hecho, internamente Mongo usa BSON (Binary JSON), pero a ti, como desarrollador, se te presenta como JSON.

Una **colección** es un grupo de documentos relacionados. El equivalente a una tabla de SQL, pero sin esquema rígido: cada documento puede tener campos distintos.

Comparado con SQL:

| SQL                           | MongoDB                                                   |
| ----------------------------- | --------------------------------------------------------- |
| Base de datos                 | Base de datos                                             |
| Tabla                         | Colección                                                 |
| Fila                          | Documento                                                 |
| Columna                       | Campo                                                     |
| `id` (entero autoincremental) | `_id` (ObjectId, una cadena hexadecimal de 24 caracteres) |
| Schema fijo                   | Schema flexible                                           |

Un documento real de Mongo se parece a esto:

```javascript
{
  _id: ObjectId("65f1a2b3c4d5e6f7g8h9i0j1"),
  titulo: "Mi nota",
  contenido: "Hola Mongo",
  autor: {
    nombre: "Ana",
    email: "ana@example.com"
  },
  tags: ["primera", "test"],
  createdAt: ISODate("2026-06-01T10:00:00Z")
}
```

El campo `_id` es **automático**: Mongo lo genera por ti cada vez que insertas un documento. Es único globalmente. Lo importante para hoy: **los IDs en Mongo son strings (de 24 caracteres hex), no números enteros**. Esto va a impactar en un detalle de tu código, ya lo verás.

---

## ¿Y Mongoose qué es?

MongoDB, por defecto, no tiene esquema. Puedes meter en una colección un documento con `{ nombre: "Ana" }` y otro con `{ titulo: "Mi nota", precio: 42, etiquetas: ["a", "b"] }`. Mongo no se queja. Eso es muy flexible, pero también muy peligroso: si no controlas qué campos tiene cada documento, tu API se llena de bugs.

**Mongoose** es una librería de Node que se pone "encima" de MongoDB y le añade:

- **Esquemas:** defines qué campos tiene cada documento y de qué tipo.
- **Validaciones:** Mongoose comprueba antes de guardar que los datos cumplen el esquema.
- **Métodos de modelo cómodos:** `Modelo.find()`, `Modelo.findById()`, `Modelo.create()`, etc.
- **Middleware:** funciones que se ejecutan antes o después de guardar/actualizar.

Mongoose es a Mongo lo que un ORM (como Prisma o Sequelize) es a SQL: una capa intermedia que te facilita la vida. La diferencia es que en bases de datos de documentos se llama **ODM** (Object Document Mapper).

Casi todos los proyectos Node con MongoDB usan Mongoose. Es lo estándar.

---

## Paso 1: crear el cluster en MongoDB Atlas

No vamos a instalar MongoDB en tu máquina. Vamos a usarlo en la nube, gratis, con **MongoDB Atlas**. Es la solución oficial de los creadores de Mongo y es lo que usarás en producción (Render se conecta a Atlas perfectamente).

### 1.1 Crea una cuenta

Ve a [cloud.mongodb.com](https://cloud.mongodb.com) y regístrate con tu email o con Google. Es gratis y no pide tarjeta.

### 1.2 Crea un cluster gratuito

Al entrar te propone crear tu primer cluster:

- Tipo: **M0 Free** (el plan gratuito de por vida; 512 MB, suficientísimo para el proyecto).
- Proveedor: **AWS**, **Google Cloud** o **Azure**, da igual cuál.
- Región: la más cercana a ti. Para España, alguna europea (`eu-west-*` o `europe-west*`).
- Nombre del cluster: el que quieras, p. ej. `bootcamp-cluster`.

Crear. Tarda 1-3 minutos en estar listo.

### 1.3 Crea un usuario de base de datos

Atlas te pedirá un usuario y contraseña para conectarte. **No es tu cuenta de Atlas**, es un usuario de la base de datos en sí.

- Authentication Method: **Username and Password**.
- Username: lo que quieras, p. ej. `bootcamp_user`.
- Password: que sea segura. **Apúntala bien**, la vas a necesitar.

> ⚠️ Evita usar `@`, `#`, `:`, `/` y otros caracteres especiales en la contraseña. Mongo Atlas los permite pero hay que escaparlos en la URL de conexión y es un dolor.

### 1.4 Permite el acceso desde tu IP

Por seguridad, Atlas solo deja conectarse desde IPs que tú autorices. Para desarrollo:

- Ve a **Network Access** en el menú lateral.
- **Add IP Address**.
- **Allow Access from Anywhere** → te pone `0.0.0.0/0`. Esto significa "desde cualquier IP".

> ⚠️ `0.0.0.0/0` no es seguro para producción real. Para el proyecto del bootcamp está bien, pero apúntate mentalmente que en una empresa de verdad **nunca** abrirías Atlas al mundo entero.

### 1.5 Obtén el connection string

- Vuelve a **Database** en el menú lateral.
- Botón **Connect** sobre tu cluster.
- **Drivers** → Node.js.
- Copia el connection string. Tiene esta forma:

```
mongodb+srv://bootcamp_user:<password>@bootcamp-cluster.xxxxx.mongodb.net/?retryWrites=true&w=majority
```

Sustituye `<password>` por la contraseña real. **Apúntala bien, esto es lo que vas a meter en `.env` en un momento.**

---

## Paso 2: conectar tu servidor a Mongo

### 2.1 Instala Mongoose

En la raíz de tu proyecto:

```bash
npm install mongoose
```

### 2.2 Mete el connection string en `.env`

Abre tu `.env` (el que creaste en la clase 20) y añade:

```
MONGODB_URI=mongodb+srv://bootcamp_user:tu_password_real@bootcamp-cluster.xxxxx.mongodb.net/notas_db?retryWrites=true&w=majority
```

Fíjate en algo nuevo: justo antes del `?` he metido `/notas_db`. Eso es **el nombre de la base de datos** que vas a usar dentro del cluster. Si no la pones, Mongo guardará todo en una base por defecto llamada `test`. Mejor un nombre propio. Ponle el que quieras (`notas_db`, `recetas_db`, `mi_proyecto_db`...).

Asegúrate de que tu `.env` está en `.gitignore`. Si subes esto a GitHub, cualquiera podría conectarse a tu base de datos. **Las credenciales NUNCA en el repo.**

### 2.3 Crea un archivo de configuración para la conexión

Sigue el patrón limpio que ya tienes. Crea `src/config/db.js`:

```javascript
const mongoose = require("mongoose");

async function conectarBD() {
  try {
    await mongoose.connect(process.env.MONGODB_URI);
    console.log("✅ Conectado a MongoDB");
  } catch (error) {
    console.error("❌ Error conectando a MongoDB:", error.message);
    process.exit(1); // Si no hay BD, no tiene sentido arrancar el servidor
  }
}

module.exports = conectarBD;
```

> El `process.exit(1)` es importante: si no hay base de datos, no levantes el servidor. Es preferible que falle al arrancar (donde lo ves) a que arranque y se rompa en la primera petición.

### 2.4 Llama a `conectarBD` desde `index.js`

```javascript
require("dotenv").config();
const app = require("./src/app");
const conectarBD = require("./src/config/db");

const PUERTO = process.env.PUERTO || 3000;

async function arrancar() {
  await conectarBD();
  app.listen(PUERTO, () => {
    console.log(`Servidor escuchando en http://localhost:${PUERTO}`);
  });
}

arrancar();
```

Hemos envuelto el arranque en una función `async` porque conectar a la base de datos es asíncrono. Primero conectamos, después escuchamos.

Arranca con `npm run dev`. Si todo va bien, verás en consola:

```
✅ Conectado a MongoDB
Servidor escuchando en http://localhost:3000
```

Si ves un error, lo más probable es:

- **Contraseña mal**: revisa que la pusiste correctamente en el `.env`.
- **IP no permitida**: revisa Network Access en Atlas.
- **Carácter especial en la contraseña**: cámbiala por una sin símbolos raros.

---

## Paso 3: definir el esquema y el modelo

Ahora vamos a decirle a Mongoose qué pinta tienen los documentos de tu colección. Crea `src/models/notas.model.js` — sí, vamos a reescribir el archivo que tenías. Pero las **funciones que exporta seguirán llamándose igual**, así que los controladores no se enterarán.

### 3.1 El esquema

```javascript
const mongoose = require("mongoose");

const notaSchema = new mongoose.Schema(
  {
    titulo: {
      type: String,
      required: true,
      trim: true,
    },
    contenido: {
      type: String,
      required: true,
    },
  },
  {
    timestamps: true,
  },
);

const Nota = mongoose.model("Nota", notaSchema);
```

Lo que has hecho aquí:

- **`new mongoose.Schema({...})`** define el esquema: qué campos tiene un documento de notas y de qué tipo.
- **`type: String`**, `required: true`, `trim: true` son **validadores**. Mongoose comprueba estas reglas antes de guardar. `trim` elimina espacios al principio y al final automáticamente.
- **`{ timestamps: true }`** es magia pura: Mongoose añade automáticamente dos campos a cada documento: `createdAt` y `updatedAt`. Se rellenan solos cada vez que creas o actualizas un documento. Gratis. Brutal.
- **`mongoose.model("Nota", notaSchema)`** crea el modelo. Por convención, el primer argumento va en **singular y mayúscula**. Mongoose creará la colección en plural y minúscula automáticamente: `notas`.

### 3.2 Las funciones del modelo

Ahora, debajo del esquema y modelo, las funciones públicas:

```javascript
async function obtenerTodas() {
  return await Nota.find();
}

async function obtenerPorId(id) {
  return await Nota.findById(id);
}

async function crear(datos) {
  const nueva = new Nota(datos);
  return await nueva.save();
}

async function actualizar(id, datos) {
  return await Nota.findByIdAndUpdate(id, datos, {
    new: true, // devuelve el documento DESPUÉS de actualizar
    runValidators: true, // ejecuta los validadores del esquema
  });
}

async function borrar(id) {
  return await Nota.findByIdAndDelete(id);
}

module.exports = {
  obtenerTodas,
  obtenerPorId,
  crear,
  actualizar,
  borrar,
};
```

**Cosas importantes:**

- Los métodos de Mongoose (`find`, `findById`, `findByIdAndUpdate`...) devuelven Promesas. Por eso todo es `async/await`.
- Si `findById` no encuentra nada, devuelve `null` — exactamente igual que en tu modelo anterior. Por eso los controladores no necesitan cambios.
- `findByIdAndUpdate` por defecto devuelve el documento **antes** de actualizar. La opción `{ new: true }` cambia eso para que devuelva el **después**, que es lo que casi siempre quieres.
- `runValidators: true` hace que Mongoose ejecute los validadores también en updates. Por defecto solo los ejecuta en creates, lo cual es un comportamiento confuso. Mejor poner siempre `runValidators: true` en los updates.

### Compara con el modelo anterior

Si lo pones al lado del modelo de fichero JSON que tenías el jueves, verás que **las funciones públicas tienen exactamente la misma firma**:

- `obtenerTodas()` → devuelve un array de notas
- `obtenerPorId(id)` → devuelve una nota o null
- `crear(datos)` → devuelve la nota creada
- `actualizar(id, datos)` → devuelve la nota actualizada o null
- `borrar(id)` → devuelve la nota borrada o null

**Esto es lo que hace que los controladores no tengan que cambiar.** El modelo "promete" cierta interfaz, y lo cumple, sea cual sea su implementación interna. Eso es **abstracción** bien hecha.

---

## Paso 4: el ajuste mínimo del controlador

Te dije que el refactor a MVC iba a permitir cambiar a Mongo tocando "casi solo" el modelo. La palabra clave es **casi**. Hay un detalle.

En tu modelo anterior, los IDs eran números enteros (1, 2, 3...). En Mongo los IDs son strings hexadecimales de 24 caracteres: algo como `"65f1a2b3c4d5e6f7g8h9i0j1"`.

¿Recuerdas que en el controlador hacías esto?

```javascript
const id = Number(req.params.id);
```

Eso ya no tiene sentido: si conviertes a número un string hexadecimal, te sale `NaN`. **Hay que quitar el `Number()` de los controladores que reciben un `id`.**

```javascript
// ANTES (fichero JSON):
const id = Number(req.params.id);

// AHORA (Mongo):
const id = req.params.id;
```

O directamente usar `req.params.id` sin variable intermedia. Aplícalo en `obtenerNota`, `actualizarNota` y `borrarNota`.

> Sé honesto contigo mismo: el "solo tocas el modelo" era casi cierto. Esta es la realidad de los refactors: incluso con buena arquitectura, casi nunca son perfectos. Lo bueno es que el cambio en el controlador es mínimo y muy fácil de identificar. Sin la separación en capas, este cambio habría sido revisar 50 líneas en distintos sitios.

---

## Paso 5: comprobar que todo funciona

Arranca el servidor (`npm run dev`) y prueba las cinco operaciones con tu `peticiones.http` como el jueves.

### Crear una nota

```http
POST http://localhost:3000/api/notas
Content-Type: application/json

{
  "titulo": "Mi primera nota en Mongo",
  "contenido": "¡Esto sí es persistencia de verdad!"
}
```

Respuesta esperada (status 201):

```json
{
  "_id": "65f1a2b3c4d5e6f7g8h9i0j1",
  "titulo": "Mi primera nota en Mongo",
  "contenido": "¡Esto sí es persistencia de verdad!",
  "createdAt": "2026-06-01T10:00:00.000Z",
  "updatedAt": "2026-06-01T10:00:00.000Z",
  "__v": 0
}
```

Date cuenta de lo que ha aparecido sin que tú hicieras nada:

- **`_id`** generado automáticamente por Mongo.
- **`createdAt` y `updatedAt`** rellenados solos gracias a `timestamps: true`.
- **`__v`** es un campo interno de Mongoose para control de versiones. Ignóralo.

### Listar todas

```http
GET http://localhost:3000/api/notas
```

Te devuelve un array con todas las notas que tengas en la colección.

### Obtener una por id

```http
GET http://localhost:3000/api/notas/65f1a2b3c4d5e6f7g8h9i0j1
```

Copia el `_id` que te devolvió el POST y úsalo. Fíjate en que ahora la URL lleva un string largo, no un número.

### Actualizar

```http
PUT http://localhost:3000/api/notas/65f1a2b3c4d5e6f7g8h9i0j1
Content-Type: application/json

{
  "titulo": "Editada",
  "contenido": "Cambiado el contenido"
}
```

Si miras el resultado, verás que `updatedAt` ha cambiado y `createdAt` se ha mantenido.

### Borrar

```http
DELETE http://localhost:3000/api/notas/65f1a2b3c4d5e6f7g8h9i0j1
```

### Y un test definitivo: reinicia el servidor

Para el servidor (`Ctrl+C`) y vuelve a arrancarlo. Haz `GET /api/notas`. Los datos siguen ahí. **Eso es una base de datos de verdad.**

---

## Bonus: mira tus datos desde Atlas

Vuelve a la web de [cloud.mongodb.com](https://cloud.mongodb.com), entra en tu cluster, pestaña **Collections**. Ahí verás:

- Tu base de datos (`notas_db` o como la hayas llamado)
- La colección `notas` (en plural, en minúscula — eso lo hace Mongoose)
- Los documentos que has creado

Puedes navegar, filtrar, editar y borrar documentos directamente desde la UI. Es muy útil para depurar.

> Si quieres una herramienta aún más potente, está **MongoDB Compass**: una app de escritorio gratuita para inspeccionar Atlas. Es opcional; con la UI web de Atlas vas servido para el bootcamp.

---

## Aplicación al proyecto

Migra tu proyecto personal a MongoDB:

1. **Crea tu cluster en Atlas** (M0 Free) y obtén el connection string.
2. **Añade `MONGODB_URI` a tu `.env`** (revisa que `.env` está en `.gitignore`).
3. **Instala Mongoose** y crea `src/config/db.js`.
4. **Conecta la base de datos al arrancar el servidor** desde `index.js`.
5. **Reescribe `src/models/[tu_entidad].model.js`** con el esquema y modelo de Mongoose. Mantén exactamente las mismas funciones públicas (`obtenerTodas`, `obtenerPorId`, `crear`, `actualizar`, `borrar`).
6. **Quita el `Number(req.params.id)`** de los controladores que reciben IDs.
7. **Prueba las cinco operaciones del CRUD** y confirma que funcionan.
8. **Borra `datos.json`** del proyecto (ya no hace falta) y quítalo del `.gitignore` si lo tenías ahí.
9. **Actualiza el README** mencionando que el proyecto usa MongoDB Atlas.

Cuando lo tengas funcionando:

```bash
git add .
git commit -m "feat: migrar persistencia de fichero JSON a MongoDB Atlas"
git push
```

Y mueve a `Done` las tarjetas correspondientes del Kanban.

---

## Cierre

Hoy has dado un salto enorme: tu API ahora habla con una base de datos profesional en la nube. Y lo has hecho cambiando muy pocas líneas, gracias a la separación en capas del jueves.

Lo que tienes que llevarte grabado:

- **Mongo guarda documentos en colecciones**. Cada documento es prácticamente un JSON. El esquema es flexible por defecto.
- **Mongoose es un ODM** que le pone esquema y validaciones a Mongo, y te da una API muy cómoda para hacer queries.
- **`_id` es un string** de 24 caracteres hex, generado automáticamente. No es un número.
- **Variables de entorno** para las credenciales de la base de datos. Nunca en el código. Nunca en el repo.
- **`{ timestamps: true }`** en el esquema te da `createdAt` y `updatedAt` gratis. Úsalo siempre.
- **`{ new: true, runValidators: true }`** en los updates: para que devuelva el documento actualizado y para que ejecute las validaciones.
- El **`process.exit(1)`** si falla la conexión es un patrón: si no hay base de datos, no arrances el servidor.
