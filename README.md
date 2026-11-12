# Práctica 2: Ver el protocolo MCP por dentro

## 1. Descripción

En esta práctica se observan los mensajes JSON-RPC 2.0 que intercambian un cliente y un servidor MCP. Primero con el **MCP Inspector** (un cliente ya hecho, con interfaz web) y después con un **cliente mínimo propio** (`cliente_mcp.mjs`, sin dependencias) que habla con el servidor por stdio. En ambos casos el servidor es `@modelcontextprotocol/server-filesystem`, con acceso a la carpeta de la práctica.

Entorno con el que se trabajó:

| Componente | Versión |
|---|---|
| Sistema operativo | Windows 11 Home (10.0.26200), con PowerShell |
| Node.js | v24.22.1 (npm 11.19.1) |
| MCP Inspector | 2.15.0 (según su interfaz y su `clientInfo`) |
| Servidor | `@modelcontextprotocol/server-filesystem`, que se identifica como `secure-filesystem-server` 2.3.0 en `serverInfo` |

## 2. Requisitos y reproducción

_(Se completa al final de la práctica.)_

## 3. Actividad A: el protocolo visto con el MCP Inspector

Comando usado, desde la carpeta de la práctica:

```powershell
npx -y "@modelcontextprotocol/inspector" --config mcp.json
```

El historial exportado está en [`salida/historial_inspector.json`](salida/historial_inspector.json) y tiene **8 entradas**. El Inspector guarda cada solicitud junto con su respuesta en la misma entrada (campos `message` y `response`), por eso la tabla tiene una fila por solicitud o notificación.

### Mensajes en orden cronológico

| # | Hora (UTC) | Dirección | Método | Tipo | `id` |
|---|---|---|---|---|---|
| 1 | 04:05:02.019 | Cliente → Servidor | `initialize` | Solicitud (con respuesta) | 0 |
| 2 | 04:05:04.385 | Cliente → Servidor | `notifications/initialized` | Notificación | — |
| 3 | 04:05:04.436 | Cliente → Servidor | `resources/list` | Solicitud (con respuesta) | 1 |
| 4 | 04:05:04.436 | Cliente → Servidor | `tools/list` | Solicitud (con respuesta) | 2 |
| 5 | 04:05:04.436 | Cliente → Servidor | `resources/templates/list` | Solicitud (con respuesta) | 3 |
| 6 | 04:05:04.437 | **Servidor → Cliente** | `roots/list` | Solicitud (con respuesta) | 0 |
| 7 | 04:06:55.083 | Cliente → Servidor | `tools/call` (`list_directory`, `path: "datos"`) | Solicitud (con respuesta) | 4 |
| 8 | 04:08:47.051 | Cliente → Servidor | `tools/call` (`read_text_file`, `path: "datos/calificaciones.csv"`) | Solicitud (con respuesta) | 5 |

Los mensajes 3, 4 y 5 tienen la misma marca de tiempo; su orden sale de sus `id` (1, 2, 3). El `id` 0 del mensaje 6 no choca con el del mensaje 1: cada lado numera sus propias solicitudes.

### Datos de la inicialización

| Dato | Valor |
|---|---|
| Versión del protocolo que propuso el cliente | `2025-11-25` |
| Versión del protocolo que aceptó el servidor | `2025-11-25` |
| `serverInfo` | `secure-filesystem-server`, versión `2.3.0` |
| Capacidades del servidor | `tools` (`listChanged: true`) y `resources` (`listChanged: true`) |
| `clientInfo` del Inspector | `mcp-inspector`, versión `2.15.0` |
| Capacidades del cliente (Inspector) | `sampling`, `elicitation` (`form`, `url`), `roots` (`listChanged: true`), `tasks` (`list`, `cancel` y `requests` para `sampling/createMessage` y `elicitation/create`) y `extensions` (`io.modelcontextprotocol/tasks`, `io.modelcontextprotocol/ui`, `io.modelcontextprotocol/skills`) |
| Mensaje en dirección contraria | `roots/list`: el **servidor** le pide al cliente sus carpetas raíz. El Inspector respondió `{"roots": []}` |
| Herramientas que ofrece el servidor | 16 (pestaña *Tools* y respuesta de `tools/list`) |
| Segunda herramienta ejecutada | `read_text_file` con `path: "datos/calificaciones.csv"`. Devolvió el contenido del CSV |

### Capturas

Inspector conectado al servidor. En el panel *Messages* se ve el `roots/list` en dirección Servidor → Cliente:

![Inspector conectado](evidencias/01_conectado.png)

Ejecución de `list_directory` con `path: "datos"` y su mensaje `tools/call`:

![tools/call de list_directory](evidencias/02_tools_call.png)

Segunda herramienta: `read_text_file` con `datos/calificaciones.csv`:

![tools/call de read_text_file](evidencias/03_read_text_file.png)

Versión del Inspector que muestra su interfaz:

![Versión del Inspector](evidencias/version.png)

## 4. Actividad B: un cliente MCP mínimo

`cliente_mcp.mjs` se copió de las tres partes de la guía y se ejecutó con `node cliente_mcp.mjs`. Terminó con código de salida 0 y dejó `salida/traza.jsonl` con **7 líneas**: 4 mensajes del cliente y 3 del servidor. La salida completa de la terminal está en [`evidencias/terminal_cliente_B.txt`](evidencias/terminal_cliente_B.txt).

### Los siete mensajes

| # | Dirección | Método o contenido | Tipo de mensaje | `id` | Función del cliente que lo produjo |
|---|---|---|---|---|---|
| 1 | Cliente → Servidor | `initialize` (propone `2025-06-18`, `capabilities: {}`, `clientInfo` `cliente-minimo` 1.0.0) | Solicitud | 1 | `solicitar()` |
| 2 | Servidor → Cliente | `result`: versión `2025-06-18`, capacidades `tools` y `resources`, `serverInfo` `secure-filesystem-server` 2.3.0 | Respuesta | 1 | Lector de líneas (`createInterface`) |
| 3 | Cliente → Servidor | `notifications/initialized` | Notificación | — | `notificar()` |
| 4 | Cliente → Servidor | `tools/list` | Solicitud | 2 | `solicitar()` |
| 5 | Servidor → Cliente | `result.tools`: definición de las 16 herramientas | Respuesta | 2 | Lector de líneas |
| 6 | Cliente → Servidor | `tools/call` con `name: "list_directory"` y `path: "datos"` | Solicitud | 3 | `solicitar()` |
| 7 | Servidor → Cliente | `result.content`: `[FILE] calificaciones.csv` | Respuesta | 3 | Lector de líneas |

**Cómo se empareja cada respuesta con su solicitud:** `solicitar()` asigna un `id` nuevo (`siguienteId++`) y guarda en el mapa `pendientes` la función que resuelve su promesa. Cuando llega una línea del servidor, el lector busca `pendientes.get(mensaje.id)` y la llama. Así cada `await solicitar(...)` termina cuando llega la respuesta con su mismo `id`. La notificación no tiene `id`, no se guarda en `pendientes` y el cliente no espera nada por ella.

Las líneas sueltas de la terminal, como `Secure MCP Filesystem Server running on stdio`, `Effective allowed directories (from the command line): ...` y `Client does not support MCP Roots, ...`, salen del stderr del servidor. No son mensajes del protocolo y no aparecen en la traza: el cliente lanza el servidor con `stdio: ['pipe', 'pipe', 'inherit']`, así que stderr va directo a la terminal y solo stdout se interpreta como JSON-RPC.

### La herramienta `list_directory`

Así la define el servidor en la respuesta a `tools/list`:

- **Descripción:** muestra lo que hay directamente dentro de una carpeta, sin entrar en subcarpetas. Marca cada entrada como `[FILE]` (archivo) o `[DIR]` (carpeta) y devuelve una línea por entrada. La propia descripción recomienda otras herramientas para otros casos: `list_directory_with_sizes` si también quieres tamaños u ordenar por tamaño, y `directory_tree` para ver todo el árbol. Solo funciona dentro de las carpetas permitidas.
- **`inputSchema`:** espera un objeto con un único parámetro obligatorio (`required: ["path"]`), `path`, de tipo texto. Es la carpeta a listar, absoluta o relativa a una carpeta permitida, y debe quedar dentro de las carpetas permitidas.
- **`annotations`:** `readOnlyHint: true` (no modifica nada), `idempotentHint: true` (repetirla con los mismos argumentos no tiene efectos adicionales), `destructiveHint: false` (no borra ni sobrescribe) y `openWorldHint: false` (solo trabaja con un mundo cerrado, el sistema de archivos local permitido, sin servicios externos). Son pistas para el cliente, por ejemplo para decidir si pide confirmación antes de usarla, no garantías.
- Además trae `title: "List Directory"` y un `outputSchema`: el resultado estructurado es un objeto con un campo `content` de tipo texto.

## 5. Actividad C: ampliar la conversación

### Código agregado

En la parte 3 de `cliente_mcp.mjs`, antes del paso 5 (`servidor.stdin.end()`):

```js
// Actividad C.
// Una lectura correcta: archivo dentro de la carpeta permitida.
await solicitar('tools/call', {
  name: 'read_text_file',
  arguments: { path: 'datos/calificaciones.csv' },
});

// Una lectura prohibida: ruta fuera de la carpeta de la practica.
await solicitar('tools/call', {
  name: 'read_text_file',
  arguments: { path: 'C:/Windows/win.ini' },
});

// Un metodo que no existe en el protocolo.
await solicitar('herramientas/inventada', {});
```

Al ejecutar `node cliente_mcp.mjs` (código de salida 0), `salida/traza.jsonl` quedó con **13 líneas**. La salida de la terminal está en [`evidencias/terminal_cliente_C.txt`](evidencias/terminal_cliente_C.txt). Las tres solicitudes nuevas se comportaron así:

| `id` | Solicitud | Resultado |
|---|---|---|
| 4 | `tools/call` `read_text_file` con `datos/calificaciones.csv` | `result` con el contenido completo del CSV |
| 5 | `tools/call` `read_text_file` con `C:/Windows/win.ini` | `result` con `"isError": true` (falla) |
| 6 | `herramientas/inventada` | `error` con código `-32601` (falla) |

> **Nota de privacidad:** el mensaje de error del `id` 5 traía la ruta real de la carpeta de la práctica, con el nombre de usuario del sistema. En `salida/traza.jsonl` y en los archivos `evidencias/terminal_cliente_*.txt` ese nombre se sustituyó por `TU_USUARIO`, como sugiere la guía. Es el único cambio que se hizo a esos archivos.

### Las dos respuestas fallidas (copiadas de la traza)

Respuesta al `id` 5, la lectura prohibida:

```json
{"direccion":"SERVIDOR -> CLIENTE","mensaje":{"result":{"content":[{"type":"text","text":"Access denied - path outside allowed directories: C:\\Windows\\win.ini not in C:\\Users\\TU_USUARIO\\practica2-mcp"}],"isError":true},"jsonrpc":"2.0","id":5}}
```

Respuesta al `id` 6, el método inventado:

```json
{"direccion":"SERVIDOR -> CLIENTE","mensaje":{"jsonrpc":"2.0","id":6,"error":{"code":-32601,"message":"Method not found"}}}
```

### En qué se diferencian

- **`id` 5: error de ejecución de herramienta.** El método `tools/call` existe y los parámetros son válidos, así que el servidor **sí invocó** `read_text_file`. Lo que falló fue el trabajo de la herramienta: la ruta está fuera de las carpetas permitidas. Se reconoce porque la respuesta trae **`result`**, no `error`, con un `content` normal cuyo texto explica el problema ("Access denied - path outside allowed directories") y el campo **`"isError": true`**. Para JSON-RPC esta solicitud se atendió bien. El fallo lo cuenta la herramienta dentro de su resultado, y quien lo lea, por ejemplo un modelo, puede entenderlo y corregirse.
- **`id` 6: error de protocolo.** El servidor **no pudo atender** el mensaje porque `herramientas/inventada` no es un método que conozca, así que no se ejecutó nada. Se reconoce porque la respuesta trae **`error`** en lugar de `result`, con **`code: -32601`** y **`message: "Method not found"`**. Ese código es el que JSON-RPC 2.0 reserva para "método no encontrado".

En los dos casos la respuesta conserva el `id` de su solicitud, así que el cliente las empareja igual que a las respuestas correctas.

## 6. Actividad D: comparar dos clientes

Comparación de los mensajes `initialize` y sus respuestas en `salida/historial_inspector.json` (Inspector) y en `salida/traza.jsonl` (cliente mínimo):

| | Inspector | Cliente mínimo |
|---|---|---|
| Nombre y versión del cliente (`clientInfo`) | `mcp-inspector` 2.15.0 | `cliente-minimo` 1.0.0 |
| Versión del protocolo que propuso el cliente | `2025-11-25` | `2025-06-18` |
| Versión del protocolo que respondió el servidor | `2025-11-25` | `2025-06-18` |
| Capacidades que declaró el cliente | `sampling`, `elicitation` (`form`, `url`), `roots` (`listChanged: true`), `tasks` y `extensions` (`tasks`, `ui`, `skills`) | Ninguna (`capabilities: {}`) |
| ¿El servidor envió una solicitud al cliente? ¿Cuál? | Sí: `roots/list` (el Inspector respondió `{"roots": []}`) | No |

En los dos casos el servidor fue el mismo (`secure-filesystem-server` 2.3.0) y declaró las mismas capacidades (`tools` y `resources`, ambas con `listChanged: true`).

### Por qué el servidor se comportó distinto con cada cliente

- **Versión del protocolo.** El servidor contestó con la misma versión que propuso cada cliente: `2025-11-25` al Inspector y `2025-06-18` al cliente mínimo. Soporta las dos, así que aceptó la propuesta de cada uno. La especificación de MCP dice que, si el servidor soporta la versión pedida, debe responder con esa misma versión.
- **La solicitud `roots/list`.** El Inspector declaró la capacidad `roots`, así que el servidor sabía que podía preguntarle sus carpetas raíz y lo hizo justo después de la inicialización. El cliente mínimo no declaró ninguna capacidad y el servidor no le pidió nada: ningún lado debe usar una capacidad que el otro no declaró. El propio servidor lo dice en su stderr al atender al cliente mínimo: `Client does not support MCP Roots, using allowed directories set from server args` (ver [`evidencias/terminal_cliente_C.txt`](evidencias/terminal_cliente_C.txt)). Con el Inspector, la lista de raíces llegó vacía y, aun así, las llamadas posteriores sobre `datos` funcionaron.

Hubo otra diferencia, pero esa la decidió cada cliente y no el servidor: después de inicializar, el Inspector pidió por su cuenta `resources/list`, `tools/list` y `resources/templates/list` para llenar su interfaz. El cliente mínimo solo pide `tools/list`.

### Experimento: versión de protocolo inexistente

Se cambió temporalmente `protocolVersion` a `'1999-01-01'` y se ejecutó el cliente. Las dos primeras líneas de esa traza están guardadas en [`evidencias/experimento_version_1999.jsonl`](evidencias/experimento_version_1999.jsonl):

```json
{"direccion":"CLIENTE -> SERVIDOR","mensaje":{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"1999-01-01","capabilities":{},"clientInfo":{"name":"cliente-minimo","version":"1.0.0"}}}}
{"direccion":"SERVIDOR -> CLIENTE","mensaje":{"result":{"protocolVersion":"2025-11-25","capabilities":{"tools":{"listChanged":true},"resources":{"listChanged":true}},"serverInfo":{"name":"secure-filesystem-server","version":"2.3.0"}},"jsonrpc":"2.0","id":1}}
```

**Resultado:** el servidor no devolvió un error. Respondió con otra versión que sí soporta, **`2025-11-25`**, la misma que aceptó con el Inspector. Esto coincide con la especificación: si el servidor no soporta la versión pedida, responde con otra que sí soporte, de preferencia la más reciente. La sesión siguió como si nada y la traza tuvo los mismos 13 mensajes, porque el cliente mínimo no revisa la versión de la respuesta. Según la especificación, un cliente real que no soporte la versión que le devuelve el servidor debería desconectarse.

Después se restauró `protocolVersion: '2025-06-18'` y se volvió a ejecutar el cliente. La traza de `salida/traza.jsonl` es la de la actividad C: 13 mensajes y el servidor respondió `2025-06-18`.

## 7. Actividad E: diagrama de secuencia

Sesión completa del cliente mínimo, tal como quedó en la actividad C. Contiene los 13 mensajes de `salida/traza.jsonl`, en orden y con su `id`. Las dos respuestas fallidas están marcadas con su clase de error y con un fondo de color: amarillo para el error de ejecución y rojo para el de protocolo. El cierre no es un mensaje JSON-RPC: el cliente cierra el stdin del servidor y este termina.

```mermaid
sequenceDiagram
    participant C as Cliente
    participant S as Servidor
    Note over C,S: Inicializacion
    C->>S: initialize (id 1)
    S-->>C: result con version, capacidades e identidad (id 1)
    C-)S: notifications/initialized (sin id)
    Note over C,S: Operacion
    C->>S: tools/list (id 2)
    S-->>C: result con las 16 herramientas (id 2)
    C->>S: tools/call list_directory datos (id 3)
    S-->>C: result [FILE] calificaciones.csv (id 3)
    C->>S: tools/call read_text_file datos/calificaciones.csv (id 4)
    S-->>C: result con el contenido del CSV (id 4)
    C->>S: tools/call read_text_file C:/Windows/win.ini (id 5)
    rect rgba(255, 190, 0, 0.25)
    S-->>C: ERROR DE EJECUCION - result con isError true, Access denied (id 5)
    end
    C->>S: herramientas/inventada (id 6)
    rect rgba(255, 0, 0, 0.18)
    S-->>C: ERROR DE PROTOCOLO - error -32601 Method not found (id 6)
    end
    Note over C,S: Cierre
    Note over C,S: el cliente cierra stdin y el servidor termina (no es un mensaje JSON-RPC)
```

## 8. Problemas encontrados

_(Se completa al final de la práctica.)_

## 9. Fuentes

_(Se completa al final de la práctica.)_
