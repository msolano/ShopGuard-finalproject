## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 4. Especificación de la API

ShopGuard expone dos APIs FastAPI con documentación OpenAPI automática (`/docs`):

- **API de tienda** (`app/main.py`, puerto 8000): vídeo, cámaras, alertas, estadísticas,
  configuración en caliente y WebSocket de alertas.
- **API del gateway** (`gateway/main.py`, puerto 8080): login, agregación de tiendas,
  estadísticas globales, proxy de streams y WebSocket multiplexado.

Cuando `AUTH_REQUIRED=true`, los endpoints de tienda requieren `X-API-Key` (REST),
`?token=` firmado (MJPEG) o `?api_key=` (WebSocket). `/health` y `/` quedan exentos.

### 4.1. API de tienda (`:8000`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `GET` | `/` | — | Info del servicio |
| `GET` | `/health` | — | Healthcheck (usado por el gateway) |
| `GET` | `/video/stream?cam_id=` | token | MJPEG de una cámara (default cam_id=0) |
| `GET` | `/cameras` | API key | Lista de cámaras con estado de salud |
| `GET` | `/cameras/{id}/stream` | token | MJPEG de la cámara `{id}` |
| `GET` | `/cameras/{id}/stream/token` | API key | Emite token MJPEG corto (TTL 60 s) |
| `GET` | `/cameras/{id}/health` | API key | Salud: `ok` / `degraded` / `disconnected` |
| `POST` | `/cameras/{id}/start` | API key | Enciende la detección de una cámara |
| `POST` | `/cameras/{id}/stop` | API key | Apaga la detección de una cámara |
| `GET` | `/alerts` | API key | Lista alertas (filtros: `level`, `resolved`, `date_from/to`, `limit`, `offset`) |
| `POST` | `/alerts/bulk/resolve` | API key | Resuelve varias (por `alert_ids` o por filtros) |
| `POST` | `/alerts/{id}/resolve` | API key | Resuelve una alerta |
| `GET` | `/alerts/{id}/evidence` | API key | Devuelve el JPG de evidencia |
| `GET` | `/stats` | API key | Estadísticas (hoy, por nivel, sin resolver, por hora) |
| `GET` | `/config` | API key | Configuración actual en caliente |
| `POST` | `/config` | API key | Actualiza `confidence_threshold` / `suspicious_time_seconds` |
| `POST` | `/config/reload` | API key | Recarga `rules.yaml` en los detectores activos |
| `WS` | `/ws/alerts` | `?api_key=` | Push de alertas en tiempo real |

**Ejemplo — listar alertas:**
```http
GET /alerts?level=ALTO&resolved=false&limit=50 HTTP/1.1
X-API-Key: <key>
```
```json
{
  "total": 1,
  "alertas": [{
    "id": 42, "timestamp": "2026-06-10T14:03:11", "level": "ALTO",
    "pattern": "A", "description": "Objeto oculto tras acercar la mano",
    "evidence_path": "evidence/alert_42.jpg", "camera_id": 1, "resolved": false
  }]
}
```

**Ejemplo — resolución masiva (`POST /alerts/bulk/resolve`):**
```json
{"alert_ids": [1,2,3]}          // resuelve esas IDs
{"level": "MEDIO"}              // resuelve todas las MEDIO pendientes
{"date_from": "2026-06-01"}     // resuelve todas las pendientes desde esa fecha
```

### 4.2. API del gateway (`:8080`)

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/api/auth/login` | Login del supervisor (set cookie firmada) |
| `POST` | `/api/auth/logout` | Cierra sesión |
| `GET` | `/api/auth/me` | Usuario de la sesión actual |
| `GET` | `/api/stores` | Lista de tiendas con estado |
| `GET` | `/api/stores/{id}/cameras` | Cámaras de una tienda |
| `GET` | `/api/stores/{id}/stats` | Estadísticas de una tienda |
| `GET` | `/api/stores/{id}/alerts` | Alertas de una tienda (enriquecidas con `store_id`) |
| `POST` | `/api/stores/{id}/alerts/{alert_id}/resolve` | Resuelve una alerta en la tienda |
| `POST` | `/api/stores/{id}/alerts/bulk/resolve` | Resolución masiva en la tienda |
| `GET` | `/api/stores/{id}/alerts/{alert_id}/evidence` | Evidencia (proxy a la tienda) |
| `GET` | `/api/stores/{id}/cameras/{cam_id}/stream` | Stream MJPEG (redirect 302 o proxy) |
| `GET` | `/api/stats/global` | Estadísticas agregadas (`asyncio.gather` + caché 3 s) |
| `WS` | `/api/ws/alerts` | Alertas multiplexadas de todas las tiendas |
| `GET` | `/login`, `/`, `/healthz` | SPA de login, SPA principal, healthcheck |

---

**Prompt 1: Diseño de la API REST de tienda**
*Contexto: Definir los contratos entre backend, dashboard y futuro gateway.*
```
# ROL
Actúa como API designer senior especializado en REST sobre FastAPI y en diseño de contratos
estables (contract-first).

# CONTEXTO
La API de la tienda es el contrato que consumen el dashboard local y el gateway central. Debe
ser clara, versionable de facto y autodocumentada.

# OBJETIVO
Diseñar la API REST de la instancia de tienda.

# RECURSOS A MODELAR
- Cámaras: listar, salud por cámara, stream MJPEG, start/stop.
- Alertas: listar con filtros (level, resolved, rango de fechas) y paginación; resolver
  individual; resolver en bulk; servir la evidencia JPG.
- Estadísticas del sistema.
- Configuración en caliente (ver/actualizar umbrales, recargar reglas).

# REQUISITOS
- Documentación OpenAPI automática en /docs, con tags por recurso.
- Especifica método, ruta, query/path params y forma de request/response por endpoint.
- TRAMPA A EVITAR: las rutas literales (p. ej. /alerts/bulk/resolve) deben declararse ANTES
  que las paramétricas (/alerts/{id}/resolve), o FastAPI matchea "bulk" como id y devuelve 422.

# ENTREGABLE
Tabla de endpoints (método, ruta, auth, descripción) + ejemplos de request/response de los
endpoints de alertas.
```
**Respuesta clave del asistente:**
> "Endpoints por recurso con tags OpenAPI. Para alertas: `GET /alerts` con filtros
> `level/resolved/date_from/date_to/limit/offset`, `POST /alerts/{id}/resolve`,
> `POST /alerts/bulk/resolve` y `GET /alerts/{id}/evidence`. Importante declarar
> `/alerts/bulk/resolve` ANTES que `/alerts/{id}/resolve` para que FastAPI no matchee 'bulk'
> como id."

**Impacto en el proyecto:**
Definió las rutas de `app/main.py` y el orden literal-antes-que-paramétrica para evitar 422.

**Prompt 2: WebSocket de alertas en tiempo real**
*Contexto: Empujar alertas al dashboard sin polling.*
```
# ROL
Eres ingeniero backend especializado en tiempo real (WebSockets) y en la frontera
threading ↔ asyncio.

# CONTEXTO
El dashboard debe recibir las alertas SIN hacer polling. La fuente de alertas son los hilos
SÍNCRONOS de cámara; el servidor WebSocket vive en el event loop ASÍNCRONO de FastAPI. Cruzar
esa frontera mal provoca corrupción de estado o pérdidas.

# OBJETIVO
Diseñar el endpoint WebSocket /ws/alerts para push de alertas en tiempo real.

# REQUISITOS
- Empujar cada alerta nueva a todos los clientes conectados.
- Puente seguro hilo→event-loop: los hilos de cámara encolan en una queue.Queue thread-safe
  que un loop asyncio drena y reenvía (no llamar directamente al WS desde el hilo).
- Ping keepalive cada 30 s y limpieza de clientes muertos/desconectados.
- Seguridad: con AUTH_REQUIRED=true, validar ?api_key= ANTES de aceptar el handshake.

# ENTREGABLES
1. El endpoint /ws/alerts y la función broadcast_alert.
2. El mecanismo de puente queue.Queue → loop de drenado.
3. La integración de la autenticación previa al accept().
```
**Respuesta clave del asistente:**
> "`/ws/alerts` valida con `ws_authenticate` antes de `accept()`, mantiene una lista de
> clientes y hace `broadcast_alert`. Los hilos de cámara encolan en una `queue.Queue` que un
> loop async (`_ws_drain_loop`) drena y reenvía. Ping cada 30 s; los sockets que fallan se
> eliminan."

**Impacto en el proyecto:**
Implementó `/ws/alerts`, `broadcast_alert` y el puente `queue.Queue` → loop async.

**Prompt 3: API de agregación del gateway**
*Contexto: Exponer una API única sobre N tiendas.*
```
# ROL
Actúa como arquitecto de API gateways especializado en agregación/composición de servicios y
en BFF (backend-for-frontend).

# CONTEXTO
El gateway expone UNA API sobre N tiendas ShopGuard, cada una con su propia API REST + WS de
alertas. El consumidor es la SPA del supervisor.

# OBJETIVO
Diseñar la API del gateway.

# REQUISITOS FUNCIONALES
- Autenticación de supervisor con cookie de sesión (login/logout/me).
- Listar tiendas y su estado (online/offline).
- Por tienda: cámaras, stats, alertas, resolver (individual y bulk), evidencia, stream.
- Global: /api/stats/global agregando TODAS las tiendas en PARALELO (asyncio.gather) con
  caché corta para no saturarlas.
- Tiempo real: /api/ws/alerts que MULTIPLEXE las alertas de todas las tiendas, cada una
  enriquecida con su store_id.

# REQUISITO DE STREAMING
El MJPEG debe soportar dos modos según despliegue: redirect (302 a la tienda con token corto)
y proxy (el gateway reenvía los bytes). Seleccionable por configuración (SHOPGUARD_MJPEG_MODE).

# ENTREGABLE
Tabla de endpoints del gateway + explicación del flujo de /api/stats/global y de
/api/ws/alerts (cómo se añade store_id).
```
**Respuesta clave del asistente:**
> "`gateway/main.py` con `/api/auth/*`, `/api/stores`, `/api/stores/{id}/*`,
> `/api/stats/global` (con `asyncio.gather` + caché 3 s) y `/api/ws/alerts` multiplexado.
> El stream usa redirect 302 por defecto (token corto) o proxy httpx según
> `SHOPGUARD_MJPEG_MODE`."

**Impacto en el proyecto:**
Generó la API del gateway, la agregación con `asyncio.gather` y los dos modos MJPEG.
