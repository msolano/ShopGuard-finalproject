## Índice

0. [Ficha del proyecto](#0-ficha-del-proyecto)
1. [Descripción general del producto](#1-descripción-general-del-producto)
2. [Arquitectura del sistema](#2-arquitectura-del-sistema)
3. [Modelo de datos](#3-modelo-de-datos)
4. [Especificación de la API](#4-especificación-de-la-api)
5. [Historias de usuario](#5-historias-de-usuario)
6. [Tickets de trabajo](#6-tickets-de-trabajo)
7. [Pull requests](#7-pull-requests)

---

## 0. Ficha del proyecto

### **0.1. Tu nombre completo:**

Mauricio Solano

### **0.2. Nombre del proyecto:**

ShopGuard — Sistema de Videovigilancia Inteligente con IA para Retail

### **0.3. Descripción breve del proyecto:**

ShopGuard detecta comportamientos sospechosos de robo en tiempo real dentro de comercios.
Procesa el vídeo de una o varias cámaras (USB o IP/RTSP), aplica modelos de visión por
computador (YOLOv8 + MediaPipe Pose) sobre cada frame y, cuando reconoce un patrón de riesgo,
genera una alerta y notifica al responsable por Telegram con la foto del frame sospechoso.
Incluye un dashboard local por tienda y un gateway central que agrega N tiendas detrás de una
sola URL. La detección corre 100% en local, sin servicios de pago en la nube.

### **0.4. URL del proyecto:**

_(pendiente)_

### 0.5. URL o archivo comprimido del repositorio

_(pendiente — se indicará el repositorio del código)_

---

## 1. Descripción general del producto

### **1.1. Objetivo:**

Las tiendas pequeñas y medianas no pueden permitirse personal dedicado a vigilar pantallas de
CCTV de forma continua; la mayoría de cámaras solo sirven para revisar el robo *después* de
que ocurrió. ShopGuard convierte cámaras pasivas en un sistema de **detección proactiva en
tiempo real** y de bajo coste, sin hardware especializado.

**Valor que aporta:**
- Detección en tiempo real sobre hardware común (CPU i5/i7; GPU opcional).
- Alertas accionables por Telegram con evidencia fotográfica del incidente.
- Configurable sin tocar código (umbrales y patrones en `rules.yaml`, recargables en caliente).
- Escalable de una tienda a una cadena mediante un gateway central.

**Para quién:**
- **Operador de tienda:** recibe alertas por Telegram y revisa/gestiona incidentes en el
  dashboard local.
- **Supervisor de cadena:** monitoriza todas las tiendas desde el gateway central.

### **1.2. Características y funcionalidades principales:**

**Detección de patrones de riesgo** (visión por computador):

| Patrón | Nivel | Descripción |
|--------|-------|-------------|
| A | ALTO | Objeto visible → la mano se acerca → el objeto desaparece (ocultamiento) |
| B | MEDIO | Persona quieta más de N segundos en un área (permanencia anómala) |
| C | MEDIO/ALTO | Postura sospechosa (agachado, espalda a cámara, manos al torso) |
| D | ALTO | Objeto entra al *bounding box* de la persona y no reaparece |
| Custom | ALTO | Clase sospechosa detectada por un modelo entrenado a medida |

**Otras funcionalidades:**
- Streaming de vídeo en vivo (MJPEG) por cámara, embebible en el dashboard.
- Notificaciones por Telegram con foto del frame del incidente.
- Dashboard local por tienda (Streamlit) con vídeo, alertas, estadísticas e historial.
- Gestión multicámara con estado de salud (`ok`, `degraded`, `disconnected`) y reconexión
  automática (backoff exponencial 1→2→4→…→60 s).
- Ajuste de umbrales en caliente vía `rules.yaml` + `POST /config/reload`.
- Gateway central multi-tienda con login, estadísticas globales y alertas en vivo
  multiplexadas de todas las tiendas.
- Autenticación opt-in (API key, token MJPEG firmado HMAC, auth de WebSocket) sin romper
  instalaciones legacy.
- Soporte SQLite (piloto) o PostgreSQL (producción) sobre el mismo ORM.

### **1.3. Diseño y experiencia de usuario:**

El sistema ofrece dos interfaces:

- **Dashboard local de tienda (Streamlit, `:8501`):** el operador ve el vídeo en vivo de cada
  cámara, el listado de alertas recientes con su evidencia, estadísticas (alertas de hoy, por
  nivel, por hora) e historial. Se autorrefresca cada 3 s. Permite resolver alertas
  individualmente o en bloque.
- **Panel central del supervisor (SPA vanilla del gateway, `:8080`):** tras login, muestra la
  lista de tiendas con su estado online/offline, estadísticas globales agregadas de la cadena
  y un flujo único de alertas en vivo de todas las tiendas, cada una etiquetada con su tienda
  de origen.

> _Nota: las capturas de pantalla / videotutorial de la experiencia de usuario se adjuntarán
> aquí._

### **1.4. Instrucciones de instalación:**

Requisitos: Python 3.11+, una cámara USB (o cámara IP accesible por red) y conexión a internet
para la descarga inicial del modelo YOLOv8.

```bash
# 1. Crear y activar el entorno virtual
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. Instalar dependencias (la primera vez tarda 5-10 min)
pip install -r requirements.txt
# El modelo YOLOv8n (~6 MB) se descarga automáticamente a models/ al primer arranque.

# 3. Configurar variables de entorno (.env)
#   TELEGRAM_BOT_TOKEN=tu_token
#   TELEGRAM_CHAT_ID=tu_chat_id
#   CAMERA_SOURCE=0
#   CONFIDENCE_THRESHOLD=0.5
#   SUSPICIOUS_TIME_SECONDS=3
#   (Producción) DATABASE_URL=postgresql+psycopg://shopguard:password@localhost:5432/shopguard

# 4. Iniciar API + dashboard de una tienda
./start.sh                       # Windows: start.bat
```

Acceso: dashboard `http://localhost:8501` · API `http://localhost:8000` · docs API
`http://localhost:8000/docs`.

**PostgreSQL local (opcional):**
```bash
docker compose --env-file .env.postgres.example up -d postgres
```

**Gateway central multi-tienda:**
```bash
# Generar credenciales del gateway
python -m gateway.scripts.hash_password          # → SHOPGUARD_GATEWAY_PASS_HASH=...
python -c "import secrets; print(secrets.token_urlsafe(48))"   # → SHOPGUARD_GATEWAY_SECRET
# Configurar .env (USER/PASS_HASH/SECRET) y stores.yaml, luego:
./start_gateway.sh               # SPA en http://localhost:8080/login
```

---

## 2. Arquitectura del Sistema

### **2.1. Diagrama de arquitectura:**

ShopGuard es un **monolito modular**: cada tienda corre en un único proceso Python con
concurrencia por hilos (un `CameraDetector` por cámara) y sin cola de mensajes externa. Sobre
esas instancias de tienda se sitúa, opcionalmente, un **gateway central** que las agrega.

```
┌─────────────────────────────────────────────────────────┐
│              Proceso de tienda (1 por sede)              │
│                                                          │
│  CameraDetector × N          PatternDetector × N         │
│  (hilo por cámara)           (instancia por cámara)      │
│       │  YOLO propio × N           │  rules.yaml         │
│       │  MediaPipe propio × N      │  (recarga en vivo)  │
│       └──────────────┬─────────────┘                     │
│                      │ alertas                           │
│              alert_broadcast_queue (queue.Queue)         │
│                      │                                   │
│         FastAPI  ────┴──── WebSocket _ws_drain_loop      │
│         REST / MJPEG         push a clientes             │
│      + auth opt-in (X-API-Key, ?token=, ?api_key=)       │
│                │                                         │
│           SQLite/PostgreSQL + evidence/                  │
└─────────────────────────────────────────────────────────┘
       ▲                              ▲
       │ Streamlit local (:8501)      │ httpx + WS (gateway)
       │              ┌───────────────┴────────────────┐
       │              │   Gateway central (:8080)       │
       │              │   - SPA (HTML + JS vanilla)     │
       │              │   - /api/* (proxy + cache 3 s)  │
       │              │   - /api/ws/alerts (multiplex)  │
       │              │   - cookie firmada (sesión)     │
       │              └───────────────┬────────────────┘
       │                          Navegador del supervisor
```

**Patrón:** monolito modular con separación por responsabilidad (captura/inferencia ↔ reglas ↔
persistencia ↔ API). La concurrencia es por **hilos** (síncronos) en captura/inferencia y
**asíncrona** (asyncio) en la capa FastAPI/WebSocket; el puente es una `queue.Queue`
thread-safe drenada por un loop async.

**Beneficios:** cero dependencias de infraestructura para arrancar; cada cámara/tienda aislada;
despliegue on-premise sencillo. **Sacrificios:** límite de ~4–8 cámaras por PC (N modelos en
paralelo); SQLite no soporta alta concurrencia (mitigado con PostgreSQL); sin observabilidad
avanzada en el MVP.

**Decisiones de diseño:**

| Decisión | Alternativa | Razón |
|----------|-------------|-------|
| Un YOLO por cámara | Modelo compartido con `Lock` | Tracking `persist=True` aislado por stream |
| `queue.Queue` sync→async | `asyncio.Queue` directa | Los hilos de cámara son síncronos; no es thread-safe |
| YAML para umbrales | Solo variables de entorno | Recarga en caliente sin reiniciar |
| SQLite para piloto | PostgreSQL desde el inicio | Arranque sin dependencias; migración documentada |
| Gateway separado | Multi-tenancy en un proceso | Aísla fallos; cada sede autónoma |

### **2.2. Descripción de componentes principales:**

| Componente | Tecnología | Rol |
|-----------|-----------|-----|
| `app/main.py` | FastAPI | API REST, MJPEG, WebSocket, `POST /config(/reload)` |
| `app/camera_manager.py` | OpenCV + ultralytics + threading | `CameraDetector` (hilo/cámara) y `CameraManager` (ciclo de vida, reconexión) |
| `app/patterns.py` | NumPy + MediaPipe | `PatternDetector`: heurísticas A–D + custom, estado temporal, cooldowns |
| `app/rules_loader.py` | PyYAML | Singleton de `rules.yaml`, recarga en caliente |
| `app/alerts.py` | python-telegram-bot | `send_alert`: evidencia + BD + Telegram + cola WebSocket |
| `app/database.py` | SQLAlchemy 2.0 | ORM (tabla `alerts`) + consultas/estadísticas; SQLite o PostgreSQL |
| `app/auth.py` | hmac / hashlib | Auth opt-in: API key, token MJPEG firmado, auth WebSocket |
| `dashboard/streamlit_app.py` | Streamlit + Plotly | Dashboard local: vídeo, alertas, estadísticas |
| `gateway/main.py` | FastAPI + httpx + websockets | SPA, `/api/stores`, `/api/stats/global`, `/api/ws/alerts`, login |
| `gateway/ws_fanout.py` | asyncio + websockets | 1 task WS por tienda con reconexión exponencial + `store_id` |
| `gateway/cache.py` | asyncio | Caché TTL=3 s con lock por key (anti-stampede) |
| `gateway/auth.py` | itsdangerous + passlib | Cookie de sesión firmada (pbkdf2_sha256) |
| `gateway/static/` | HTML + JS + CSS vanilla | SPA del supervisor (gráficos SVG inline, sin npm) |

### **2.3. Descripción de alto nivel del proyecto y estructura de ficheros**

Organización por **responsabilidad técnica**: backend de tienda (`app/`), dashboard local
(`dashboard/`), gateway central (`gateway/`) y entrenamiento (`training/`). La configuración
vive fuera del código (`.env`, `rules.yaml`, `stores.yaml`).

```
ShopGuard/
├── app/                      # Backend de tienda (FastAPI)
│   ├── main.py               # API REST + MJPEG + WebSocket + /config
│   ├── camera_manager.py     # CameraDetector (hilo/cámara) + CameraManager
│   ├── patterns.py           # PatternDetector: heurísticas A–D + custom
│   ├── rules_loader.py       # Singleton de rules.yaml (recarga en caliente)
│   ├── rules.yaml            # Umbrales, cooldowns, flags enabled
│   ├── alerts.py             # send_alert: evidencia + BD + Telegram + WS
│   ├── database.py           # ORM SQLAlchemy (tabla alerts)
│   ├── auth.py               # Auth opt-in (API key / token MJPEG / WS)
│   └── config.py             # BASE_DIR, cámaras, umbrales, claves
├── dashboard/streamlit_app.py  # Dashboard local de tienda (:8501)
├── gateway/                  # Gateway central multi-tienda (:8080)
│   ├── main.py · ws_fanout.py · cache.py · proxy.py · stores.py · auth.py
│   └── static/               # SPA vanilla (index.html, app.js, style.css, login.html)
├── training/                 # Entrenamiento del modelo custom (YOLOv8)
├── scripts/migrate_sqlite_to_postgres.py
├── postgres/init/            # Init SQL del contenedor Postgres
├── data/ · evidence/ · logs/ · models/   # Datos, evidencias, logs, pesos
├── stores.yaml               # Catálogo de tiendas del gateway
├── docker-compose.yml        # Servicio Postgres local
├── requirements.txt
├── start.sh / start.bat      # Arranque API + dashboard local
└── start_gateway.sh          # Arranque del gateway
```

### **2.4. Infraestructura y despliegue**

Despliegue **on-premise por tienda**: cada sede ejecuta el proceso de detección + API
(`uvicorn app.main:app`, `:8000`) y el dashboard local (`streamlit`, `:8501`). El gateway corre
en una máquina con red a las tiendas (`:8080`). BD por defecto SQLite; PostgreSQL opcional vía
`docker-compose.yml`; migración con `scripts/migrate_sqlite_to_postgres.py`.

| Servicio | Puerto | Arranque |
|----------|--------|----------|
| API de tienda (FastAPI) | 8000 | `start.sh` / `uvicorn app.main:app` |
| Dashboard local (Streamlit) | 8501 | `start.sh` / `streamlit run` |
| Gateway central (FastAPI + SPA) | 8080 | `start_gateway.sh` |
| PostgreSQL (opcional) | 5432 | `docker compose up -d postgres` |

**Proceso de despliegue:** (1) clonar y crear venv; (2) `pip install -r requirements.txt`;
(3) configurar `.env`; (4) `start.sh` por tienda; (5) opcionalmente configurar `stores.yaml` +
credenciales del gateway y `start_gateway.sh`. La migración a auth/gateway es **rolling**:
tienda por tienda, sin downtime de las que aún no han migrado.

### **2.5. Seguridad**

Autenticación **opt-in** (`SHOPGUARD_AUTH_REQUIRED=false` por defecto) para no romper
instalaciones legacy; al activarse protege REST, MJPEG y WebSocket. El gateway falla rápido si
faltan sus credenciales.

| Superficie | Mecanismo | Detalle |
|-----------|-----------|---------|
| REST de tienda | `X-API-Key` | `require_api_key` con `hmac.compare_digest` (timing-safe) |
| MJPEG (`<img>`) | Token corto HMAC en `?token=` | TTL 60 s, ligado al `cam_id`; emitido por `GET /cameras/{id}/stream/token` |
| WebSocket | `?api_key=` en handshake | `ws_authenticate` valida antes de `accept()` |
| Gateway (supervisor) | Cookie de sesión firmada | itsdangerous + pbkdf2_sha256 |
| Secretos | `.env` + interpolación `${VAR}` | Las API keys no se commitean en `stores.yaml` |
| Arranque | Fail-fast | `AUTH_REQUIRED=true` sin key → error; gateway aborta sin `PASS_HASH`/`SECRET` |

Ejemplo (token MJPEG): el navegador no puede enviar headers en `<img src>`, por eso el cliente
pide un token con `GET /cameras/{id}/stream/token` (con `X-API-Key`) y abre
`/cameras/{id}/stream?token=<TOKEN>`; el servidor valida firma HMAC-SHA256 y expiración.

### **2.6. Tests**

Estrategia en **pirámide** (definida en `docs/06_test_suite.md`):
- **Unitarios:** lógica aislada con cámara/YOLO/MediaPipe/Telegram mockeados
  (`test_patterns.py`, `test_alerts.py`, `test_rules_loader.py`, `test_camera_manager.py`).
- **Integración:** API REST con `TestClient` contra SQLite en memoria `sqlite:///:memory:`
  (`test_api.py`, `test_database.py`).
- **E2E:** flujo completo de frame de cámara → alerta → resolución vía API
  (`test_e2e_detection_to_resolution.py`).

Ejemplos de casos: el patrón A dispara al desaparecer un objeto tras acercar la mano; los
cooldowns evitan alertas duplicadas; `GET /alerts` respeta los filtros; con `AUTH_REQUIRED=true`
una request sin `X-API-Key` devuelve `401`.

---

## 3. Modelo de Datos

### **3.1. Diagrama del modelo de datos:**

```mermaid
erDiagram
    ALERTS {
        Integer  id            PK "autoincremental, indexado"
        DateTime timestamp        "UTC, default now(), NOT NULL"
        String   level            "ALTO | MEDIO — NOT NULL"
        String   pattern          "A | B | C | D | custom — NOT NULL"
        Text     description      "descripción legible — NOT NULL"
        String   evidence_path    "ruta al JPG en evidence/ — NULLABLE"
        Integer  camera_id        "cámara origen — default 0, NOT NULL"
        Boolean  resolved         "atendida — default false, NOT NULL"
    }
```

El modelo es deliberadamente minimalista: una sola entidad de persistencia (`alerts`). La
configuración (cámaras, umbrales, tiendas) NO se persiste en BD (vive en `config.py`,
`rules.yaml`, `stores.yaml`) y la evidencia se guarda como archivo en `evidence/` (la fila solo
almacena su ruta).

### **3.2. Descripción de entidades principales:**

**`alerts`** — registra cada detección sospechosa.

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | Integer | PK, autoincrement, index | Identificador único |
| `timestamp` | DateTime | NOT NULL, default `utcnow` | Momento de la detección (UTC) |
| `level` | String(10) | NOT NULL | Nivel de riesgo: `ALTO` o `MEDIO` |
| `pattern` | String(50) | NOT NULL | Patrón que la disparó: `A`/`B`/`C`/`D`/`custom` |
| `description` | Text | NOT NULL | Texto legible del comportamiento |
| `evidence_path` | String(500) | NULLABLE | Ruta al JPG del frame en `evidence/` |
| `camera_id` | Integer | NOT NULL, default 0 | Cámara que originó la alerta |
| `resolved` | Boolean | NOT NULL, default `false` | Si el operador ya la atendió |

**Notas de diseño:** `timestamp` en UTC para que las estadísticas por hora
(`extract('hour', ...)`) funcionen igual en SQLite y PostgreSQL. En la versión multi-tienda, el
`store_id` NO se añade al esquema: el gateway lo enriquece en el *payload* (REST y WebSocket).
Por defecto solo se persisten las alertas `ALTO` (`alerts.save_medium_to_db: false`).

---

## 4. Especificación de la API

Tres endpoints principales de la API de tienda (FastAPI, OpenAPI en `/docs`). Cuando
`AUTH_REQUIRED=true` requieren `X-API-Key`.

```yaml
openapi: 3.0.3
info:
  title: ShopGuard Store API
  version: "1.1"
paths:
  /alerts:
    get:
      summary: Lista alertas con filtros y paginación
      security: [{ ApiKeyAuth: [] }]
      parameters:
        - { name: level,    in: query, schema: { type: string, enum: [ALTO, MEDIO] } }
        - { name: resolved, in: query, schema: { type: boolean } }
        - { name: date_from, in: query, schema: { type: string, format: date } }
        - { name: date_to,   in: query, schema: { type: string, format: date } }
        - { name: limit,  in: query, schema: { type: integer, default: 50, maximum: 200 } }
        - { name: offset, in: query, schema: { type: integer, default: 0 } }
      responses:
        "200": { description: Lista de alertas }

  /alerts/{alert_id}/resolve:
    post:
      summary: Marca una alerta como resuelta
      security: [{ ApiKeyAuth: [] }]
      parameters:
        - { name: alert_id, in: path, required: true, schema: { type: integer } }
      responses:
        "200": { description: Alerta resuelta }
        "404": { description: Alerta no encontrada }

  /stats:
    get:
      summary: Estadísticas (hoy, por nivel, sin resolver, por hora)
      security: [{ ApiKeyAuth: [] }]
      responses:
        "200": { description: Estadísticas del sistema }

components:
  securitySchemes:
    ApiKeyAuth: { type: apiKey, in: header, name: X-API-Key }
```

**Ejemplo — `GET /alerts?level=ALTO&resolved=false&limit=50`:**
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

---

## 5. Historias de Usuario

**Historia de Usuario 1 — Alerta en tiempo real**

> **Como** operador de tienda **quiero** recibir una notificación inmediata por Telegram con la
> foto del momento cuando el sistema detecta un comportamiento sospechoso **para** poder
> reaccionar antes de que el robo se complete.

*Criterios:* notificación Telegram en < 5 s para nivel `ALTO`; la alerta se persiste y se empuja
por WebSocket; cada patrón respeta su cooldown anti-spam; las `MEDIO` no envían Telegram por
defecto.

**Historia de Usuario 2 — Monitoreo en vivo**

> **Como** operador de tienda **quiero** ver el vídeo en directo de todas las cámaras y el
> listado de alertas recientes en un único panel **para** vigilar la tienda y resolver
> incidentes sin tocar la consola.

*Criterios:* stream MJPEG por cámara con estado de salud; resolución de alertas individual y en
bloque; autorefresh cada 3 s; reconexión automática con backoff si una cámara cae.

**Historia de Usuario 3 — Supervisión multi-tienda**

> **Como** supervisor de cadena **quiero** un panel único con login que agregue todas las
> tiendas, sus estadísticas globales y las alertas en vivo de todas **para** monitorizar la
> operación sin abrir el dashboard de cada sede.

*Criterios:* login con cookie firmada; lista de tiendas con estado online/offline;
`/api/stats/global` con totales agregados; WebSocket único con alertas etiquetadas por
`store_id`; las tiendas legacy sin auth siguen funcionando.

---

## 6. Tickets de Trabajo

**Ticket 1 — Backend: motor de patrones de detección (A–D + custom)**

*Tipo:* Backend / Visión por computador · *Estimación:* 13 pts

Implementar `PatternDetector` que, a partir de las detecciones de YOLO y la pose de MediaPipe
por frame, evalúe los patrones A (ocultamiento), B (permanencia), C (postura), D (objeto en
bbox de persona) y las clases del modelo custom, manteniendo estado temporal por cámara y
cooldowns por patrón.
*Criterios de aceptación:* cada patrón se activa con su flag `enabled` en `rules.yaml`; respeta
`cooldown_seconds`; el patrón C solo confía en la pose si YOLO ve una persona con confianza ≥
`require_person_min_confidence`; `reload_from_disk()` aplica nuevos umbrales sin perder estado.
*DoD:* tests unitarios de A y D con detecciones sintéticas; cooldowns verificados.

**Ticket 2 — Base de datos: modelo de alertas + capa de acceso y estadísticas**

*Tipo:* Base de datos / Persistencia · *Estimación:* 8 pts

Diseñar la entidad `alerts` (SQLAlchemy 2.0) y la capa de acceso: listar con filtros y
paginación, resolver (individual y en bulk), servir evidencia y calcular estadísticas, sobre
SQLite y PostgreSQL.
*Criterios de aceptación:* el mismo ORM funciona en ambos motores cambiando solo `DATABASE_URL`;
`get_stats` usa `extract('hour', ...)` portable; `bulk_mark_resolved` acepta ids o filtros;
`timestamp` en UTC.
*DoD:* tests de integración con SQLite en memoria; script de migración SQLite→PostgreSQL.

**Ticket 3 — Frontend: SPA del gateway central multi-tienda**

*Tipo:* Frontend / Gateway · *Estimación:* 13 pts

Construir la SPA vanilla del gateway (sin npm) que muestre la lista de tiendas, estadísticas
globales y alertas en vivo multiplexadas, tras login con cookie firmada. Gráficos en SVG inline.
*Criterios de aceptación:* login funcional; rutas protegidas redirigen a `/login`; vista de
tiendas con estado; `/api/stats/global` agregado; WebSocket `/api/ws/alerts` con `store_id`;
MJPEG embebido (modo redirect). Sin dependencias npm.
*DoD:* SPA navegable end-to-end con ≥ 2 tiendas; tiendas legacy sin auth siguen visibles.

---

## 7. Pull Requests

> El histórico de cambios se organizó en tres bloques principales, documentados con plantilla de
> PR (resumen · cambios principales · checklist).

**Pull Request 1 — feat: autenticación opt-in (API key + token MJPEG + WS)**

Introduce seguridad opcional sin romper instalaciones legacy (`AUTH_REQUIRED=false` por
defecto). Cambios en `app/auth.py` (API key timing-safe, `mint/verify_mjpeg_token`,
`ws_authenticate`), `app/config.py` (claves + fail-fast) y `app/main.py` (dependencies de auth
en REST/MJPEG/WS + `GET /cameras/{id}/stream/token`). *Checklist:* compatibilidad legacy ·
comparaciones timing-safe · `/health` y `/` exentos.

**Pull Request 2 — feat: gateway central multi-tienda con SPA**

Añade el paquete `gateway/` que agrega N tiendas: `main.py` (SPA + `/api/*` +
`/api/stats/global` con `asyncio.gather` + `/api/ws/alerts`), `ws_fanout.py` (1 task WS por
tienda con reconexión exponencial + `store_id`), `cache.py` (TTL=3 s anti-stampede), `auth.py`
(cookie firmada) y `static/` (SPA vanilla). *Checklist:* sin dependencias npm · arranque
fail-fast (`validate_or_die`) · modos MJPEG redirect/proxy · tiendas legacy siguen funcionando.

**Pull Request 3 — feat: soporte PostgreSQL + migración desde SQLite**

Permite PostgreSQL para instalaciones grandes manteniendo SQLite como default. Cambios en
`app/database.py` (`_normalize_database_url` al driver psycopg, `pool_pre_ping`, estadísticas
portables), `docker-compose.yml` (postgres:16-alpine con healthcheck y volumen) y
`scripts/migrate_sqlite_to_postgres.py`. *Checklist:* SQLite sigue siendo el default · cambio de
motor solo con `DATABASE_URL` · migración de datos sin pérdida.
