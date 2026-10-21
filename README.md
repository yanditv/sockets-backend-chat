# Chat en tiempo real con arquitectura MVC

**Guía de laboratorio para el estudiante**

Nombre: __________________________________  Curso: ______________

Fecha: ___________________  Integrante adicional: ___________________

## Propósito de la práctica

Construir y comprobar un backend de mensajería que combine una API REST con comunicación en tiempo real. Se utilizarán JavaScript, Bun, Express, Socket.IO, PostgreSQL y Prisma ORM. El trabajo comprende registro e inicio de sesión, creación de conversaciones, almacenamiento de mensajes y entrega de eventos a los participantes.

Sigue las etapas en orden. Cada paso indica qué hacer, dónde trabajar y qué comprobar. Los bloques de código son ejemplos de apoyo; las funciones descritas sin código se completan como parte de la práctica, usando la implementación MVC trabajada en clase.

### Resultados de aprendizaje

- Organizar rutas, controladores, servicios y modelos con responsabilidades diferenciadas.
- Autenticar solicitudes HTTP y conexiones Socket.IO mediante JWT.
- Verificar la pertenencia a una conversación antes de consultar o enviar mensajes.
- Persistir mensajes y demostrar su recepción simultánea desde dos clientes.

### Requisitos y organización

Trabajo individual o en parejas, según indique el docente. Duración sugerida: tres sesiones de dos horas. Se requieren conocimientos de JavaScript, funciones asíncronas, HTTP y bases de datos relacionales.

Herramientas: Bun instalado, Docker con Compose, editor de código y Postman. Utiliza JavaScript y módulos ES. No necesitas construir una interfaz gráfica para esta práctica.

### Producto esperado

Un proyecto que pueda iniciarse con bun run dev, una base de datos con migraciones y un cliente de terminal capaz de intercambiar mensajes. Entrega también un informe breve con las pruebas solicitadas al final.

## Arquitectura y carpetas del proyecto

### Árbol de carpetas y archivos

Esta es la estructura final. Crea los archivos en las rutas indicadas; las carpetas `node_modules/` y `prisma/migrations/` se generan mediante los comandos de instalación y migración.

```text
chat-backend/
├── prisma/
│   ├── migrations/                  # Generada por Prisma
│   └── schema.prisma                # Modelos y relaciones
├── scripts/
│   └── test-chat.js                 # Cliente para la prueba en terminal
├── src/
│   ├── config/
│   │   ├── env.js                   # Variables de entorno validadas
│   │   └── prisma.js                # Instancia compartida de Prisma
│   ├── controllers/
│   │   ├── auth.controller.js       # Registro e inicio de sesión
│   │   ├── user.controller.js       # Consulta de usuarios
│   │   └── conversation.controller.js
│   ├── middlewares/
│   │   ├── auth.middleware.js       # Verificación del token HTTP
│   │   └── error.middleware.js      # Respuestas de error
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── user.routes.js
│   │   └── conversation.routes.js
│   ├── services/
│   │   ├── auth.service.js          # Registro, contraseña y login
│   │   ├── user.service.js          # Consultas de usuarios
│   │   ├── conversation.service.js  # Participantes y autorización
│   │   └── message.service.js       # Guardado e historial
│   ├── sockets/
│   │   └── chat.socket.js           # Conexión, salas y eventos
│   ├── utils/
│   │   ├── http-error.js
│   │   ├── jwt.js
│   │   └── public-user.js
│   ├── validators/
│   │   └── schemas.js               # Validaciones Zod
│   ├── app.js                      # Configuración de Express
│   └── server.js                   # Inicio de HTTP y Socket.IO
├── .env                            # Configuración local privada
├── .env.example                    # Ejemplo sin secretos reales
├── .gitignore
├── compose.yaml                    # PostgreSQL en Docker
├── prisma.config.js                # Configuración de Prisma CLI
├── package.json
└── bun.lock                        # Generado por Bun
```

### Cómo se aplica MVC

| Parte | Ubicación | Función |
| --- | --- | --- |
| Modelo | `prisma/schema.prisma` | Define entidades y relaciones persistentes. |
| Controlador | `src/controllers/` | Recibe la solicitud, valida y construye la respuesta. |
| Presentación | Respuestas JSON de la API | Entrega los datos al cliente. No se crean plantillas HTML. |
| Servicios | `src/services/` | Contienen la lógica que comparten REST y Socket.IO. |
| Rutas | `src/routes/` | Relacionan cada URL con un controlador. |
| Tiempo real | `src/sockets/` | Gestiona conexiones y notificaciones del chat. |

### Recorrido de una solicitud y un mensaje

```mermaid
flowchart TD
  A[Cliente] --> B{Entrada}
  B --> C[API REST]
  B --> D[Socket.IO]
  C --> E[Ruta y autenticación HTTP]
  E --> F[Controlador]
  F --> G[Servicios compartidos]
  D --> H[Autenticación y evento]
  H --> G
  G --> I[Prisma y PostgreSQL]
  I --> J[Mensaje guardado]
  J --> K[Evento a participantes]
```

**Regla de organización:** las rutas no ejecutan consultas de Prisma. Los servicios verifican permisos y guardan datos. REST y Socket.IO usan el mismo servicio para crear mensajes.

## Orden de trabajo

| Etapa | Pasos | Resultado al finalizar |
| --- | --- | --- |
| Preparación | 1 a 4 | Dependencias y PostgreSQL disponibles. |
| Entorno y modelos | 5 a 7 | Variables de entorno y esquema definidos. |
| Conexión y MVC | 8 a 11 | Migración aplicada y archivos organizados. |
| Autenticación | 12 a 15 | Registro, login y rutas protegidas. |
| Conversaciones | 16 a 19 | Participantes, mensajes e historial. |
| Tiempo real | 20 a 24 | Socket.IO autenticado y eventos del chat. |
| Pruebas | 25 a 28 | Comunicación demostrada con dos usuarios. |

**Terminal:** salvo indicación contraria, ejecuta los comandos desde la raíz `chat-backend/`. Las rutas de los archivos también son relativas a esa carpeta.

---

## Etapa 1 — Preparar el proyecto

### Paso 01 — Crear el proyecto e instalar dependencias

**Dónde trabajar:** Terminal en `chat-backend/`.

**Qué hacer:** Crea la carpeta del proyecto e instala las dependencias. Ejecuta cada comando desde la terminal del editor.

```bash
mkdir chat-backend
cd chat-backend
bun init -y
bun add express@5 socket.io@4 cors dotenv bcryptjs jsonwebtoken zod
bun add pg @prisma/client@7 @prisma/adapter-pg@7
bun add -d prisma@7 socket.io-client@4
mkdir -p prisma scripts
mkdir -p src/config src/controllers src/middlewares src/routes
mkdir -p src/services src/sockets src/utils src/validators
```

### Paso 02 — Configurar los scripts de Bun

**Dónde trabajar:** `package.json`.

**Qué hacer:** En package.json conserva las dependencias y configura los siguientes scripts. Elimina index.ts si fue generado; el punto de entrada será src/server.js.

Actualiza estas propiedades dentro del objeto existente; conserva `dependencies` y `devDependencies`.

```jsonc
"type": "module",
"scripts": {
  "dev": "bun --watch src/server.js",
  "start": "bun src/server.js",
  "db:migrate": "bunx --bun prisma migrate dev",
  "db:generate": "bunx --bun prisma generate",
  "db:studio": "bunx --bun prisma studio",
  "test:chat": "bun scripts/test-chat.js"
}
```

### Paso 03 — Crear el servicio de PostgreSQL

**Dónde trabajar:** `compose.yaml`.

**Qué hacer:** Crea compose.yaml en la raíz. Estas credenciales son únicamente para el entorno local de la práctica.

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: chat
      POSTGRES_PASSWORD: chat_dev_password
      POSTGRES_DB: chat_db
    ports:
      - "5433:5432"
    volumes:
      - chat_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chat -d chat_db"]
      interval: 5s
      timeout: 5s
      retries: 10
volumes:
  chat_data:
```

### Paso 04 — Iniciar y verificar PostgreSQL

**Dónde trabajar:** Terminal en `chat-backend/`.

**Qué hacer:** Inicia la base de datos y comprueba que el servicio se encuentre saludable.

```bash
docker compose up -d
docker compose ps
```

**Comprueba antes de continuar:** el puerto de conexión desde Bun es 5433. El puerto interno del contenedor es 5432. No continúes con las migraciones hasta que PostgreSQL esté disponible.

## Etapa 2 — Configurar el entorno y los modelos

### Paso 05 — Definir las variables de entorno

**Dónde trabajar:** `.env` y `.gitignore`.

**Qué hacer:** Crea .env con el siguiente contenido. Sustituye JWT_SECRET por una clave aleatoria de al menos 32 caracteres y agrega .env y node_modules/ a .gitignore.

```dotenv
PORT=3000
DATABASE_URL="postgresql://chat:chat_dev_password@localhost:5433/chat_db"
JWT_SECRET="REEMPLAZAR_CON_UNA_CLAVE_ALEATORIA_DE_32_O_MAS_CARACTERES"
CLIENT_ORIGIN="http://localhost:5173"
```

### Paso 06 — Configurar Prisma CLI

**Dónde trabajar:** `prisma.config.js`.

**Qué hacer:** Crea prisma.config.js. La URL de las migraciones se define aquí para Prisma 7.

```javascript
import "dotenv/config";
import { defineConfig, env } from "prisma/config";
export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: env("DATABASE_URL") },
});
```

### Paso 07 — Definir los modelos de datos

**Dónde trabajar:** `prisma/schema.prisma`.

**Qué hacer:** Crea prisma/schema.prisma con estos modelos. Identifica la tabla que resuelve la relación de varios usuarios con varias conversaciones.

```prisma
generator client {
  provider = "prisma-client-js"
}
datasource db {
  provider = "postgresql"
}
model User {
  id String @id @default(uuid()) @db.Uuid
  name String
  email String @unique
  password String
  createdAt DateTime @default(now())
  memberships ConversationMember[]
  messages Message[]
}
model Conversation {
  id String @id @default(uuid()) @db.Uuid
  title String?
  createdAt DateTime @default(now())
  members ConversationMember[]
  messages Message[]
}
model ConversationMember {
  userId String @db.Uuid
  conversationId String @db.Uuid
  joinedAt DateTime @default(now())
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  conversation Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  @@id([userId, conversationId])
  @@index([conversationId])
}
model Message {
  id String @id @default(uuid()) @db.Uuid
  content String
  senderId String @db.Uuid
  conversationId String @db.Uuid
  createdAt DateTime @default(now())
  sender User @relation(fields: [senderId], references: [id])
  conversation Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  @@index([conversationId, createdAt])
}
```

## Etapa 3 — Conectar Prisma y organizar MVC

### Paso 08 — Crear las tablas y generar Prisma Client

**Dónde trabajar:** Terminal en `chat-backend/`.

**Qué hacer:** Aplica la migración y genera el cliente. Abre Prisma Studio y verifica las cuatro tablas, inicialmente vacías.

```bash
bun run db:migrate --name init
bun run db:generate
bun run db:studio
```

### Paso 09 — Validar las variables de entorno

**Dónde trabajar:** `src/config/env.js`.

**Qué hacer:** Crea src/config/env.js. Valida PORT, DATABASE_URL, JWT_SECRET y CLIENT_ORIGIN con Zod. Importa dotenv/config antes de leer process.env. Si falta una variable requerida, el servidor debe detenerse con un mensaje de configuración.

### Paso 10 — Conectar Prisma con PostgreSQL

**Dónde trabajar:** `src/config/prisma.js`.

**Qué hacer:** Crea src/config/prisma.js. Utiliza una sola instancia de Prisma para todo el proyecto.

```javascript
import { PrismaClient } from "@prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
import { env } from "./env.js";
const adapter = new PrismaPg({
  connectionString: env.DATABASE_URL,
});
export const prisma = new PrismaClient({ adapter });
```

### Paso 11 — Organizar los archivos MVC

**Dónde trabajar:** Carpetas de `src/` y archivos del árbol anterior.

**Qué hacer:** crea los archivos del árbol y distribuye sus responsabilidades según esta matriz. No coloques consultas de base de datos directamente en las rutas.

Completa también estos archivos auxiliares antes de integrar las rutas:

- `src/utils/public-user.js`: exporta la selección pública `{ id: true, name: true, email: true }`.
- `src/validators/schemas.js`: exporta los esquemas de registro, login, conversación, mensaje, UUID y paginación.
- `src/services/user.service.js`: implementa `listUsers` excluyendo al usuario autenticado y sin devolver contraseñas.
- `src/controllers/user.controller.js`: llama a `listUsers` y responde con JSON.
- `src/app.js`: configura CORS y JSON, monta las rutas, agrega `/api/health`, respuesta 404 y middleware de errores al final.

| Ubicación | Responsabilidad |
| --- | --- |
| prisma/schema.prisma | Modelos y relaciones de la base de datos. |
| src/routes | auth.routes.js, user.routes.js y conversation.routes.js. |
| src/controllers | Recibir datos, validar entradas, llamar servicios y devolver JSON. |
| src/services | auth.service.js, user.service.js, conversation.service.js y message.service.js. |
| src/middlewares | Autenticación HTTP y manejo centralizado de errores. |
| src/utils | JWT, HttpError y selección de campos públicos del usuario. |
| src/validators | Esquemas Zod de registro, login, conversaciones y mensajes. |
| src/sockets | Autenticación de conexiones y eventos del chat. |
| src/app.js y src/server.js | Configurar Express y compartir el servidor HTTP con Socket.IO. |

### Validaciones mínimas

Nombre de 2 a 100 caracteres; correo válido y normalizado a minúsculas; contraseña de al menos 8 caracteres y máximo 72 bytes para bcrypt; identificadores UUID; mensaje de 1 a 4000 caracteres después de trim. No aceptes senderId desde el cliente: debe obtenerse del token.

### Comprobación de esta etapa

Explica dónde está el modelo en esta implementación y por qué no hay plantillas HTML. En una API, los controladores producen respuestas JSON que consumirá un cliente externo.

## Etapa 4 — Implementar autenticación y rutas REST

### Paso 12 — Implementar registro e inicio de sesión

**Dónde trabajar:** `src/services/auth.service.js` y `src/controllers/auth.controller.js`.

**Qué hacer:** En auth.service.js implementa registerUser y loginUser. Guarda la contraseña con bcrypt.hash y verifica con bcrypt.compare. Devuelve únicamente id, name y email del usuario, además del token; nunca devuelvas el hash.

### Paso 13 — Proteger solicitudes con JWT

**Dónde trabajar:** `src/utils/jwt.js` y `src/middlewares/auth.middleware.js`.

**Qué hacer:** En utils/jwt.js implementa createToken y verifyToken usando HS256, subject igual al ID del usuario y expiración de ocho horas. En auth.middleware.js exige Authorization: Bearer TOKEN y asigna req.userId desde el token verificado.

### Paso 14 — Conectar rutas y controladores

**Dónde trabajar:** `src/routes/`, `src/controllers/` y `src/app.js`.

**Qué hacer:** Conecta rutas y controladores. Usa Express 5 y deja el middleware de errores al final de app.js.

| Método y ruta | Comportamiento esperado |
| --- | --- |
| POST /api/auth/register | Crear usuario. Respuesta 201 con user y token. |
| POST /api/auth/login | Validar credenciales. Respuesta 200 con user y token. |
| GET /api/users | Listar usuarios disponibles sin contraseña y con un límite de resultados. |
| POST /api/conversations | Crear conversación e incluir al usuario autenticado. |
| GET /api/conversations | Listar únicamente conversaciones del usuario. |
| GET /api/conversations/:id/messages | Consultar historial con limit y before opcional. |
| POST /api/conversations/:id/messages | Guardar un mensaje y emitir message:new. |
| GET /api/health | Comprobar conexión a PostgreSQL con SELECT 1. |

### Paso 15 — Centralizar el manejo de errores

**Dónde trabajar:** `src/utils/http-error.js` y `src/middlewares/error.middleware.js`.

**Qué hacer:** Implementa un error HttpError con status y message. Centraliza las respuestas: 400 para entradas inválidas, 401 para autenticación inválida, 403 para acceso ajeno, 409 para correo duplicado y 500 para errores internos. Registra el detalle interno en el servidor.

### Comprobación en Postman

Registra a Junior y Ana con cuerpos JSON como el siguiente. Guarda el ID y el token de cada uno en variables de tu colección; no incluyas tokens completos en el informe.

```http
POST http://localhost:3000/api/auth/register
{
  "name": "Junior",
  "email": "junior@example.com",
  "password": "Prueba12345"
}
```

Repite con Ana y ana@example.com. Luego inicia sesión mediante /api/auth/login utilizando email y password. Consulta /api/users con el token de Junior.

### Comprobación de la etapa

Demuestra que una contraseña incorrecta devuelve 401 y que registrar el mismo correo dos veces devuelve 409. Verifica que ninguna respuesta contenga password.

## Etapa 5 — Implementar conversaciones e historial

### Paso 16 — Crear conversaciones y participantes

**Dónde trabajar:** `src/services/conversation.service.js`.

**Qué hacer:** En conversation.service.js crea createConversation. Recibe title opcional y memberIds. Agrega req.userId, elimina IDs repetidos, exige al menos dos usuarios distintos y verifica que todos existan antes de crear la conversación con sus participantes.

```http
POST http://localhost:3000/api/conversations
Authorization: Bearer TOKEN_JUNIOR
{
  "title": "Chat de laboratorio",
  "memberIds": ["ID_REAL_DE_ANA"]
}
```

Guarda el id devuelto como conversationId. Los textos ID_REAL_DE_ANA y TOKEN_JUNIOR son marcadores y deben reemplazarse con valores obtenidos del registro.

### Paso 17 — Verificar la pertenencia a la conversación

**Dónde trabajar:** `src/services/conversation.service.js`.

**Qué hacer:** Implementa requireMembership. Reutiliza esta comprobación para consultar y enviar mensajes, tanto desde REST como desde Socket.IO.

```javascript
export async function requireMembership(userId, conversationId) {
  const member = await prisma.conversationMember.findUnique({
    where: {
      userId_conversationId: { userId, conversationId },
    },
  });
  if (!member) {
    throw new HttpError(403, "No perteneces a esta conversación");
  }
}
```

Importa prisma y HttpError en ese archivo y valida conversationId como UUID antes de consultar. El nombre userId_conversationId corresponde a la clave compuesta del modelo.

### Paso 18 — Guardar mensajes

**Dónde trabajar:** `src/services/message.service.js`.

**Qué hacer:** En message.service.js implementa createMessage. Valida content y conversationId, verifica la membresía y guarda el mensaje con senderId obtenido de la autenticación. Devuelve el mensaje con los campos públicos de su autor.

### Paso 19 — Consultar el historial con paginación

**Dónde trabajar:** `src/services/message.service.js`.

**Qué hacer:** Implementa listMessages con limit entre 1 y 100 y valor predeterminado de 30. Ordena por createdAt e id, ambos descendentes; devuelve la página en orden cronológico. Para before, verifica que el mensaje usado como cursor pertenezca a la misma conversación.

Consulta limit + 1 registros para determinar si existe otra página. Devuelve messages y nextCursor; usa null cuando ya no haya mensajes anteriores.

### Comprobación por HTTP

```http
POST /api/conversations/ID_CONVERSACION/messages
Authorization: Bearer TOKEN_JUNIOR
{ "content": "Hola Ana, mensaje de prueba" }

GET /api/conversations/ID_CONVERSACION/messages?limit=30
```

Resultado esperado: respuesta 201 al enviar y el mensaje dentro del historial. Registra un tercer usuario, sin agregarlo a esta conversación, y comprueba que no pueda leerla ni enviar mensajes.

## Etapa 6 — Conectar Socket.IO al servidor

### Paso 20 — Compartir el servidor HTTP

**Dónde trabajar:** `src/server.js`.

**Qué hacer:** En server.js crea un servidor HTTP compartido por Express y Socket.IO. No ejecutes app.listen en un segundo puerto.

```javascript
import { createServer } from "node:http";
import { Server } from "socket.io";
import app from "./app.js";
import { env } from "./config/env.js";
import { prisma } from "./config/prisma.js";
import { configureChatSocket } from "./sockets/chat.socket.js";
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: env.CLIENT_ORIGIN },
  maxHttpBufferSize: 100_000,
});
app.set("io", io);
configureChatSocket(io);
await prisma.$connect();
httpServer.listen(env.PORT, () => console.log("Servidor iniciado"));
```

### Paso 21 — Autenticar las conexiones Socket.IO

**Dónde trabajar:** `src/sockets/chat.socket.js`.

**Qué hacer:** En configureChatSocket, usa io.use para leer socket.handshake.auth.token. Verifica el JWT y que el usuario exista. Guarda su ID en socket.data.userId y rechaza la conexión con next(new Error(...)) si no es válida.

### Paso 22 — Asignar salas y controlar la expiración

**Dónde trabajar:** `src/sockets/chat.socket.js`.

**Qué hacer:** Al conectarse, agrega el socket a user:ID_USUARIO. Programa la desconexión al vencer el JWT y elimina ese temporizador en disconnect. La autenticación inicial no vuelve a ejecutarse por cada mensaje.

### Paso 23 — Recibir y confirmar mensajes

**Dónde trabajar:** `src/sockets/chat.socket.js`.

**Qué hacer:** Implementa message:send. Llama al mismo createMessage utilizado por REST. Después de guardar, consulta los participantes y emite message:new a sus salas de usuario. Responde mediante el callback de confirmación.

```javascript
socket.on("message:send", async (data, callback) => {
  const reply = typeof callback === "function" ? callback : () => {};
  try {
    // Verifica también la vigencia del token.
    const message = await createMessage(socket.data.userId, data);
    // Guarda primero; maneja un fallo de emisión por separado.
    try {
      await broadcastMessage(io, message);
    } catch (error) {
      console.error("Falló la emisión", error);
    }
    reply({ ok: true, message });
  } catch (error) {
    // Devuelve un mensaje controlado, sin detalles de la base de datos.
    reply({ ok: false, error: "No se pudo enviar el mensaje" });
  }
});
```

### Paso 24 — Emitir mensajes a los participantes

**Dónde trabajar:** `src/sockets/chat.socket.js` y `src/controllers/conversation.controller.js`.

**Qué hacer:** Implementa broadcastMessage con io.to(listaDeSalas).emit. Invócalo también desde el controlador POST de mensajes. El evento no debe enviarse a todos los usuarios del servidor.

### Comprobación de la etapa

El receptor obtiene id, conversationId, content, createdAt y sender. El emisor también recibe message:new. Si el frontend utiliza el callback y el evento para pintar mensajes, debe evitar duplicarlos por id.

## Etapa 7 — Probar con dos clientes

### Paso 25 — Crear el cliente de terminal

**Dónde trabajar:** `scripts/test-chat.js`.

**Qué hacer:** Crea scripts/test-chat.js. Este cliente permite comprobar el backend sin interfaz gráfica.

```javascript
import { createInterface } from "node:readline";
import { io } from "socket.io-client";
const [token, conversationId] = process.argv.slice(2);
if (!token || !conversationId) {
  console.log("Uso: bun run test:chat TOKEN ID_CONVERSACION");
  process.exit(1);
}
const socket = io("http://localhost:3000", { auth: { token } });
const terminal = createInterface({
  input: process.stdin,
  output: process.stdout,
});
socket.on("connect", () => console.log("Conectado. Escribe un mensaje"));
socket.on("connect_error", (e) => console.log(e.message));
socket.on("disconnect", (reason) => console.log("Desconectado", reason));
socket.on("message:new", (message) => {
  if (message.conversationId === conversationId) {
    console.log(`${message.sender.name}: ${message.content}`);
  }
});
terminal.on("line", (content) => {
  if (!content.trim()) return;
  if (!socket.connected) return console.log("Sin conexión");
  socket.timeout(5000).emit(
    "message:send",
    { conversationId, content },
    (error, result) => {
      if (error) {
        console.log("Sin confirmación. Revisa el historial antes de reenviar");
        return;
      }
      if (!result.ok) console.log(result.error);
    },
  );
});
terminal.on("close", () => socket.disconnect());
```

### Paso 26 — Iniciar el servidor y los dos clientes

**Dónde trabajar:** Tres terminales abiertas en `chat-backend/`.

**Qué hacer:** Ejecuta bun run dev en una terminal. Abre otras dos con los tokens de los participantes y el mismo ID de conversación.

**Terminal 1 — servidor**

```bash
bun run dev
```

**Terminal 2 — Junior**

```bash
bun run test:chat "TOKEN_JUNIOR" "ID_CONVERSACION"
```

**Terminal 3 — Ana**

```bash
bun run test:chat "TOKEN_ANA" "ID_CONVERSACION"
```

**Comprueba:** ambos clientes deben mostrar `Conectado`. Sustituye los marcadores por los tokens reales y utiliza el mismo ID de conversación en ambas terminales.

### Paso 27 — Comprobar intercambio y reconexión

**Dónde trabajar:** Clientes de terminal y Postman.

**Qué hacer:** Envía al menos tres mensajes desde cada usuario. Desconecta a Ana, envía otro mensaje desde Junior y vuelve a conectar a Ana. Consulta el historial por REST para recuperar el mensaje enviado durante la desconexión.

### Paso 28 — Comprobar el envío desde REST

**Dónde trabajar:** Postman y cliente de Ana.

**Qué hacer:** Mantén el cliente de Ana conectado y envía un mensaje desde Postman. Ana debe recibir message:new sin reiniciar el cliente.

Resultado esperado: los mensajes se guardan una sola vez por envío y llegan a ambos participantes conectados. Esta implementación no recupera automáticamente eventos perdidos ni garantiza reintentos sin duplicados; el historial REST es la fuente de recuperación.

## Entrega — Registrar pruebas y resultados

Completa la matriz con el resultado real y una referencia a la evidencia. Una captura del código no sustituye una prueba del comportamiento.

| Prueba | Resultado esperado | Resultado real |
| --- | --- | --- |
| Registro y login | 201 en registro y 200 en login. |  |
| Correo duplicado | 409 sin crear otro usuario. |  |
| Token ausente o inválido | 401 en REST y rechazo de conexión Socket.IO. |  |
| Mensaje vacío | Rechazo sin insertar un registro. |  |
| Envío bidireccional | Ambos clientes reciben los mensajes. |  |
| Envío desde REST | El cliente conectado recibe message:new. |  |
| Usuario ajeno | 403 en REST; rechazo por evento y sin inserción. |  |
| Persistencia | El historial conserva mensajes tras reiniciar el backend. |  |
| Paginación | Con limit=2 se recupera la página anterior sin repetir IDs. |  |
| Desconexión | El mensaje pendiente aparece en el historial REST. |  |

### Lista de entrega

- [ ] Proyecto comprimido o repositorio con código, bun.lock y migraciones. Excluye node_modules y .env. Incluye .env.example sin secretos reales.
- [ ] README con requisitos, comandos de inicio, migración y prueba del chat.
- [ ] Colección de Postman sin tokens vigentes ni contraseñas personales.
- [ ] Informe breve con matriz completada, evidencias y respuestas de análisis.

### Preguntas de análisis

1. ¿Qué operaciones resolviste con REST y cuáles con Socket.IO?
1. ¿Por qué se guarda el mensaje antes de emitir el evento?
1. ¿Por qué senderId debe obtenerse del JWT y no del cuerpo del mensaje?
1. ¿Qué impide que un tercer usuario lea una conversación ajena?
1. ¿Qué ocurre si el mensaje se guarda, pero el emisor no recibe la confirmación?
1. ¿Cómo recuperarías los mensajes que llegaron durante una desconexión?

Cierre: realiza una demostración breve al docente y explica el recorrido de un mensaje desde el cliente hasta PostgreSQL y los receptores.

## Apoyo — Resolver problemas frecuentes

| Situación | Acción de revisión |
| --- | --- |
| PostgreSQL no responde | Ejecuta docker compose ps y docker compose logs postgres. Revisa el puerto 5433 y DATABASE_URL. |
| Prisma Client no disponible | Ejecuta bun run db:generate después de instalar o cambiar el esquema. |
| No existen las tablas | Ejecuta bun run db:migrate --name init y revisa la base de destino. |
| Error de configuración de Prisma | No mezcles instrucciones de Prisma 6 con Prisma 7. Conserva URL en prisma.config.js y usa PrismaPg en la aplicación. |
| 401 al consultar rutas | Envía Authorization: Bearer TOKEN con un token vigente. |
| Socket no conecta | Utiliza socket.io-client y auth.token. Un cliente WebSocket nativo no habla el protocolo de Socket.IO. |
| 403 al consultar mensajes | Comprueba que el usuario autenticado esté en ConversationMember. |
| Mensaje guardado sin evento | Revisa broadcastMessage, salas user:ID y conexión del receptor. Consulta el historial. |
| Mensajes duplicados en pantalla | Compara id antes de agregar. El callback y message:new pueden representar el mismo mensaje. |
| Cambio de código desconecta clientes | El modo --watch reinicia el servidor. Reconecta y consulta nuevamente el historial. |

### Extensión opcional

Implementa typing:start y typing:stop. Verifica la membresía antes de notificar y envía el indicador únicamente a los demás participantes. No almacenes este indicador como mensaje. Explica cómo evitar que permanezca visible tras una desconexión.

### Documentación de consulta

Consulta las páginas oficiales y verifica que las instrucciones correspondan a las versiones instaladas. Usa bun.lock para conservar las versiones resueltas del proyecto.

Bun — https://bun.sh/docs

Express — https://expressjs.com/

Socket.IO — https://socket.io/docs/v4/

Prisma ORM — https://www.prisma.io/docs/

PostgreSQL — https://www.postgresql.org/docs/

Alcance: entorno local de aprendizaje con una instancia del backend. La publicación, el escalado entre servidores, los archivos adjuntos y la confirmación de lectura quedan fuera de esta práctica.
