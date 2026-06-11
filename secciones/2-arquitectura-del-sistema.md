## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 2. Arquitectura del Sistema

### **2.1. Diagrama de arquitectura**

ShopGuard es un **monolito modular**: cada tienda corre en un único proceso Python con
concurrencia por hilos (un `CameraDetector` por cámara) y sin cola de mensajes externa.
Sobre esas instancias de tienda se sitúa, opcionalmente, un **gateway central** que las
agrega para el supervisor de cadena.

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
       │ Streamlit local              │ httpx + WS (gateway)
       │ (:8501)                      │
       │              ┌───────────────┴────────────────┐
       │              │   Gateway central (:8080)       │
       │              │   - SPA (HTML + JS vanilla)     │
       │              │   - /api/* (proxy + cache 3 s)  │
       │              │   - /api/ws/alerts (multiplex)  │
       │              │   - cookie firmada (sesión)     │
       │              └───────────────┬────────────────┘
       │                          Navegador del supervisor
```

**Patrón arquitectónico.** Monolito modular con separación clara de responsabilidades
(captura/inferencia ↔ reglas ↔ persistencia ↔ API). La concurrencia es por **hilos**
(síncronos) en la captura/inferencia y **asíncrona** (asyncio) en la capa FastAPI/WebSocket;
el puente entre ambos mundos es una `queue.Queue` thread-safe (`alert_broadcast_queue`)
drenada por un loop async (`_ws_drain_loop`).

**Decisiones de diseño registradas:**

| Decisión | Alternativa considerada | Razón |
|----------|------------------------|-------|
| Un YOLO por cámara | Modelo compartido con `Lock` | Tracking `persist=True` independiente por stream; sin contaminación de IDs |
| `queue.Queue` sync→async | `asyncio.Queue` directa | Los hilos de cámara son síncronos; `asyncio.Queue` no es thread-safe |
| YAML para umbrales | Solo variables de entorno | Permite recargar en caliente sin reiniciar |
| SQLite para piloto | PostgreSQL desde el inicio | Cero dependencias para arranque rápido; migración a Postgres documentada |
| Gateway separado | Multi-tenancy en un proceso | Aísla fallos por tienda; cada sede sigue siendo autónoma |

**Prompt 1: Diseño de la arquitectura base**
*Contexto: Definir el patrón arquitectónico antes de escribir código.*
```
# ROL
Actúa como arquitecto de software senior especializado en sistemas de visión por computador
en tiempo real y en concurrencia (threading + asyncio) en Python.

# CONTEXTO
Tengo el MVP de ShopGuard definido: detección sobre N cámaras en local (OpenCV + YOLOv8 +
MediaPipe), una API en FastAPI y un dashboard. El problema central de diseño: la captura e
inferencia de vídeo son SÍNCRONAS y bloqueantes por naturaleza, mientras que FastAPI y el
push de alertas por WebSocket son ASÍNCRONOS.

# OBJETIVO
Definir el patrón arquitectónico y el diagrama de la instancia de tienda.

# PREGUNTAS A RESOLVER
- ¿Cómo conviven hilos de cámara síncronos con un servidor asyncio sin condiciones de carrera?
- ¿Un modelo YOLO compartido entre cámaras o uno por cámara? Justifica respecto al tracking.
- ¿Qué estructura de datos hace de puente seguro thread→event-loop?

# RESTRICCIONES
- Monoproceso por tienda, sin broker de mensajes externo (Kafka/Redis) en el MVP.
- Cada cámara debe poder caerse y reconectarse sin afectar a las demás.

# ENTREGABLES
1. Diagrama de la arquitectura de la instancia de tienda (ASCII).
2. Patrón elegido y justificación.
3. Mecanismo de puente sync→async, con su garantía de thread-safety.
4. Tabla de decisiones de diseño con la alternativa descartada y el porqué.
```
**Respuesta clave del asistente:**
> "Monolito modular con un hilo por cámara (`CameraDetector`) para la captura/inferencia y
> FastAPI async para servir. El puente sync→async es una `queue.Queue` thread-safe que un
> loop asyncio drena y reenvía por WebSocket. Un YOLO por cámara evita contaminar los IDs
> de tracking entre streams."

**Impacto en el proyecto:**
Fijó el patrón monolito-modular y el puente `queue.Queue` → `_ws_drain_loop`, base de
`app/camera_manager.py` y la capa WebSocket de `app/main.py`.

**Prompt 2: Diagrama del gateway multi-tienda**
*Contexto: Añadir la capa de agregación sin acoplar las tiendas entre sí.*
```
# ROL
Eres arquitecto de sistemas distribuidos con experiencia en API gateways y agregación de
servicios.

# CONTEXTO
Cada tienda ShopGuard expone su propia API REST + un WebSocket de alertas (/ws/alerts).
Quiero una capa de agregación para el supervisor de cadena, SIN acoplar las tiendas entre sí
ni entre ellas y el gateway (las tiendas siguen siendo autónomas).

# OBJETIVO
Diseñar el gateway central que agrega N tiendas detrás de una sola URL.

# REQUISITOS FUNCIONALES
- Servir una SPA al supervisor.
- Endpoints agregados: /api/stores y /api/stats/global (totales de toda la cadena).
- Multiplexar los WebSocket de todas las tiendas en uno solo para el navegador.
- Cachear respuestas para no saturar las tiendas con peticiones repetidas.

# REQUISITOS NO FUNCIONALES
- Resiliencia: reconexión automática a tiendas caídas, sin tirar el gateway.
- Eficiencia: las consultas a tiendas en paralelo, no secuenciales.

# ENTREGABLE PRINCIPAL
Traza el camino completo de UNA alerta: desde la cámara de una tienda hasta el navegador del
supervisor, indicando cada componente y transformación (incluido cómo se identifica la
tienda de origen). Razona los trade-offs.
```
**Respuesta clave del asistente:**
> "El gateway mantiene 1 task WS persistente por tienda (`ws_fanout.py`) con reconexión
> exponencial, enriquece cada alerta con `store_id` y la reenvía a los clientes de
> `/api/ws/alerts`. Las llamadas REST pasan por una caché TTL=3 s con lock por key
> (anti-stampede) en `cache.py`."

**Impacto en el proyecto:**
Originó `gateway/ws_fanout.py`, `gateway/cache.py` y el flujo de alertas multiplexado.

**Prompt 3: Estrategia de configuración en caliente**
*Contexto: Permitir ajustar umbrales sin reiniciar el proceso de detección.*
```
# ROL
Actúa como ingeniero de plataforma centrado en operabilidad y configuración de sistemas en
producción.

# CONTEXTO
En ShopGuard, el operador necesita calibrar la detección durante la operación real
(sensibilidad, tiempos, cooldowns, activar/desactivar patrones). Reiniciar el proceso
implica perder el estado temporal de los detectores (timers del patrón B, historial del A) y
cortar los streams.

# OBJETIVO
Diseñar un mecanismo de configuración en caliente, con un modelo de precedencia claro.

# REQUISITOS
- Definir qué parámetros viven en variables de entorno (.env) y cuáles en un YAML recargable.
- Recargar el YAML SIN reiniciar el proceso ni perder el estado de los detectores activos.
- Propagar los cambios a todos los PatternDetector en ejecución de forma consistente.
- Un endpoint para forzar la recarga.

# ENTREGABLES
1. Tabla de precedencia parámetro → fuente → cómo se cambia en vivo.
2. Diseño del cargador de reglas (singleton) y del método de recarga sin pérdida de estado.
3. Contrato de los endpoints de configuración (POST /config y POST /config/reload).
Señala los riesgos (p. ej. recarga parcial / inconsistente) y cómo los evitas.
```
**Respuesta clave del asistente:**
> "Dos numéricos (`confidence_threshold`, `suspicious_time_seconds`) en `app.config`
> editables vía `POST /config`; el resto (umbrales de patrones, cooldowns, flags `enabled`)
> en `rules.yaml` recargable con `POST /config/reload`. `reload_from_disk()` aplica reglas
> sin reiniciar hilos ni perder el estado entre frames."

**Impacto en el proyecto:**
Definió `app/rules_loader.py`, `app/rules.yaml` y los endpoints `POST /config` /
`POST /config/reload`.

---

### **2.2. Descripción de componentes principales**

| Componente | Rol |
|-----------|-----|
| `app/main.py` | Aplicación FastAPI: REST, MJPEG, WebSocket, `POST /config(/reload)` |
| `app/camera_manager.py` | `CameraDetector` (hilo + YOLO + MediaPipe por cámara) y `CameraManager` (ciclo de vida, reconexión con backoff) |
| `app/patterns.py` | `PatternDetector`: heurísticas A–D + custom, estado temporal por cámara, cooldowns |
| `app/rules_loader.py` | Singleton YAML: `get_rule()`, `reload_rules()` |
| `app/rules.yaml` | Umbrales de patrones, cooldowns, flags `enabled`, política de persistencia/Telegram |
| `app/alerts.py` | `send_alert`: guarda evidencia, persiste en BD, notifica Telegram, encola para WebSocket |
| `app/database.py` | ORM SQLAlchemy (tabla `alerts`) + funciones de consulta/estadística |
| `app/auth.py` | Auth opt-in: `require_api_key`, `require_mjpeg_token`, `ws_authenticate` |
| `app/config.py` | `BASE_DIR`, rutas, cámaras, umbrales mutables, claves de auth |
| `dashboard/streamlit_app.py` | Dashboard local: vídeo en vivo, alertas, estadísticas, historial |
| `gateway/main.py` | FastAPI :8080: SPA, `/api/stores`, `/api/stats/global`, `/api/ws/alerts`, login |
| `gateway/ws_fanout.py` | 1 task WS por tienda con reconexión exponencial, multiplexado con `store_id` |
| `gateway/cache.py` | Caché TTL=3 s con `asyncio.Lock` por key (anti-stampede) |
| `gateway/auth.py` | Cookie de sesión firmada (itsdangerous + pbkdf2_sha256) |
| `gateway/static/` | SPA 100% vanilla (HTML + JS + CSS, gráficos SVG inline) |

**Prompt 1: Aislamiento de estado por cámara**
*Contexto: Cada cámara debe llevar su propio tracking y estado de patrones.*
```
# ROL
Eres ingeniero de visión por computador con experiencia en pipelines multi-stream y object
tracking (ByteTrack/BoT-SORT en ultralytics).

# CONTEXTO
ShopGuard procesa N cámaras simultáneas en un mismo proceso. Cada cámara necesita su propio
tracking persistente y su propio estado temporal de patrones, sin contaminación entre
streams.

# OBJETIVO
Implementar `CameraDetector` (unidad de captura+inferencia) y `CameraManager` (ciclo de vida).

# REQUISITOS
- Un hilo por cámara, cada uno con su instancia de YOLO (model.track persist=True) y su
  PatternDetector.
- El tracking y el estado temporal (timers del patrón B, historial del A) NO deben mezclarse
  entre cámaras.
- Los IDs de tracker de objetos COCO deben empezar en un rango alto (p. ej. 10000+) para no
  colisionar con los IDs del modelo custom.
- Reconexión automática con backoff exponencial (1→2→4→…→60 s) si una cámara cae.
- Estado de salud por cámara: ok / degraded / disconnected.

# ENTREGABLES
1. Diseño de clases CameraDetector y CameraManager y su API pública.
2. Estrategia de aislamiento de estado y de IDs.
3. Manejo de reconexión y reporte de salud.
```
**Respuesta clave del asistente:**
> "Cada `CameraDetector` mantiene su propio `model.track(persist=True)` y su
> `PatternDetector`. Los IDs de objetos COCO arrancan en 10000 para no colisionar con los
> IDs del modelo custom. `CameraManager` gestiona arranque, parada y reconexión por backoff."

**Impacto en el proyecto:**
Estableció el aislamiento por cámara y el offset de IDs de tracker (10000+).

**Prompt 2: Motor de patrones con estado y cooldowns**
*Contexto: Las heurísticas A–D requieren memoria entre frames y anti-spam.*
```
# ROL
Eres ingeniero de behavior analytics sobre vídeo, experto en máquinas de estado temporales
sobre secuencias de detecciones.

# CONTEXTO
Las alertas de ShopGuard nacen de patrones que requieren MEMORIA entre frames: A
(ocultamiento: objeto presente → mano cerca → objeto ausente), B (permanencia anómala), C
(postura sospechosa con MediaPipe) y D (objeto dentro del bbox de persona durante N
segundos). Además hay clases del modelo custom.

# OBJETIVO
Diseñar `PatternDetector` con estado por cámara, cooldowns por patrón y recarga de umbrales
en caliente.

# REQUISITOS
- Estado temporal por persona/objeto (historiales, temporizadores, IoU acumulada).
- Cooldown por patrón para no spamear el mismo evento (configurable por patrón).
- Gate anti falso-positivo del patrón C: confiar en la pose solo si YOLO ve una persona con
  confianza ≥ umbral (MediaPipe inventa esqueletos en escenas vacías).
- `reload_from_disk()` que aplique nuevos umbrales de rules.yaml SIN perder el estado.

# ENTREGABLES
1. Diseño de PatternDetector y su estado interno.
2. Lógica de cada patrón A/B/C/D + custom con sus parámetros.
3. Mecanismo de cooldowns y de recarga sin pérdida de estado.
```
**Respuesta clave del asistente:**
> "PatternDetector guarda historial de detecciones y timers por persona. Cada patrón aplica
> su cooldown vía `_COOLDOWN_ATTR`. `reload_from_disk()` relee `rules.yaml` y reconfigura
> umbrales/cooldowns en `_apply_rules()` sin reinicializar el estado temporal."

**Impacto en el proyecto:**
Definió `app/patterns.py` con cooldowns por patrón y recarga sin pérdida de estado.

**Prompt 3: Caché anti-stampede del gateway**
*Contexto: Evitar que muchos viewers golpeen las tiendas simultáneamente.*
```
# ROL
Eres ingeniero backend especializado en rendimiento y patrones de caché bajo concurrencia.

# CONTEXTO
El gateway hace fan-out a varias tiendas. Si 10 viewers piden /api/stats/global a la vez, no
quiero generar 10×N llamadas HTTP a las tiendas: es un cache stampede (thundering herd).

# OBJETIVO
Implementar una caché en memoria, asíncrona, anti-stampede, para las respuestas agregadas.

# REQUISITOS
- TTL corto (3 s) por key.
- Lock por key (asyncio.Lock): solo la PRIMERA request rellena la caché; las demás esperan y
  reutilizan el valor fresco, sin disparar peticiones duplicadas.
- 100% asyncio (no bloquear el event loop).
- Combinable con consultas en paralelo a las tiendas (asyncio.gather).

# ENTREGABLES
1. Diseño de la caché (estructura, API get-or-compute).
2. Estrategia de locking por key y por qué evita el stampede.
3. Ejemplo de uso en /api/stats/global con asyncio.gather.
```
**Respuesta clave del asistente:**
> "`cache.py` con TTL=3 s y un `asyncio.Lock` por key: la primera request adquiere el lock
> y rellena, las demás esperan y reutilizan el valor fresco. Combinado con `asyncio.gather`
> en `/api/stats/global` para consultar las tiendas en paralelo."

**Impacto en el proyecto:**
Originó `gateway/cache.py` y el patrón de agregación con `asyncio.gather`.

---

### **2.3. Descripción de alto nivel del proyecto y estructura de ficheros**

ShopGuard separa el **backend de tienda** (`app/`), el **dashboard local** (`dashboard/`),
el **gateway central** (`gateway/`) y los activos de **entrenamiento** (`training/`). La
configuración vive en `.env` + `rules.yaml` + `stores.yaml`. Sigue una organización por
**responsabilidad técnica** más que por capas.

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
├── dashboard/
│   └── streamlit_app.py      # Dashboard local de tienda (:8501)
├── gateway/                  # Gateway central multi-tienda (:8080)
│   ├── main.py               # SPA + /api/* + /api/ws/alerts + login
│   ├── ws_fanout.py          # Multiplexado de WS de N tiendas
│   ├── cache.py              # Caché TTL=3 s anti-stampede
│   ├── proxy.py              # Proxy/redirect MJPEG hacia tiendas
│   ├── stores.py             # Carga de stores.yaml con interpolación ${VAR}
│   ├── auth.py               # Cookie de sesión firmada
│   ├── scripts/hash_password.py
│   └── static/               # SPA vanilla (index.html, app.js, style.css, login.html)
├── training/                 # Entrenamiento del modelo custom (YOLOv8)
│   ├── train.py, dataset.yaml, README_TRAINING.md
├── scripts/
│   └── migrate_sqlite_to_postgres.py
├── postgres/init/            # Init SQL para el contenedor Postgres
├── data/                     # SQLite + datos de Postgres (volumen)
├── evidence/                 # JPGs de evidencia de cada alerta
├── logs/                     # shopguard.log + gateway.log (rotación diaria)
├── models/                   # Pesos YOLOv8 (yolov8n.pt, custom_best.pt)
├── docs/                     # Documentación técnica del proyecto
├── stores.yaml               # Catálogo de tiendas del gateway
├── docker-compose.yml        # Servicio Postgres local
├── requirements.txt
├── start.sh / start.bat      # Arranque API + dashboard local
└── start_gateway.sh          # Arranque del gateway
```

**Prompt 1: Estructura inicial del repositorio**
*Contexto: Organizar el proyecto antes de empezar a desarrollar.*
```
# ROL
Actúa como tech lead que define la estructura de un repositorio Python pensando en su
evolución a 12 meses.

# CONTEXTO
Voy a empezar ShopGuard. Quiero un layout que separe responsabilidades y que no me obligue a
reorganizar todo cuando añada un gateway multi-tienda más adelante.

# OBJETIVO
Proponer la estructura de ficheros y carpetas del proyecto.

# REQUISITOS
- Separar: backend de tienda (FastAPI), dashboard local, lógica de detección
  (cámara/patrones/reglas), persistencia y modelos de IA.
- Configuración de detección FUERA del código (YAML).
- Credenciales en .env (nunca en el código ni en YAML versionado).
- Carpetas para evidencias, logs y pesos de modelos.
- Preparado para añadir un paquete gateway/ hermano sin tocar app/.

# ENTREGABLE
Árbol de directorios comentado (una línea por elemento) y una frase justificando el criterio
de organización (por responsabilidad técnica vs. por capas).
```
**Respuesta clave del asistente:**
> "Estructura por responsabilidad: `app/` (API + detección + BD), `dashboard/` (Streamlit),
> `models/`, `training/`, `evidence/`, `logs/`, con `rules.yaml` para umbrales y `.env`
> para credenciales. El futuro `gateway/` se añade como paquete hermano sin tocar `app/`."

**Impacto en el proyecto:**
Estableció el layout `app/` + `dashboard/` + `gateway/` y la separación
configuración/código.

**Prompt 2: Dónde colocar el gateway sin acoplar**
*Contexto: Incorporar el agregador como módulo independiente.*
```
# ROL
Eres arquitecto que vela por el desacoplamiento y los límites de módulo (module boundaries).

# CONTEXTO
Voy a añadir el gateway central a ShopGuard. Las tiendas son procesos REMOTOS a los que el
gateway llama por HTTP/WebSocket; no comparten memoria ni base de datos.

# OBJETIVO
Definir la estructura interna del paquete gateway/ como módulo autónomo.

# RESTRICCIÓN CRÍTICA
gateway/ NO debe importar nada de app/. Si lo hiciera, acoplaría el panel central al código
de tienda y rompería la autonomía de las sedes.

# REQUISITOS
Define un fichero por responsabilidad: configuración, autenticación de supervisor, caché,
fan-out de WebSockets, proxy/redirect de MJPEG, carga de stores.yaml con interpolación de
variables, y los activos estáticos de la SPA.

# ENTREGABLE
Árbol del paquete gateway/ con una línea por archivo explicando su responsabilidad, y nota
sobre cómo consume a las tiendas (httpx/websockets) sin acoplarse.
```
**Respuesta clave del asistente:**
> "`gateway/` autónomo: `main.py`, `config.py`, `auth.py`, `cache.py`, `ws_fanout.py`,
> `proxy.py`, `stores.py` y `static/` con la SPA vanilla. No importa `app/`; consume las
> tiendas vía httpx/websockets, así cada sede es independiente."

**Impacto en el proyecto:**
Definió `gateway/` como paquete desacoplado con su propia configuración y SPA.

**Prompt 3: Externalizar rutas y home del proyecto**
*Contexto: Evitar rutas hardcodeadas para poder desplegar en cualquier máquina.*
```
# ROL
Eres ingeniero de portabilidad/empaquetado de aplicaciones Python multiplataforma.

# CONTEXTO
En ShopGuard, BASE_DIR está hardcodeado a D:\ShopGuard (Windows). Quiero desplegar también
en Mac/Linux y en rutas arbitrarias sin tocar el código.

# OBJETIVO
Hacer portable la resolución de rutas del proyecto.

# REQUISITOS
- BASE_DIR debe derivar de una variable de entorno SHOPGUARD_HOME.
- Fallback automático al directorio raíz del repositorio si la variable no está definida.
- Todas las rutas derivadas (evidence, logs, base de datos, models) deben colgar de BASE_DIR.
- Crear los directorios necesarios al importar el módulo si no existen.

# ENTREGABLE
Implementación de la resolución de BASE_DIR y de las rutas derivadas, válida en Windows,
macOS y Linux, con su fallback.
```
**Respuesta clave del asistente:**
> "`BASE_DIR = SHOPGUARD_HOME` si está definido, si no `Path(__file__).parent.parent`.
> `EVIDENCE_DIR`, `LOGS_DIR`, `DB_PATH` y `MODELS_DIR` derivan de `BASE_DIR`, que se crea
> al import si no existe."

**Impacto en el proyecto:**
Hizo el proyecto portable (Windows/Mac/Linux) y eliminó la ruta hardcodeada `D:\ShopGuard`.

---

### **2.4. Infraestructura y despliegue**

Despliegue **on-premise por tienda**: cada sede ejecuta el proceso de detección + API
(`uvicorn app.main:app`, puerto 8000) y el dashboard local (`streamlit`, puerto 8501),
arrancados con `start.sh`/`start.bat`. El **gateway** corre en una máquina con red a las
tiendas (`start_gateway.sh`, puerto 8080). La base de datos es **SQLite** por defecto y
**PostgreSQL** (vía `docker-compose.yml`) para instalaciones con más volumen; la migración
se hace con `scripts/migrate_sqlite_to_postgres.py`.

| Servicio | Puerto | Arranque |
|----------|--------|----------|
| API de tienda (FastAPI) | 8000 | `start.sh` / `uvicorn app.main:app` |
| Dashboard local (Streamlit) | 8501 | `start.sh` / `streamlit run` |
| Gateway central (FastAPI + SPA) | 8080 | `start_gateway.sh` |
| PostgreSQL (opcional) | 5432 | `docker compose up -d postgres` |

**Prompt 1: Scripts de arranque multiplataforma**
*Contexto: Levantar API + dashboard con un solo comando en Windows y Mac/Linux.*
```
# ROL
Eres ingeniero DevOps centrado en la experiencia de arranque (DX) en entornos on-premise sin
contenedores.

# CONTEXTO
El operador de tienda no es técnico. Debe poder arrancar todo el sistema de su sede con un
solo comando (o doble clic), tanto en Windows como en Mac/Linux.

# OBJETIVO
Crear los scripts de arranque de una instancia de tienda.

# REQUISITOS
- Activar el entorno virtual (.venv) automáticamente.
- Lanzar la API (uvicorn app.main:app en :8000) y el dashboard (streamlit en :8501) juntos.
- Versión .sh (Mac/Linux) y .bat (Windows) equivalentes.
- En el primer arranque, descargar el modelo YOLOv8n a models/ si no existe, sin intervención.
- Mensajes claros de qué se está levantando y en qué URLs.

# ENTREGABLES
1. start.sh y start.bat.
2. start_gateway.sh para arrancar el gateway por separado.
```
**Respuesta clave del asistente:**
> "`start.sh` y `start.bat` activan el venv, lanzan `uvicorn app.main:app` y
> `streamlit run dashboard/streamlit_app.py` en paralelo. El primer arranque descarga
> `yolov8n.pt` a `models/` automáticamente vía ultralytics."

**Impacto en el proyecto:**
Generó `start.sh`, `start.bat` y `start_gateway.sh`.

**Prompt 2: PostgreSQL local con Docker**
*Contexto: Ofrecer una BD robusta para tiendas con más cámaras/volumen.*
```
# ROL
Eres ingeniero de infraestructura especializado en bases de datos y contenedores.

# CONTEXTO
SQLite es perfecto para el piloto, pero no soporta alta concurrencia cuando varias cámaras
escriben alertas a la vez. Quiero ofrecer PostgreSQL como opción para instalaciones grandes,
SIN forzar a las pequeñas a usarlo.

# OBJETIVO
Añadir un PostgreSQL local containerizado y hacer que el backend pueda usarlo solo por
configuración.

# REQUISITOS
- docker-compose.yml con postgres:16-alpine.
- Healthcheck (pg_isready), variables de entorno parametrizadas con defaults, volumen
  persistente dentro del proyecto (data/postgres) y carpeta de init SQL.
- El backend cambia de motor SOLO definiendo DATABASE_URL; SQLite sigue siendo el default.
- Normalizar URLs postgres:// y postgresql:// al driver psycopg.
- pool_pre_ping para conexiones resilientes.

# ENTREGABLES
1. docker-compose.yml del servicio Postgres.
2. Ajustes en la capa de BD para soportar ambos motores transparentemente.
```
**Respuesta clave del asistente:**
> "`docker-compose.yml` con `postgres:16-alpine`, healthcheck `pg_isready`, volumen
> `./data/postgres` e init en `./postgres/init`. `database.py` normaliza `DATABASE_URL`
> al driver `psycopg` y usa `pool_pre_ping`; SQLite sigue siendo el default."

**Impacto en el proyecto:**
Añadió soporte PostgreSQL opcional y el normalizador de `DATABASE_URL` en `app/database.py`.

**Prompt 3: Migración de SQLite a PostgreSQL**
*Contexto: Mover datos existentes al cambiar de motor.*
```
# ROL
Eres ingeniero de datos especializado en migraciones entre motores SQL.

# CONTEXTO
Una tienda que ya operaba con SQLite quiere pasar a PostgreSQL sin perder su histórico de
alertas. Ambos motores usan el mismo modelo SQLAlchemy (tabla alerts).

# OBJETIVO
Escribir un script de migración de datos SQLite → PostgreSQL.

# REQUISITOS
- Leer la tabla alerts del SQLite origen y volcarla al Postgres destino respetando el
  esquema SQLAlchemy.
- Crear el esquema en destino si no existe (Base.metadata.create_all).
- Inserción por lotes; idempotente en lo posible (no duplicar si se reejecuta).
- Reportar cuántas filas se migraron y validar el conteo final origen vs destino.
- Parámetros de conexión por variables de entorno / argumentos.

# ENTREGABLE
scripts/migrate_sqlite_to_postgres.py funcional, con manejo de errores y log de progreso.
```
**Respuesta clave del asistente:**
> "`scripts/migrate_sqlite_to_postgres.py` abre ambas conexiones con SQLAlchemy, crea el
> esquema en destino con `Base.metadata.create_all` y copia las filas de `alerts` en lotes."

**Impacto en el proyecto:**
Generó `scripts/migrate_sqlite_to_postgres.py`.

---

### **2.5. Seguridad**

La autenticación es **opt-in** (`SHOPGUARD_AUTH_REQUIRED=false` por defecto) para no romper
instalaciones legacy; cuando se activa, protege REST, MJPEG y WebSocket con mecanismos
adaptados a cada caso. El gateway, en cambio, **falla rápido** (`validate_or_die`) si
faltan sus credenciales.

| Superficie | Mecanismo | Detalle |
|-----------|-----------|---------|
| REST de tienda | `X-API-Key` | `require_api_key` compara con `hmac.compare_digest` (timing-safe) |
| MJPEG (`<img>`) | Token corto HMAC en `?token=` | El navegador no envía headers en `<img>`; token firmado con TTL 60 s, emitido por `GET /cameras/{id}/stream/token` |
| WebSocket | `?api_key=` en handshake | `ws_authenticate` valida antes de `accept()` |
| Gateway (supervisor) | Cookie de sesión firmada | itsdangerous + pbkdf2_sha256 (`gateway/auth.py`) |
| Secretos | `.env` + interpolación `${VAR}` | Las API keys de tienda no se commitean en `stores.yaml` |
| `/health` y `/` | Exentos | Permiten al gateway detectar tiendas offline sin credenciales |

**Prompt 1: Autenticación opt-in sin romper lo existente**
*Contexto: Añadir seguridad a instalaciones ya desplegadas sin auth.*
```
# ROL
Actúa como ingeniero de seguridad de aplicaciones (AppSec) con foco en autenticación de APIs
y en migraciones de seguridad sin downtime.

# CONTEXTO
ShopGuard tiene tiendas ya desplegadas SIN autenticación. Necesito añadir auth a la API de
tienda, pero introducirla no puede romper esas instalaciones de un día para otro.

# OBJETIVO
Diseñar una autenticación opt-in para la API REST de tienda.

# REQUISITOS
- Desactivada por defecto: AUTH_REQUIRED=false → todos los chequeos pasan (compatibilidad
  legacy).
- Al activarse, protege los endpoints REST con header X-API-Key.
- Comparación de credenciales TIMING-SAFE (hmac.compare_digest), nunca con ==.
- Fail-fast de configuración: si AUTH_REQUIRED=true pero no hay API key, el proceso debe
  fallar al arrancar (no arrancar inseguro silenciosamente).
- /health y / deben quedar EXENTOS para que el gateway detecte tiendas offline sin credencial.

# ENTREGABLES
1. Módulo de auth con la dependency require_api_key.
2. Variables de configuración y la validación fail-fast.
3. Lista de endpoints exentos y su justificación.
```
**Respuesta clave del asistente:**
> "`app/auth.py` con `require_api_key` como dependency de FastAPI; si `AUTH_REQUIRED` es
> false todos los chequeos pasan. Usa `hmac.compare_digest` contra `cfg.API_KEY`. En
> `config.py`, si `AUTH_REQUIRED=true` y `API_KEY` está vacío, se lanza error al import."

**Impacto en el proyecto:**
Creó `app/auth.py` y el principio auth opt-in que recorre todo el sistema.

**Prompt 2: Streaming MJPEG autenticado en el navegador**
*Contexto: Un `<img src>` no puede enviar el header X-API-Key.*
```
# ROL
Eres ingeniero de seguridad web especializado en autenticación de recursos que el navegador
carga sin control de cabeceras.

# CONTEXTO
El vídeo en vivo se consume con <img src="/cameras/{id}/stream">. Un <img> NO puede enviar
el header X-API-Key, así que el esquema de API key no sirve para proteger el MJPEG. No quiero
exponer la API key en la URL (queda en logs/historial).

# OBJETIVO
Diseñar un esquema de token corto, firmado y efímero para autorizar el stream por query
string.

# REQUISITOS
- El cliente, ya autenticado con X-API-Key, pide un token a GET /cameras/{id}/stream/token.
- El token se firma con HMAC-SHA256, va ligado al cam_id y caduca rápido (TTL 60 s).
- El stream se abre con ?token=... y se valida firma + expiración con comparación timing-safe.
- El secreto de firma deriva de la API key (o de SHOPGUARD_MJPEG_SECRET si se define).

# ENTREGABLES
1. Funciones mint_mjpeg_token / verify_mjpeg_token y su formato de token.
2. Dependency require_mjpeg_token para el endpoint de stream.
3. Endpoint que emite el token tras validar la API key.
```
**Respuesta clave del asistente:**
> "Token `f'{exp_ts}.{hex_sig}'` firmado con HMAC-SHA256 sobre `cam_id:exp_ts`, TTL 60 s.
> `mint_mjpeg_token` lo emite tras validar X-API-Key; `require_mjpeg_token` lo verifica en
> el stream (firma + expiración con `compare_digest`)."

**Impacto en el proyecto:**
Implementó `mint_mjpeg_token`/`verify_mjpeg_token` y el endpoint
`GET /cameras/{id}/stream/token`.

**Prompt 3: Sesión del gateway con cookie firmada**
*Contexto: Login del supervisor sin almacenar sesiones en servidor.*
```
# ROL
Eres ingeniero de seguridad backend especializado en gestión de sesiones stateless.

# CONTEXTO
El gateway necesita login para el supervisor de cadena. Quiero evitar una base de datos de
sesiones: el estado de sesión debe viajar firmado en el cliente.

# OBJETIVO
Implementar autenticación de sesión del gateway basada en cookie firmada.

# REQUISITOS
- Cookie de sesión firmada (itsdangerous) que transporte el usuario autenticado.
- Password de admin almacenado como hash pbkdf2_sha256 (passlib), nunca en claro.
- Script CLI para generar el hash del password admin.
- Fail-fast: el gateway NO arranca si faltan el hash del password o el secret de firma
  (>=32 chars).
- Rutas protegidas redirigen a /login sin sesión válida.

# ENTREGABLES
1. Módulo de auth del gateway (firmar/verificar cookie, verificar password).
2. Script hash_password para generar el hash admin.
3. Validación de arranque (validate_or_die).
```
**Respuesta clave del asistente:**
> "`gateway/auth.py` firma la cookie `shopguard_session` con itsdangerous y verifica el
> password con passlib (pbkdf2_sha256). `gateway/scripts/hash_password.py` genera el hash.
> `validate_or_die` aborta el arranque si faltan `PASS_HASH` o `SECRET`."

**Impacto en el proyecto:**
Creó `gateway/auth.py`, el script `hash_password.py` y el arranque fail-fast del gateway.

---

### **2.6. Tests**

Estrategia de pruebas en **pirámide**: unitarios (lógica aislada con cámara/YOLO/Telegram
mockeados), integración (API REST contra SQLite en memoria `sqlite:///:memory:`) y un flujo
E2E (de frame de cámara hasta resolución de alerta vía API). Documentada en
`docs/06_test_suite.md`.

```
┌──────────────────────────────────────────────┐
│  E2E (1 flujo)                                │
│  test_e2e_detection_to_resolution.py          │
├──────────────────────────────────────────────┤
│  Integración (API + BD real en memoria)       │
│  test_api.py  ·  test_database.py             │
├──────────────────────────────────────────────┤
│  Unitarios (lógica aislada)                   │
│  test_patterns.py · test_alerts.py            │
│  test_rules_loader.py · test_camera_manager.py│
└──────────────────────────────────────────────┘
```

**Prompt 1: Estrategia de pruebas**
*Contexto: Definir qué y cómo testear un sistema con I/O de cámara y modelos pesados.*
```
# ROL
Actúa como QA engineer / SDET especializado en testear sistemas con dependencias de hardware
e I/O externo (cámaras, modelos de IA, APIs de terceros).

# CONTEXTO
ShopGuard depende de cámara (OpenCV), modelos pesados (YOLO, MediaPipe) y un servicio externo
(Telegram). Ejecutar eso en CI es inviable y lento; necesito una estrategia que pruebe la
LÓGICA sin esas dependencias reales.

# OBJETIVO
Definir la estrategia de pruebas completa del proyecto.

# REQUISITOS
- Pirámide de tests: unitarios (lógica pura con cámara/YOLO/MediaPipe/Telegram mockeados),
  integración (API REST contra SQLite en memoria), y un E2E (frame simulado → alerta →
  resolución vía API).
- Definir los fixtures de pytest necesarios (engine sqlite:///:memory:, TestClient, mocks).
- Indicar qué módulo cubre cada nivel.

# ENTREGABLES
1. Diagrama de la pirámide con los archivos de test por nivel.
2. Esqueleto de conftest.py (fixtures de BD en memoria, TestClient, mocks de detección).
3. Lista de casos mínimos por módulo.
```
**Respuesta clave del asistente:**
> "Pirámide de 3 niveles. `conftest.py` con engine SQLite `:memory:` y `TestClient` de
> FastAPI; YOLO/MediaPipe/cámara mockeados con `MagicMock`; Telegram parcheado. Unitarios
> para `patterns`, `alerts`, `rules_loader`, `camera_manager`; integración para `api` y
> `database`; un E2E de detección→resolución."

**Impacto en el proyecto:**
Definió `docs/06_test_suite.md` y la organización de fixtures/mocks de la suite.

**Prompt 2: Tests de las heurísticas de patrones**
*Contexto: Verificar la lógica de A–D sin depender de cámaras reales.*
```
# ROL
Eres SDET especializado en testear máquinas de estado temporales y lógica basada en
secuencias.

# CONTEXTO
PatternDetector decide alertas a partir de SECUENCIAS de detecciones entre frames y de
temporizadores. Quiero probar esa lógica de forma determinista, sin cámara ni YOLO reales.

# OBJETIVO
Escribir tests unitarios de PatternDetector.

# REQUISITOS
- Alimentar el detector con secuencias sintéticas de detecciones (frames = listas de bounding
  boxes / poses), sin I/O real.
- Casos: patrón A (objeto presente → mano cerca → objeto ausente dispara alerta) y patrón D
  (objeto dentro del bbox de persona durante N segundos dispara alerta).
- Verificar que respeta los cooldowns (no duplica alertas dentro del cooldown).
- Verificar que los flags enabled de rules.yaml activan/desactivan cada patrón.
- Inyectar umbrales con un rules.yaml de prueba recargado vía reload_from_disk().

# ENTREGABLE
test_patterns.py con los casos anteriores y aserciones claras sobre alertas emitidas y
cooldowns respetados.
```
**Respuesta clave del asistente:**
> "`test_patterns.py` alimenta `PatternDetector` con secuencias sintéticas de detecciones y
> assertea la emisión de alertas y el respeto de cooldowns. Los umbrales se inyectan con un
> `rules.yaml` de prueba recargado vía `reload_from_disk()`."

**Impacto en el proyecto:**
Definió los casos de prueba de las heurísticas y el uso de detecciones sintéticas.

**Prompt 3: Tests de integración de la API**
*Contexto: Asegurar contratos REST estables para el dashboard y el gateway.*
```
# ROL
Eres SDET especializado en testing de APIs REST y en pruebas de contrato.

# CONTEXTO
El dashboard local y el gateway dependen de los contratos REST de la tienda. Un cambio
accidental rompería a ambos. Quiero tests de integración rápidos contra una BD real pero
efímera.

# OBJETIVO
Escribir tests de integración de la API con el TestClient de FastAPI contra SQLite en memoria.

# REQUISITOS
- Cubrir: crear alertas, listarlas con filtros (level, resolved, date_from/to, paginación),
  resolver una, resolver en bulk, y GET /stats.
- Verificar el contrato exacto de las respuestas (campos y tipos) que consumen dashboard y
  gateway.
- Caso de regresión de seguridad: con AUTH_REQUIRED=true, una request sin X-API-Key debe
  devolver 401.
- Aislamiento entre tests (rollback / BD por test).

# ENTREGABLE
test_api.py con los casos anteriores usando TestClient(app) y la BD en memoria del conftest.
```
**Respuesta clave del asistente:**
> "`test_api.py` usa `TestClient(app)` con la BD en memoria del `conftest`. Verifica los
> contratos de `/alerts`, `/alerts/{id}/resolve`, `/alerts/bulk/resolve` y `/stats`, y que
> con auth activa una request sin `X-API-Key` devuelve 401."

**Impacto en el proyecto:**
Fijó los contratos REST verificados y el caso de regresión de autenticación.
