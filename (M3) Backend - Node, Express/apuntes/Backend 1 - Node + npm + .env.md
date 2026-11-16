# Día 20 Apertura M2 + Node + npm + .env

### ¿Qué es el backend?

Dibujo simple:

```
[NAVEGADOR]  <--HTTP-->  [SERVIDOR]  <-->  [BASE DE DATOS]
   (frontend)            (backend)         (donde viven los datos)
```

- Hasta ahora todo lo que se ha hecho vive **en el navegador del usuario**. Si refrescas, se pierde. Si otro usuario entra, no ve lo tuyo.
- El backend resuelve dos cosas: **persistencia** (los datos sobreviven) y **lógica compartida** (todos los usuarios ven lo mismo).
- Una API REST es un servidor que habla HTTP y devuelve JSON. Punto. Nada mágico.
- En M1 _consumíamos_ APIs ajenas. En M2 _construimos_ la nuestra.

### ¿Qué entregamos al final del M2?

- **Qué entregamos el 12 jun:**
  - API REST con CRUD funcionando
  - Persistencia en MongoDB
  - Autenticación con JWT + contraseñas con bcrypt
  - Documentación con Swagger
  - Deploy (URL pública)
  - Repo en GitHub bien organizado

### La filosofía iterativa

> _"En este módulo NO vais a construir durante 3 semanas para entregar al final. Vais a tener una API que funciona el jueves de esta semana. Y cada día le añadís una capa nueva: persistencia real, validación, login, archivos, tests, docs, deploy. Lo que hagáis cada día queda en el proyecto. Nada se tira."_

---

## Node + npm

### Teoría rápida

- **Node.js** = JavaScript fuera del navegador. El mismo lenguaje que ya conocéis, pero ejecutándose en vuestra máquina como un programa cualquiera.
- **npm** = gestor de paquetes de Node. Sirve para instalar librerías hechas por otros (y publicar las tuyas).
- Diferencia clave con el navegador: aquí **no hay `window`, no hay `document`, no hay DOM**. Hay otras cosas: `fs` (filesystem), `process` (info del proceso), módulos del sistema.

#### Paso 1: comprobar instalación

```bash
node -v
npm -v
```

#### Paso 2: el REPL de Node

```bash
node
> 2 + 2
> const nombre = "Fran"
> `Hola ${nombre}`
> .exit
```

> _"Esto es como la consola del navegador, pero sin navegador. ¿Veis? El mismo JavaScript."_

#### Paso 3: primer script

Crear `hola.js`:

```javascript
console.log("Hola desde Node");
console.log("La hora es:", new Date().toLocaleTimeString());
```

Ejecutar:

```bash
node hola.js
```

#### Paso 4: leer un fichero con `fs`

Crear `frase.txt` con cualquier texto. Crear `leer.js`:

```javascript
const fs = require("fs");

const contenido = fs.readFileSync("frase.txt", "utf-8");
console.log("El fichero dice:", contenido);
```

#### Paso 5: `npm init` y primer paquete

```bash
mkdir node_lessons
cd node_lessons
npm init -y
```

Abrir `package.json`. Pasear por los campos: `name`, `version`, `main`, `scripts`.

Instalar algo trivial pero visual:

```bash
npm install chalk@4
```

> ⚠️ Forzar v4 porque v5+ es ESM-only y rompe con `require`. Detalle pequeño pero les ahorra un disgusto.

Crear `index.js`:

```javascript
const chalk = require("chalk");

console.log(chalk.green("Esto sale verde"));
console.log(chalk.red.bold("Esto rojo y negrita"));
```

Mirar la carpeta `node_modules/`:

> _"Esto es lo que instala npm. NUNCA se sube a git. Hay un fichero, `package.json`, que dice qué necesita el proyecto, y cualquiera puede recrear `node_modules` con `npm install`."_

Crear `.gitignore`:

```
node_modules/
```

---

## Scripts + `.env`

### npm scripts

Editar `package.json`:

```json
"scripts": {
  "start": "node index.js",
  "saludo": "node hola.js",
  "dev": "node --watch index.js"
}
```

Ejecutar:

```bash
npm start
npm run saludo
npm run dev
```

> _"`start` es especial: no necesita `run`. El resto sí. Y `--watch` reinicia el script cuando cambia el archivo."_

### Variables de entorno

#### Por qué existen

> _"Imaginad que mañana metemos una contraseña de base de datos en el código. ¿Qué pasa cuando subes ese repo a GitHub?"_

- Las credenciales no van en el código.
- Cada entorno (mi máquina, la de mi compañero) puede tener valores distintos.
- Por eso → variables de entorno.

#### Práctica

Instalar dotenv:

```bash
npm install dotenv
```

Crear `.env`:

```
PUERTO=3000
SALUDO=Hola mundo desde variable de entorno
```

Crear `config.js`:

```javascript
require("dotenv").config();

console.log("Puerto:", process.env.PUERTO);
console.log("Saludo:", process.env.SALUDO);
console.log("Otra variable:", process.env.NO_EXISTE); // undefined
```

Ejecutar:

```bash
node config.js
```

Añadir `.env` al `.gitignore`:

```
node_modules/
.env
```

Crear `.env.example` (este SÍ se sube):

```
PUERTO=
SALUDO=
```

> _"Convención universal: `.env` con valores reales y privado, `.env.example` con las claves vacías y público. Cuando alguien clona tu repo, copia el `.example` a `.env` y rellena lo suyo."_

---

## Proyecto M2

### Idea de proyecto

- Una entidad principal con CRUD claro (notas, recetas, libros leídos, gym sets, películas vistas, gastos…)
- Que tenga sentido tener usuarios
- Que no sea Twitter ni Instagram (demasiado grande)
- Algo que **OS** les apetezca usar después

Algunos ejemplos:

- API de gestión de libros leídos
- Tracker de hábitos
- Lista de pelis vistas con valoración
- API de recetas

### Crear repo

```bash
mkdir [nombre-proyecto]_m2
cd [nombre-proyecto]_m2
npm init -y
echo "node_modules/" > .gitignore
echo ".env" >> .gitignore
git init
git add .
git commit -m "chore: setup inicial"
```

Crear repo en GitHub (público) y push.

**README mínimo**:

```markdown
# API [Nombre]

Proyecto del Módulo 2 — Full Stack Dev Bootcamp.

## Qué es

[Una frase explicando qué hace]

## Stack

- Node.js + Express
- MongoDB + Mongoose
- JWT para autenticación

## Estado

🚧 En desarrollo — Sprint 1
```

**Tablero Kanban** — usar GitHub Projects (built-in, gratis, en el mismo sitio). Cuatro columnas:

```
Backlog · Sprint actual · Doing · Done
```

Backlog inicial:

- Setup Express básico
- Endpoint `GET /health`
- Definir entidad principal y sus campos
- Endpoint `GET /[entidad]` (listar)
- Endpoint `POST /[entidad]` (crear)
- Endpoint `GET /[entidad]/:id` (uno)
- Endpoint `PUT /[entidad]/:id` (editar)
- Endpoint `DELETE /[entidad]/:id` (borrar)
- Conectar MongoDB con Mongoose
- Modelo Mongoose de la entidad

> _"Esto es vuestro sprint 1. El viernes esto está en Done."_

---
