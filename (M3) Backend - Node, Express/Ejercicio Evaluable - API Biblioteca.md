# Ejercicio evaluable M3 — API "Mi Biblioteca"

**Duración:** 1 h · **Entrega:** carpeta del proyecto (o repo Git) con `.env.example`

Construye una API REST mínima para gestionar libros: el usuario se registra, inicia sesión y gestiona **sus propios** libros. Solo cosas vistas en clase.

> Para que entre en 1 h: **no** hay README largo, **no** hay PUT, **no** hay filtros ni paginación, y la estructura es plana.

---

## 1. Stack y arranque

```bash
npm i express mongoose dotenv express-validator bcrypt jsonwebtoken
npm i -D vitest supertest
```

Scripts: `start` (`node index.js`) y `test` (`vitest run`).

`.env` (con su `.env.example`): `PORT`, `MONGO_URI`, `JWT_SECRET`.

En `app.js`: `express.json()`, un **middleware logger propio** que imprima `[fecha] MÉTODO ruta`, y la ruta pública `GET /api/health` → `200 { "status": "ok" }`.

## 2. Estructura exigida (plana, 6 archivos)

```
index.js              arranca servidor + conexión a Mongo
src/
  app.js              express, logger, health, rutas, 404, errores
  models.js           Usuario y Libro
  auth.js             rutas /api/auth + middleware verificarToken
  libros.js           rutas /api/libros (protegidas)
tests/api.test.js
```

Las rutas pueden llevar la lógica dentro (no hace falta separar controladores), pero `app.js` **no** define rutas de negocio: las monta con `app.use()`.

## 3. Modelos (`timestamps: true` en ambos)

**Usuario:** `email` (String, requerido, único, `lowercase`), `password` (String, requerido, hasheado), `nombre` (String, requerido).

**Libro:** `titulo` (String, requerido), `autor` (String, requerido), `leido` (Boolean, default `false`), `propietario` (ObjectId, `ref: 'Usuario'`, requerido).

## 4. Autenticación

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/api/auth/register` | Hashea con bcrypt (10 rondas). `201` con el usuario **sin** password. Email repetido → `409`. |
| POST | `/api/auth/login` | `bcrypt.compare`. OK → `200 { token }` (JWT con payload `{ id, email }`, expira en `1h`). Si falla → `401 { error: "Credenciales inválidas" }`. |

**`verificarToken`**: lee `Authorization: Bearer <token>`, verifica y deja `req.usuario = payload`.
- Sin cabecera → `401 { error: "Token no proporcionado" }`
- Token inválido → `401 { error: "Token inválido" }`

## 5. CRUD de libros (todas con token)

Filtra **siempre** por `req.usuario.id`.

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/api/libros` | `200` array de los libros del usuario |
| POST | `/api/libros` | `201` libro creado |
| DELETE | `/api/libros/:id` | `204` sin cuerpo; `404` si no existe o no es suyo |

> ⚠️ Borrar un libro de otro usuario debe devolver `404`.

## 6. Validación

`express-validator` + un middleware `validar` que responda `400 { errors: [...] }`.

- `register`: `email` válido, `password` mínimo 6, `nombre` no vacío (`trim`).
- `POST /api/libros`: `titulo` y `autor` no vacíos.
- `DELETE /api/libros/:id`: `param('id').isMongoId()`.

## 7. Errores

Middleware de error final (4 argumentos) → `500 { error: "Error interno del servidor" }`, y un `404` para rutas inexistentes.

## 8. Tests (3 con supertest sobre la app de `src/app.js`)

1. `GET /api/health` → 200.
2. Registro correcto → 201 y la respuesta **no** incluye `password`.
3. `GET /api/libros` sin token → 401.

---

## Rúbrica (100 pts)

| Apartado | Pts |
|---|---|
| Estructura, `.env` y scripts | 10 |
| Logger, JSON, 404 y middleware de errores | 15 |
| Modelos Mongoose con reglas y `ref` | 15 |
| Registro/login con bcrypt y JWT | 25 |
| `verificarToken` + aislamiento por usuario | 15 |
| CRUD con status codes correctos | 10 |
| Validaciones con express-validator | 5 |
| 3 tests pasando | 5 |

## 🚀 Extras opcionales (+ hasta 10 pts)

- `GET /api/libros/:id` con `404` si no es suyo.
- `PUT /api/libros/:id` con validaciones `.optional()`.
- Filtro `?leido=true|false` en `GET /api/libros`.
