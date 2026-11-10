# Taller de laboratorio: chat en tiempo real con React, Vite y Tailwind CSS

**Duración:** 4 horas (240 minutos) · **Modalidad sugerida:** parejas  
**Base:** [Guía de laboratorio Chat MVC con Bun](https://chatgpt.com/api/library/files/libfile_d5ecda67fa3081919a066ac76888bdb3/download)

Construye el cliente web del chat desarrollado en la práctica anterior. Separa la interfaz en componentes de React y usa Tailwind CSS 4 para los estilos. Al terminar, dos personas podrán registrarse, iniciar sesión, abrir la misma conversación e intercambiar mensajes sin recargar la página. Después comprobarás que los mensajes se pueden recuperar desde PostgreSQL, incluso si el destinatario estaba desconectado.

**Punto de partida:** el backend MVC debe estar completo y funcional antes del taller. No se reconstruye en estas cuatro horas. Necesitas Bun, Docker, un editor y dos perfiles del navegador o una ventana normal y otra privada.

| Hora | Pasos | Distribución y resultado |
| --- | --- | --- |
| 1 | 01–06 | 20 min para entorno y Tailwind; 40 min para registro e inicio de sesión. |
| 2 | 07–09 | 10 min para revisar la API; 25 min para crear componentes; 25 min para integrarlos y consultar el historial. |
| 3 | 10–11 | 35 min para conectar Socket.IO; 25 min para enviar y recibir mensajes entre dos usuarios. |
| 4 | 12–14 | 35 min para pruebas; 10 min para compilar; 15 min para evidencias y demostración. |

Cada bloque dura 60 minutos.

## Arquitectura y archivos

```mermaid
flowchart TD
    U["Navegador con React"] -->|Registro, login, usuarios e historial| R["API REST de Express"]
    U -->|message:send| S["Socket.IO"]
    R --> L["Servicios del backend MVC"]
    S --> L
    L --> P["Prisma y PostgreSQL"]
    S -->|message:new| V["Navegadores de los participantes"]
```

React se encarga de la interfaz y del estado visible. El backend sigue siendo responsable de validar el JWT, verificar la pertenencia al chat y guardar los mensajes. La confirmación de Socket.IO indica que el servidor aceptó el envío; no indica que el destinatario lo leyó.

```text
laboratorio-chat/
├── chat-backend/                # Proyecto MVC de la práctica anterior
│   ├── prisma/
│   ├── src/
│   ├── .env
│   └── compose.yaml
└── chat-frontend/
    ├── .env.example
    ├── .env.local              # Solo para tu equipo; no subir a Git
    ├── index.html
    ├── package.json
    ├── vite.config.js          # React y plugin de Tailwind CSS
    ├── bun.lock
    └── src/
        ├── components/
        │   ├── AuthForm.jsx     # Registro e inicio de sesión
        │   ├── Chat.jsx         # Estado y coordinación del chat
        │   ├── ConversationSidebar.jsx
        │   ├── MessageList.jsx
        │   ├── MessageComposer.jsx
        │   └── ConnectionBadge.jsx
        ├── hooks/
        │   └── useChatSocket.js # Ciclo de vida de la conexión
        ├── services/
        │   └── api.js          # Solicitudes REST y encabezado Bearer
        ├── App.jsx             # Cambia entre acceso y chat
        ├── main.jsx            # Punto de entrada generado por Vite
        └── index.css           # Importación de Tailwind CSS
```

| Componente | Responsabilidad | Datos principales que recibe |
| --- | --- | --- |
| `App` | Alternar entre acceso y chat. | Estado de sesión propio. |
| `AuthForm` | Registrar e iniciar sesión. | `onSession`. |
| `Chat` | Coordinar las consultas, la conversación activa y los mensajes. | `session` y `onUnauthorized`. |
| `ConversationSidebar` | Elegir un usuario, crear un chat y cambiar de conversación. | Listas, selección y funciones de acción. |
| `MessageList` | Mostrar el historial y distinguir mensajes propios. | `messages`, `userId` y `loading`. |
| `MessageComposer` | Mostrar el campo de texto y el botón de envío. | `value`, `onChange`, `onSend` y estado del envío. |
| `ConnectionBadge` | Mostrar si el socket está conectado. | `connected`. |

Los datos bajan desde `Chat` mediante props. Los componentes hijos avisan de las acciones mediante funciones como `onSelect` y `onSend`. Las solicitudes REST se centralizan en `api.js` y la conexión se gestiona en `useChatSocket.js`.

## Hora 1 — Preparar el entorno y la sesión

### Paso 01 — Iniciar y comprobar el backend

En `chat-backend/`, confirma que `CLIENT_ORIGIN=http://localhost:5173` esté en el `.env`. Usa exactamente ese host y puerto en el navegador.

```bash
docker compose up -d
docker compose ps
bun install
bun run db:generate
```

Si la base de datos es nueva, espera a que PostgreSQL esté disponible y ejecuta `bun run db:migrate --name init`. Luego inicia el servidor con `bun run dev`. En otra terminal comprueba:

```bash
curl http://localhost:3000/api/health
```

**Resultado esperado:** HTTP 200 y un JSON con `status: "ok"`. Si falla, revisa el backend antes de crear el frontend.

### Paso 02 — Crear el proyecto React

Abre una terminal en la carpeta que contiene `chat-backend/`, para crear ambos proyectos al mismo nivel.

```bash
bun create vite chat-frontend --template react
cd chat-frontend
bun install
bun add socket.io-client@4
bun add -d tailwindcss@4 @tailwindcss/vite@4
mkdir -p src/components src/hooks src/services
```

Si el asistente pregunta si debe iniciar el servidor automáticamente, elige **No**. Conserva `src/main.jsx` generado por Vite y comprueba que importe `./index.css`. Consulta la [guía de Bun para Vite](https://bun.com/guides/ecosystem/vite).

### Paso 03 — Configurar Tailwind y la URL del backend

**Archivo:** `vite.config.js`. Mantén el plugin de React y añade el de [Tailwind CSS para Vite](https://tailwindcss.com/docs/installation/using-vite).

```javascript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({ plugins: [react(), tailwindcss()] });
```

**Archivo:** `src/index.css`. Sustituye todo su contenido por:

```css
@import "tailwindcss";
```


En la raíz de `chat-frontend/`, crea `.env.local` y `.env.example` con esta URL pública de desarrollo. Reinicia Vite si ya estaba abierto. Agrega `.env.local` a `.gitignore` si la plantilla no lo excluye.

```dotenv
VITE_BACKEND_URL=http://localhost:3000
```

Vite expone las variables `VITE_*` al navegador. Mantén `JWT_SECRET` y `DATABASE_URL` exclusivamente en el backend. Consulta la [documentación de variables de Vite](https://vite.dev/guide/env-and-mode).

Inicia el frontend:

```bash
bun --bun run dev --host localhost --port 5173 --strictPort
```

**Comprueba:** abre [http://localhost:5173](http://localhost:5173). Para verificar Tailwind, añade temporalmente `className="text-3xl font-bold text-indigo-600"` al título de la pantalla inicial. Debe verse grande, en negrita y de color índigo.

### Paso 04 — Centralizar las solicitudes REST

**Archivo:** `src/services/api.js`.

```javascript
export const BACKEND_URL = (import.meta.env.VITE_BACKEND_URL
  || "http://localhost:3000").replace(/\/+$/, "");

export async function api(path, { method = "GET", token, body, signal } = {}) {
  const headers = {};
  if (token) headers.Authorization = `Bearer ${token}`;
  if (body !== undefined) headers["Content-Type"] = "application/json";

  const response = await fetch(`${BACKEND_URL}/api${path}`, {
    method, headers, signal,
    body: body === undefined ? undefined : JSON.stringify(body),
  });

  const data = await response.json().catch(() => null);
  if (!response.ok) {
    const error = new Error(data?.message || `Error HTTP ${response.status}`);
    error.status = response.status;
    throw error;
  }
  return data;
}
```

### Paso 05 — Crear el formulario de registro e inicio de sesión

**Archivo:** `src/components/AuthForm.jsx`.

```jsx
import { useState } from "react";
import { api } from "../services/api.js";

export default function AuthForm({ onSession }) {
  const [register, setRegister] = useState(false);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState("");

  async function submit(event) {
    event.preventDefault();
    if (busy) return;
    const data = Object.fromEntries(new FormData(event.currentTarget));
    setBusy(true);
    setError("");
    try {
      const path = register ? "/auth/register" : "/auth/login";
      onSession(await api(path, { method: "POST", body: data }));
    } catch (err) {
      setError(err.message);
    } finally {
      setBusy(false);
    }
  }

  return (
    <section className="mx-auto max-w-md rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
      <h1 className="mb-4 text-2xl font-bold">{register ? "Crear cuenta" : "Iniciar sesión"}</h1>
      <form className="grid gap-3" onSubmit={submit}>
        {register && <label className="grid gap-1 text-sm font-medium">Nombre <input className="rounded-lg border border-slate-300 px-3 py-2" name="name" required minLength={2} /></label>}
        <label className="grid gap-1 text-sm font-medium">Correo <input className="rounded-lg border border-slate-300 px-3 py-2" name="email" type="email" required /></label>
        <label className="grid gap-1 text-sm font-medium">Contraseña <input className="rounded-lg border border-slate-300 px-3 py-2" name="password" type="password" required minLength={8} /></label>
        <button className="rounded-lg bg-indigo-600 px-4 py-2 font-semibold text-white disabled:opacity-50" disabled={busy}>{busy ? "Enviando…" : "Continuar"}</button>
      </form>
      {error && <p className="mt-2 text-sm text-red-700" role="alert">{error}</p>}
      <button className="mt-4 text-sm font-medium text-indigo-700" type="button" disabled={busy} onClick={() => {
        setRegister(!register);
        setError("");
      }}>
        {register ? "Ya tengo cuenta" : "Crear una cuenta"}
      </button>
    </section>
  );
}
```

### Paso 06 — Cambiar de pantalla al iniciar o cerrar sesión

**Archivo:** `src/App.jsx`. En esta etapa crea primero el marcador de posición en `src/components/Chat.jsx` y reemplázalo en el paso 09.

```jsx
// src/components/Chat.jsx — temporal
export default function Chat({ session }) {
  return <h1>Bienvenido, {session.user.name}</h1>;
}
```

```jsx
// src/App.jsx
import { useState } from "react";
import AuthForm from "./components/AuthForm.jsx";
import Chat from "./components/Chat.jsx";

const KEY = "laboratorio-chat";

function readSession() {
  try {
    const value = JSON.parse(sessionStorage.getItem(KEY));
    return value?.user?.id && value?.token ? value : null;
  } catch {
    return null;
  }
}

export default function App() {
  const [session, setSession] = useState(readSession);

  function signIn(value) {
    sessionStorage.setItem(KEY, JSON.stringify(value));
    setSession(value);
  }

  function signOut() {
    sessionStorage.removeItem(KEY);
    setSession(null);
  }

  return (
    <main className="min-h-screen bg-slate-100 p-4 text-slate-900 sm:p-8">
      {session ? (
        <>
          <header className="mx-auto mb-4 flex max-w-6xl items-center justify-between gap-4">
            <strong>{session.user.name}</strong>
            <button className="rounded-lg border border-slate-300 bg-white px-3 py-2 text-sm" onClick={signOut}>Cerrar sesión</button>
          </header>
          <Chat session={session} onUnauthorized={signOut} />
        </>
      ) : <AuthForm onSession={signIn} />}
    </main>
  );
}
```

**Comprueba:** registra a un usuario de prueba, cierra sesión y vuelve a ingresar. Recarga la página para comprobar que la sesión sigue en esa pestaña. Usa ventanas separadas para las dos cuentas del taller. `sessionStorage` se usa aquí para la práctica local.

## Hora 2 — Construir el chat por componentes

### Paso 07 — Revisar el contrato del backend

En DevTools o Postman, verifica estos formatos antes de construir la pantalla. Todas las rutas de chat requieren `Authorization: Bearer TOKEN`.

| Operación | Ruta | Datos esperados |
| --- | --- | --- |
| Usuarios | `GET /api/users` | Arreglo de usuarios con `id` y `name`. |
| Conversaciones | `GET /api/conversations` | Arreglo de conversaciones del usuario. |
| Crear | `POST /api/conversations` | Enviar `{ title, memberIds: [id] }`. Devuelve la conversación con `id`. |
| Historial | `GET /api/conversations/:id/messages?limit=30` | Objeto `{ messages, nextCursor }`. |

Para mantener el alcance de cuatro horas, la interfaz carga los últimos 30 mensajes. La paginación de mensajes antiguos queda como extensión.

### Paso 08 — Crear los componentes de interfaz

Crea primero los tres componentes que se ocupan de mostrar y recoger datos. Usa `className` para aplicar las utilidades de Tailwind. La cuadrícula del chat se adapta a una columna en pantallas pequeñas.

**Archivo:** `src/components/ConversationSidebar.jsx`.

```jsx
export default function ConversationSidebar({
  users, conversations, selected, member, onMemberChange,
  onRefresh, onCreate, onSelect, creating, blocked = false,
}) {
  return (
    <aside className="min-w-0 rounded-2xl border border-slate-200 bg-white p-4">
      <h2 className="mb-3 text-lg font-semibold">Conversaciones</h2>
      <button className="text-sm font-medium text-indigo-700" type="button"
        onClick={onRefresh}>Actualizar lista</button>

      <form className="my-4 grid gap-2" onSubmit={onCreate}>
        <label className="grid gap-1 text-sm">
          Nuevo chat con
          <select className="w-full rounded-lg border border-slate-300 p-2"
            value={member} onChange={e => onMemberChange(e.target.value)} required>
            <option value="">Selecciona un usuario</option>
            {users.map(person => (
              <option key={person.id} value={person.id}>{person.name}</option>
            ))}
          </select>
        </label>
        <button className="rounded-lg bg-indigo-600 p-2 text-sm font-semibold text-white disabled:opacity-50"
          disabled={creating || blocked || !member}>Crear conversación</button>
      </form>

      <nav className="grid gap-2" aria-label="Conversaciones">
        {conversations.map(chat => (
          <button key={chat.id} type="button" aria-pressed={selected === chat.id}
            disabled={blocked} onClick={() => onSelect(chat.id)}
            className="break-words rounded-lg border border-slate-200 p-2 text-left text-sm aria-pressed:bg-indigo-100 disabled:opacity-50">
            {chat.title || "Conversación sin título"}
          </button>
        ))}
      </nav>
    </aside>
  );
}
```

**Archivo:** `src/components/MessageList.jsx`.

```jsx
export default function MessageList({ messages, userId, loading }) {
  return (
    <>
      {loading && <p className="text-sm text-slate-600" role="status">Cargando historial…</p>}
      {!loading && messages.length === 0 && (
        <p className="my-3 text-sm text-slate-600">Todavía no hay mensajes.</p>
      )}
      <ol className="my-3 flex max-h-96 flex-col gap-2 overflow-y-auto"
        aria-label="Historial de mensajes">
        {messages.map(message => (
          <li key={message.id} className={message.senderId === userId
            ? "max-w-[85%] self-end break-words rounded-xl bg-indigo-100 p-3"
            : "max-w-[85%] self-start break-words rounded-xl bg-slate-100 p-3"}>
            <strong className="text-sm">{message.sender?.name || "Usuario"}</strong>
            <p>{message.content}</p>
          </li>
        ))}
      </ol>
    </>
  );
}
```

**Archivo:** `src/components/MessageComposer.jsx`.

```jsx
export default function MessageComposer({ value, onChange, onSend, enabled, sending = false }) {
  return (
    <form className="flex flex-wrap items-end gap-2" onSubmit={onSend}>
      <label className="grid min-w-0 flex-1 gap-1 text-sm">
        Mensaje
        <input className="w-full rounded-lg border border-slate-300 px-3 py-2"
          value={value} maxLength={4000} disabled={sending}
          onChange={e => onChange(e.target.value)} />
      </label>
      <button className="rounded-lg bg-indigo-600 px-4 py-2 text-white disabled:opacity-50"
        disabled={!enabled || sending || !value.trim()}>
        {sending ? "Enviando…" : "Enviar"}
      </button>
    </form>
  );
}
```

### Paso 09 — Integrar los componentes en la pantalla del chat

**Archivo:** `src/components/Chat.jsx`. Reemplaza el marcador temporal. En esta etapa el historial funciona por REST y el envío permanece deshabilitado.

```jsx
import { useCallback, useEffect, useState } from "react";
import { api } from "../services/api.js";
import ConversationSidebar from "./ConversationSidebar.jsx";
import MessageList from "./MessageList.jsx";
import MessageComposer from "./MessageComposer.jsx";

function mergeMessages(previous, incoming) {
  const byId = new Map(previous.map(message => [message.id, message]));
  for (const message of incoming) byId.set(message.id, message);
  return [...byId.values()].sort((a, b) =>
    Date.parse(a.createdAt) - Date.parse(b.createdAt)
    || a.id.localeCompare(b.id));
}

export default function Chat({ session, onUnauthorized }) {
  const { user, token } = session;
  const [users, setUsers] = useState([]);
  const [conversations, setConversations] = useState([]);
  const [selected, setSelected] = useState("");
  const [member, setMember] = useState("");
  const [byConversation, setByConversation] = useState({});
  const [draft, setDraft] = useState("");
  const [error, setError] = useState("");
  const [creating, setCreating] = useState(false);
  const [loading, setLoading] = useState(false);
  // Conexión en tiempo real: incorporar aquí el paso 10.

  function report(err) {
    if (err.name === "AbortError") return;
    if (err.status === 401) onUnauthorized();
    else setError(err.message);
  }

  async function refreshLists(signal) {
    try {
      const [people, chats] = await Promise.all([
        api("/users", { token, signal }),
        api("/conversations", { token, signal }),
      ]);
      if (signal?.aborted) return;
      setUsers(people);
      setConversations(chats);
    } catch (err) {
      if (!signal?.aborted) report(err);
    }
  }

  useEffect(() => {
    const controller = new AbortController();
    refreshLists(controller.signal);
    return () => controller.abort();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [token]);

  const addMessages = useCallback((id, messages) => {
    setByConversation(previous => ({
      ...previous,
      [id]: mergeMessages(previous[id] || [], messages),
    }));
  }, []);

  useEffect(() => {
    if (!selected) return;
    const controller = new AbortController();
    setLoading(true);
    api(`/conversations/${selected}/messages?limit=30`, {
      token, signal: controller.signal,
    })
      .then(data => {
        if (!controller.signal.aborted) addMessages(selected, data.messages);
      })
      .catch(err => {
        if (!controller.signal.aborted) report(err);
      })
      .finally(() => {
        if (!controller.signal.aborted) setLoading(false);
      });
    return () => controller.abort();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [token, selected, addMessages]);

  async function create(event) {
    event.preventDefault();
    if (!member || creating) return;
    setCreating(true);
    setError("");
    try {
      const person = users.find(item => item.id === member);
      const chat = await api("/conversations", {
        method: "POST", token,
        body: { title: `${user.name} y ${person?.name || "participante"}`, memberIds: [member] },
      });
      await refreshLists();
      setSelected(chat.id);
      setDraft("");
      setMember("");
    } catch (err) {
      report(err);
    } finally {
      setCreating(false);
    }
  }

  function selectConversation(id) {
    setSelected(id);
    setDraft("");
    setError("");
  }

  return (
    <div className="mx-auto max-w-6xl">
      {error && <p className="mb-3 text-sm text-red-700" role="alert">{error}</p>}
      <div className="grid gap-4 md:grid-cols-[16rem_minmax(0,1fr)]">
        <ConversationSidebar
          users={users} conversations={conversations} selected={selected}
          member={member} onMemberChange={setMember} creating={creating}
          onRefresh={() => refreshLists()} onCreate={create}
          onSelect={selectConversation}
        />
        <section className="min-w-0 rounded-2xl border border-slate-200 bg-white p-4">
          <h2 className="text-lg font-semibold">
            {conversations.find(chat => chat.id === selected)?.title || "Chat"}
          </h2>
          {!selected ? <p className="mt-3 text-slate-600">Selecciona una conversación.</p> : (
            <>
              <MessageList messages={byConversation[selected] || []} userId={user.id} loading={loading} />
              <MessageComposer value={draft} onChange={setDraft}
                onSend={event => event.preventDefault()} enabled={false} />
            </>
          )}
        </section>
      </div>
    </div>
  );
}
```

**Comprueba:** registra a Ana en la segunda ventana. Desde la primera, actualiza los usuarios y crea una conversación con Ana. En la segunda pulsa **Actualizar lista** y abre el mismo chat. Si el historial está vacío, envía un mensaje desde Postman a `POST /api/conversations/:id/messages` con `{ "content": "Prueba REST" }` y el JWT de un participante. Recarga y abre la conversación para comprobar que el historial muestra el mensaje. Reduce el ancho de la ventana para verificar que los paneles se apilan.

## Hora 3 — Integrar Socket.IO

### Paso 10 — Abrir una conexión por sesión

**Archivo nuevo:** `src/hooks/useChatSocket.js`. El cliente se conecta al mismo host del backend, sin añadir `/api`. El JWT viaja en `auth.token`, como espera la guía MVC. Consulta las [opciones del cliente Socket.IO](https://socket.io/docs/v4/client-options/).

```javascript
import { useEffect, useRef, useState } from "react";
import { io } from "socket.io-client";
import { BACKEND_URL } from "../services/api.js";

export function useChatSocket(token, onMessage) {
  const socketRef = useRef(null);
  const [connected, setConnected] = useState(false);
  const [connectionError, setConnectionError] = useState("");
  const [connectionCount, setConnectionCount] = useState(0);

  useEffect(() => {
    const socket = io(BACKEND_URL, { auth: { token }, autoConnect: false });
    socketRef.current = socket;

    function onConnect() {
      setConnected(true);
      setConnectionError("");
      setConnectionCount(count => count + 1);
    }
    function onDisconnect() {
      setConnected(false);
    }
    function onError(error) {
      setConnectionError(error.message);
    }

    socket.on("connect", onConnect);
    socket.on("disconnect", onDisconnect);
    socket.on("connect_error", onError);
    socket.on("message:new", onMessage);
    socket.connect();

    return () => {
      socket.off("connect", onConnect);
      socket.off("disconnect", onDisconnect);
      socket.off("connect_error", onError);
      socket.off("message:new", onMessage);
      socket.disconnect();
      socketRef.current = null;
    };
  }, [token, onMessage]);

  function send(conversationId, content) {
    const socket = socketRef.current;
    if (!socket?.connected) return Promise.reject(new Error("El chat está desconectado."));
    return new Promise((resolve, reject) => {
      socket.timeout(5000).emit("message:send", { conversationId, content }, (err, result) => {
        if (err) {
          reject(new Error("No llegó la confirmación. Reabre el chat o recarga para consultar el historial antes de reenviar."));
        } else if (!result?.ok || !result.message) {
          reject(new Error(result?.error || "El servidor rechazó el mensaje."));
        } else {
          resolve(result.message);
        }
      });
    });
  }

  return { connected, connectionError, connectionCount, send };
}
```

**En `src/components/Chat.jsx`:**

1. Agrega este import:

   ```javascript
   import { useChatSocket } from "../hooks/useChatSocket.js";
   ```

2. Sustituye el comentario `// Conexión en tiempo real: incorporar aquí el paso 10.` por el estado de envío:

   ```javascript
   const [sending, setSending] = useState(false);
   ```

3. Justo después de la definición de `addMessages`, antes del efecto del historial, agrega:

   ```javascript
   const receiveMessage = useCallback(message => {
     addMessages(message.conversationId, [message]);
   }, [addMessages]);

   const { connected, connectionError, connectionCount, send } =
     useChatSocket(token, receiveMessage);
   ```

4. Al final del efecto que carga el historial, cambia sus dependencias a `[token, selected, addMessages, connectionCount]`. Así, tras una reconexión, el chat abierto vuelve a consultar REST. Los mensajes se combinan por ID, por lo que una respuesta HTTP atrasada no borra los eventos ya recibidos.
5. Crea `src/components/ConnectionBadge.jsx`:

   ```jsx
   export default function ConnectionBadge({ connected }) {
     return (
       <p role="status" className={connected
         ? "mb-3 inline-block rounded-full bg-emerald-100 px-3 py-1 text-sm text-emerald-800"
         : "mb-3 inline-block rounded-full bg-amber-100 px-3 py-1 text-sm text-amber-900"}>
         {connected ? "Chat conectado" : "Chat desconectado"}
       </p>
     );
   }
   ```

6. Importa `ConnectionBadge` desde `./ConnectionBadge.jsx`. Dentro del primer `div` del JSX de `Chat`, antes de `{error && …}`, añade:

   ```jsx
   <ConnectionBadge connected={connected} />
   {connectionError && <p className="mb-3 text-sm text-red-700" role="alert">
     Socket.IO: {connectionError}
   </p>}
   ```

### Paso 11 — Conectar el formulario de envío

**Archivo:** `src/components/Chat.jsx`. Añade esta función después de `create` y antes del `return` del componente:

```javascript
async function sendMessage(event) {
  event.preventDefault();
  const content = draft.trim();
  if (!selected || !content || !connected || sending || creating) return;

  setSending(true);
  setError("");
  try {
    const saved = await send(selected, content);
    addMessages(saved.conversationId, [saved]);
    setDraft("");
  } catch (err) {
    setError(err.message);
  } finally {
    setSending(false);
  }
}
```

En el JSX de `Chat.jsx`, sustituye la llamada temporal al componente de envío por:

```jsx
<MessageComposer value={draft} onChange={setDraft}
  onSend={sendMessage} enabled={connected && !creating} sending={sending} />
```

Agrega `blocked={sending}` a `ConversationSidebar` y cambia la validación inicial de `create` a `if (!member || creating || sending) return;`. Así el usuario no cambia de conversación mientras espera la confirmación. El componente de envío sigue recibiendo su estado por props.

**Comprueba:** escribe desde una ventana y observa la otra sin recargar. El emisor puede recibir el mismo mensaje por `message:new` y por la confirmación; `mergeMessages` mantiene un único registro por ID. Un tiempo de espera no prueba que el guardado falló, por eso no se reenvía automáticamente.

## Hora 4 — Probar y entregar

### Paso 12 — Ejecutar la matriz de pruebas

Registra dos cuentas, por ejemplo Ana y Luis, en una ventana normal y otra privada. Crea la conversación una sola vez y actualiza la lista en la otra ventana. Para verificar permisos, usa una tercera cuenta en Postman o en otro perfil del navegador.

| Prueba | Acción | Resultado esperado |
| --- | --- | --- |
| Registro y sesión | Crear dos cuentas, salir e iniciar sesión de nuevo. | Cada ventana muestra el usuario correcto. |
| Envío bidireccional | Ana envía «Hola Luis» y Luis responde. | Ambos ven las dos líneas sin recargar, una sola vez por mensaje. |
| Persistencia | Recargar ambas ventanas y abrir el chat. | El historial devuelve ambos mensajes desde PostgreSQL. |
| Reconexión | Poner a Ana temporalmente sin conexión con DevTools, esperar que indique desconexión, enviar desde Luis y volver a conectar a Ana. | Al reconectar, Ana recupera el mensaje por REST. |
| Acceso ajeno | Consultar el historial y enviar por REST con el token de un tercer usuario ajeno. | Ambas solicitudes reciben 403 y el mensaje no se guarda. |
| Diseño adaptable | Reduce la ventana a un ancho de teléfono. | Los paneles se apilan y los controles siguen visibles. |
| Entrada vacía | Escribir solo espacios. | El botón permanece desactivado. |
| Cruce REST y socket | Enviar un mensaje con Postman mientras el otro usuario está conectado. | El navegador recibe el mensaje sin recargar. |

Si DevTools no interrumpe la conexión ya abierta, cierra temporalmente la ventana de Ana, envía desde Luis y vuelve a entrar con Ana para comprobar la recuperación del historial. Puedes probar además la reconexión automática reiniciando el backend con ambas ventanas abiertas.

**Solicitudes para la prueba con un usuario ajeno:** reemplaza el ID por el de la conversación de Ana y Luis. Usa un JWT válido de la tercera cuenta. En Postman, selecciona **Authorization → Bearer Token**.

```http
GET http://localhost:3000/api/conversations/ID_CONVERSACION/messages
Authorization: Bearer TOKEN_TERCER_USUARIO
```

```http
POST http://localhost:3000/api/conversations/ID_CONVERSACION/messages
Authorization: Bearer TOKEN_TERCER_USUARIO
Content-Type: application/json

{"content":"Este mensaje no debe guardarse"}
```

### Paso 13 — Revisar errores y compilar

```bash
bun --bun run build
```

| Si ocurre… | Revisa… |
| --- | --- |
| No se aplican los estilos | Comprueba `tailwindcss()` en `vite.config.js`, la importación en `src/index.css` y que `main.jsx` importe ese archivo. Reinicia Vite. |
| Error de CORS | `CLIENT_ORIGIN` debe coincidir con `http://localhost:5173`; reinicia el backend. |
| REST funciona pero Socket.IO falla | La URL del socket no termina en `/api`; el token se envía en `auth.token`. |
| La conversación no aparece en la otra ventana | Pulsa **Actualizar lista**. La creación se sincroniza por REST en esta versión del cliente. |
| Llegan mensajes repetidos | Usa `message.id` como clave y combina tanto el evento como la confirmación con `addMessages`. |
| El token venció | Cierra sesión e inicia nuevamente. La guía no implementa renovación automática del JWT. |

### Paso 14 — Preparar la entrega

- [ ] Código del frontend con `bun.lock`, `.env.example` y README con comandos de inicio. No incluyas `node_modules/`, `dist/`, archivos de entorno locales ni tokens.
- [ ] Matriz de pruebas con resultado real y evidencia. Adjunta una captura de las dos ventanas conversando y otra del historial recuperado tras una recarga o reconexión.
- [ ] Evidencia del HTTP 403 con una cuenta ajena. Oculta el encabezado de autorización y otros datos de acceso en las capturas.
- [ ] Demostración breve: explicar qué viaja por REST, qué viaja por Socket.IO cómo se evita que un mensaje aparezca dos veces y cómo viajan los datos de `Chat` a sus componentes.

**Si terminas antes:** agrega un botón «Cargar mensajes anteriores» usando `nextCursor` como parámetro `before`. Combina por ID y no borres los mensajes recientes cuando llegue una página anterior.
