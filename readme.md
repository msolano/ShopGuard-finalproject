## Índice

0. [Ficha del proyecto](#0-ficha-del-proyecto)
- [🧭 Metodología: Ecosistema 360 (SDD con CP0-CP5)](#-metodología-ecosistema-360-sdd-con-cp0-cp5)
- [🚀 Despliegue y evidencia de funcionamiento](#-despliegue-y-evidencia-de-funcionamiento)
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

https://github.com/msolano/ShopGuard

### 0.5. URL o archivo comprimido del repositorio

https://github.com/msolano/ShopGuard

> El repositorio del código es **privado**. Se ha concedido acceso (lectura y escritura)
> al equipo del curso: **jorge@lidr.co** y **julio@lidr.co**.

---

## 🧭 Metodología: Ecosistema 360 (SDD con CP0-CP5)

Este proyecto no solo se construyó *con* IA: se construyó **con un método de desarrollo guiado
por especificación (SDD, *Spec-Driven Development*)** llamado **Ecosistema 360**, instalado en el
propio repo (`.ecosystem360/`). La regla de oro es **"contrato antes que código"**: ninguna
feature sustancial se implementa sin su especificación aprobada. Esto convierte al asistente de IA
en un ingeniero disciplinado y trazable, no en un generador de código suelto.

### Ciclo de vida por feature — CP0 → CP5

Cada feature relevante atraviesa **seis puntos de control (checkpoints)** con aprobación humana en
las fronteras:

| CP | Etapa | Salida |
|----|-------|--------|
| **CP0** | Intake | Contexto, fuentes, no-goals, preguntas abiertas |
| **CP1** | SRS | Problema, usuarios, necesidades (NEED-xxx), riesgos, trazabilidad |
| **CP2** | **SPEC** *(frontera)* | Requisitos (REQ-/NFR-), seguridad, criterios de aceptación, plan de pruebas |
| **CP3** | Plan de implementación | Tareas por capa (T1…Tx), contratos, **Repo Paths** (spec ↔ código) |
| **CP4** | Validación | Resultados de pruebas, hallazgos, revisión de seguridad, **TRUST 5** |
| **CP5** | Cierre | Memoria compartida, decisiones (ADR), aprendizajes, backlog |

Complementos: **decision-log** y **risk-register** por feature, ADRs para decisiones técnicas, y
la checklist **TRUST 5** (Tested · Readable · Unified · Secured · Trackable) en cada cambio.
Verificación de gobierno automatizada con `verify-governance.sh`.

### Features entregadas con el ciclo completo CP0-CP5

| Feature | Qué aporta | Artefactos |
|---------|-----------|------------|
| `stores-cameras-management` | Gestión de cámaras y tiendas **desde la UI**, en caliente, persistida en Postgres, con credenciales **cifradas en reposo** | CP0-CP5 + decision-log + risk-register |
| `concealment-any-object` | Detección de ocultamiento de **cualquier objeto** (modo configurable) + fix de detección de personas | CP0-CP5 + decision-log |
| `stores-cameras-management/detección` | Detección de cámaras USB conectadas (`GET /cameras/detect`) | Iteración sobre la feature |

> Estos artefactos viven en `.ecosystem360/work/features/<fecha>-<slug>/` del repo de código.
> Los prompts que guiaron cada checkpoint están documentados en [`prompts.md`](./prompts.md).

**Por qué importa:** el método garantiza que cada decisión tenga su *por qué* registrado, que la
seguridad se piense antes de codificar (no después), y que el trabajo sea auditable de principio a
fin — exactamente lo que se espera de desarrollo asistido por IA hecho con rigor.

---

## 🚀 Despliegue y evidencia de funcionamiento

El sistema está diseñado para **despliegue on-premise** (una instancia por tienda + gateway
opcional). No hay un entorno público permanente (la detección procesa vídeo local por
privacidad); a continuación, cómo ejecutarlo y evidencia del sistema **funcionando en vivo**.

### Cómo ejecutarlo (3 pasos)

```bash
# 0) Requisitos: Python 3.11+, una cámara USB, Docker (para Postgres).
git clone https://github.com/msolano/ShopGuard.git && cd ShopGuard
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt          # la primera vez descarga YOLOv8n (~6 MB)

# 1) Base de datos (Postgres en Docker; o usa SQLite por defecto sin este paso)
docker compose --env-file .env.postgres.example up -d postgres

# 2) Arrancar API + dashboard de la tienda
./start.sh          # API :8000 (docs en /docs) · Dashboard :8501

# 3) (Opcional) Gateway central multi-tienda
./start_gateway.sh  # SPA :8080
```

Accesos: **Dashboard** http://localhost:8501 · **API docs** http://localhost:8000/docs ·
**Gateway** http://localhost:8080.

### Evidencia (verificada en esta máquina)

**1. API en ejecución — Swagger `/docs`** (todos los endpoints, incluidos los de gestión y
detección de cámaras):

![API Swagger docs](evidencia/api-docs-swagger.png)

**2. Cámaras corriendo** (`GET /health`) — dos cámaras activas, detección en tiempo real:

```json
{ "camaras": [
  { "cam_id": 0, "name": "MacBook Pro (built-in)", "health": "ok", "fps": 15.0, "model": "custom_best.pt" },
  { "cam_id": 1, "name": "EMEET SmartCam S600",   "health": "ok", "fps": 14.2, "model": "custom_best.pt" }
] }
```

**3. Detección de cámaras conectadas** (`GET /cameras/detect`) — reporta índices en uso vs. libres:

```json
{ "detectadas": [
  { "index": 0, "status": "en_uso" }, { "index": 1, "status": "en_uso" },
  { "index": 2, "status": "no_detectada" }, { "index": 3, "status": "no_detectada" }
] }
```

**4. Sistema funcionando en una tienda real** — la cámara procesa las estanterías en vivo con las
detecciones dibujadas (HUD con FPS y contador de alertas):

![Snapshot en tienda real](evidencia/snapshot-tienda.jpg)

Clip animado (30 fotogramas en vivo, el contador de alertas incrementa durante la captura):
[`evidencia/demo-tienda-live.gif`](evidencia/demo-tienda-live.gif).

**5. Detección del flujo principal (Patrón D)** — prueba física real (objeto guardado en el
bolsillo) que dispara el **Patrón D [ALTO]**, registrada en el log del sistema:

```
[ALTO] Patrón D — Objeto 'cell phone' entró al área corporal de la persona y no reapareció.
[ALTO] Patrón D — Objeto 'concealing_object' entró al área corporal de la persona y no reapareció.
```

**6. Suite de pruebas** — `pytest -q tests/` → **86 passed** (ver §2.6).

**7. Presentación del proyecto** (arquitectura + call-flow): [`evidencia/presentacion-shopguard.html`](evidencia/presentacion-shopguard.html).

> **Vídeo del flujo principal (recomendado):** por ser el repositorio de código privado, se
> recomienda anexar un vídeo breve (2–3 min) mostrando: alta de una cámara desde la UI, el vídeo en
> vivo con detección, y una alerta disparándose. *(Placeholder para el enlace del vídeo.)*

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

| Patrón | Nombre | Nivel | Qué detecta |
|--------|--------|-------|-------------|
| A | Ocultamiento de objeto | ALTO | Un producto estaba a la vista, una mano se le acerca y el producto **desaparece** del campo de la cámara |
| B | Permanencia anómala | MEDIO | Una persona se queda **quieta en el mismo sitio** más de N segundos |
| C | Postura sospechosa | MEDIO/ALTO | El cuerpo adopta una **pose de ocultar**: agacharse de golpe, dar la espalda a la cámara o llevarse las manos al torso |
| D | Objeto pegado al cuerpo | ALTO | Un producto **entra en la silueta** de la persona (bolsillo, bolso, chaqueta) y ya no vuelve a verse |
| E | Objeto en bolso/mochila | ALTO | Un producto **entra en un bolso o mochila** y desaparece de la vista |
| F | Salida sin pago | ALTO | Una persona **cruza la zona de salida** poco después de un ocultamiento *(zona `exit`; off por defecto)* |
| G | Zona restringida | MEDIO/ALTO | Una persona **entra a un área no permitida** (tras el mostrador, almacén) *(zona `restricted`)* |
| H | Coordinación de grupo | MEDIO | **Varias personas** juntas de forma sostenida — posible ORC *(off por defecto)* |
| I | Merodeo repetitivo | MEDIO | Una persona hace **idas y vueltas** repetidas frente a un área |
| Custom | Modelo entrenado | ALTO | Un modelo entrenado a medida reconoce una clase de riesgo (p. ej. `hand_in_pocket`, `concealing_object`) |

> Los patrones que dependen de una zona (F `exit`, G `restricted`) y la variante B′ (permanencia
> acotada a zona `risk`) requieren definir **zonas tipadas** en `employee_zones` de
> [`app/rules.yaml`](app/rules.yaml) con el campo `role`. Sin zonas, esos patrones no disparan.

**Ejemplos reales (cómo se ve cada patrón en tienda):**

- **A — Ocultamiento de objeto.** Sobre el estante hay una botella de perfume. Un cliente la
  toma, acerca la mano al cuerpo y la botella deja de verse en cámara. → *alerta ALTO*.
  Es el caso clásico de "se lo guardó". *(Un cliente que solo mueve el producto de sitio, sin
  ocultarlo, no dispara: el objeto sigue visible.)*
- **B — Permanencia anómala.** Alguien lleva 40 s parado sin moverse frente a un expositor de
  gafas en una zona sin personal. → *alerta MEDIO*. Sirve para vigilar zonas restringidas.
  ⚠️ Genera falsos positivos con clientes que solo miran el estante, por eso viene
  **desactivado por defecto** (`patterns.B.enabled: false` en `app/rules.yaml`).
- **C — Postura sospechosa.** Una persona se agacha bruscamente detrás de una góndola y se
  lleva ambas manos al torso. → *alerta MEDIO o ALTO* según cuántos indicadores coincidan.
  Detecta el "manipular algo escondido" aunque no se vea el producto.
- **D — Objeto pegado al cuerpo.** Un desodorante se solapa con la silueta de la persona
  durante 3-4 s y ya no reaparece: se lo metió al bolsillo o al bolso. → *alerta ALTO*.
- **E — Objeto en bolso/mochila.** Un cliente abre su mochila junto al estante, mete una caja
  de maquillaje y la caja deja de verse. → *alerta ALTO*. Es el "concealment en bolso" clásico;
  como D, pero el contenedor es el bolso en vez del cuerpo.
- **F — Salida sin pago.** Segundos después de un ocultamiento (A/D/E), la persona cruza la
  franja de la puerta marcada como zona `exit`. → *alerta ALTO*. Es una **heurística sin caja
  registradora**, por eso viene **desactivada por defecto** y solo actúa si defines una zona
  `exit`.
- **G — Zona restringida.** Alguien pasa detrás del mostrador o entra al almacén (área marcada
  como zona `restricted`). → *alerta MEDIO* (o ALTO si la configuras así). La sola presencia
  basta; no necesita ocultamiento.
- **H — Coordinación de grupo.** Tres o más personas permanecen juntas de forma sostenida
  (patrón típico de banda que distrae y dispersa). → *alerta MEDIO*. Heurística cruda (no
  distingue una familia), **desactivada por defecto**.
- **I — Merodeo repetitivo.** Una persona va y viene varias veces frente al mismo expositor
  sin decidirse (preparando el hurto). → *alerta MEDIO*. Cuenta las inversiones de dirección.
- **Custom — Modelo entrenado.** El modelo propio marca la clase `concealing_object` con
  confianza ≥ 0.80. → *alerta ALTO*. Es la vía para reglas específicas de tu negocio (incluida,
  por ejemplo, la remoción de etiqueta antihurto si entrenas esa clase).

> Los nombres legibles aparecen también en el dashboard junto a la letra (ej. *Patrón A ·
> Ocultamiento de objeto*). Los umbrales de cada patrón (segundos, sensibilidad, cooldown) se
> ajustan en [`app/rules.yaml`](app/rules.yaml) y se recargan en caliente con
> `POST /config/reload`.

**Ocultar "cualquier objeto" (para tiendas que venden de todo).** Por defecto los patrones de
ocultamiento (A/D/E) vigilan una lista corta de objetos comunes. Con `patterns.concealment.mode:
any` en `rules.yaml`, **cualquier** objeto que el modelo reconozca cuenta como ocultable; con
`extra_classes: [remote, book, …]` puedes ampliar la lista sin ir al modo completo. El modo `any`
sube los falsos positivos (un cliente que guarda su propio celular también dispara), por eso se
puede acotar a **zonas `risk`** con `any_risk_zone_only`. Además la gestión de **cámaras y tiendas
se hace desde la UI** (dashboard y SPA del gateway), persistida en base de datos y con las
credenciales cifradas en reposo.

> **Gestión sin tocar archivos:** desde la v1.2 las cámaras de la tienda, sus ajustes y las
> tiendas del gateway se dan de alta/editan **desde la interfaz** (persistido en Postgres); `.env`
> y `stores.yaml` solo siembran el primer arranque.

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
| `app/patterns.py` | NumPy + MediaPipe | `PatternDetector`: heurísticas A–I + custom, estado temporal, cooldowns |
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
│   ├── patterns.py           # PatternDetector: heurísticas A–I + custom
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

Suite con **`pytest`** — **86 tests en verde** (`.venv/bin/python -m pytest -q tests/`), con
cámara/YOLO/MediaPipe/Telegram mockeados y BD aislada en temporales (`tests/conftest.py` fija
`SHOPGUARD_HOME`/`DATABASE_URL` antes de importar). Cobertura por área:

| Archivo | Qué cubre |
|---------|-----------|
| `test_patterns.py` · `test_patterns_concealment.py` | Patrones A–I, zonas, modo de ocultamiento `any`/`whitelist`, fracción de contención |
| `test_crypto.py` | Cifrado en reposo (round-trip Fernet) + enmascarado de credenciales |
| `test_config_store.py` · `test_store_repo.py` | Persistencia de cámaras/ajustes/tiendas, seed idempotente, límites |
| `test_zones.py` | Supresión por zona de empleado (point-in-polygon, normalización) |
| `test_labeling.py` · `test_dataset_export.py` | Etiquetado de evidencias y export de dataset YOLO |
| `test_telegram_sender.py` · `test_setup_telegram.py` | Envío no-bloqueante y wizard de configuración |

Ejemplos de casos: el patrón D dispara al desaparecer un objeto dentro de la silueta y no
reaparecer; en modo `any` cualquier objeto cuenta y en `whitelist` no; el round-trip de cifrado
recupera el valor y el enmascarado nunca expone la credencial; el seed no duplica en el segundo
arranque. La verificación **en vivo** (cámara real → Patrón D → alerta) está documentada en la
sección de despliegue.

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
    CAMERA_CONFIGS {
        Integer  id            PK "0..7, estable = cam_id"
        String   name             "nombre visible — NOT NULL"
        Text     source_enc       "fuente USB/RTSP CIFRADA — NOT NULL"
        Boolean  enabled          "arranca al iniciar — default true"
        DateTime created_at        "auditoría"
        DateTime updated_at        "auditoría"
    }
    STORE_SETTINGS {
        String key            PK "clave del ajuste"
        Text   value_json        "valor serializado JSON — NOT NULL"
    }
    GATEWAY_STORES {
        String  id            PK "slug estable de la tienda"
        String  name             "etiqueta visible — NOT NULL"
        String  api_url          "URL de la tienda — NOT NULL"
        Text    api_key_enc      "api_key CIFRADA — NULLABLE"
        Boolean active           "monitoreada — default true"
    }
```

Persistencia (SQLAlchemy, Postgres en producción / SQLite fallback): las alertas y, desde
la feature `stores-cameras-management`, **la configuración operativa** (cámaras, ajustes de
tienda y tiendas del gateway) viven en BD, que es la **fuente de verdad**. `.env`/`stores.yaml`
solo *siembran* (idempotente) el primer arranque. Las credenciales (RTSP, api_key) se guardan
**cifradas en reposo** (Fernet, `app/crypto.py`) y se devuelven enmascaradas. La evidencia sigue
como archivo en `evidence/` (la fila solo guarda la ruta). El diagrama omite las tablas de
soporte `zones`, `image_annotations` e `image_reviews` (features de zonas y etiquetado).

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

**`camera_configs`** — cámaras de la tienda local (feature `stores-cameras-management`).

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | Integer | PK (no autoincrement) | Estable 0..7, usado como `cam_id` y en las URLs de stream |
| `name` | String(120) | NOT NULL | Nombre visible de la cámara |
| `source_enc` | Text | NOT NULL | Fuente (índice USB o URL RTSP) **cifrada** en reposo |
| `enabled` | Boolean | NOT NULL, default `true` | Si arranca su detector al iniciar |
| `created_at` / `updated_at` | DateTime | NOT NULL | Auditoría |

**`store_settings`** — ajustes de la tienda local como clave→valor (JSON): `store_name`,
`confidence_threshold`, `suspicious_time_seconds`, `hours_enabled`, `open_hour`, `close_hour`.

**`gateway_stores`** — tiendas monitoreadas por el gateway (fuente de verdad, reemplaza
`stores.yaml` como config operativa).

| Campo | Tipo | Restricciones | Descripción |
|-------|------|---------------|-------------|
| `id` | String(64) | PK | Slug estable de la tienda (URLs `/api/stores/{id}/...`) |
| `name` | String(160) | NOT NULL | Etiqueta visible |
| `api_url` | String(300) | NOT NULL | URL de la API de la tienda |
| `api_key_enc` | Text | NULLABLE | api_key **cifrada** en reposo (enmascarada en la API) |
| `active` | Boolean | NOT NULL, default `true` | Si el gateway la agrega y suscribe al fan-out |

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

  /cameras:
    get:
      summary: Lista cámaras (config persistida + estado de runtime)
      security: [{ ApiKeyAuth: [] }]
      responses:
        "200": { description: Lista de cámaras (fuente enmascarada) }
    post:
      summary: Alta de cámara en caliente (feature stores-cameras-management)
      security: [{ ApiKeyAuth: [] }]
      requestBody:
        content: { application/json: { schema: { type: object,
          properties: { name: {type: string}, source: {type: string}, enabled: {type: boolean} },
          required: [name, source] } } }
      responses:
        "201": { description: Cámara creada y arrancada }
        "409": { description: Se alcanzó el máximo de cámaras }
        "422": { description: Fuente inválida (USB 0-7 o URL rtsp/http) }

  /cameras/{cam_id}:
    put:
      summary: Edita una cámara y aplica el cambio en caliente
      security: [{ ApiKeyAuth: [] }]
      parameters: [{ name: cam_id, in: path, required: true, schema: { type: integer } }]
      responses: { "200": { description: Cámara actualizada }, "404": { description: No encontrada } }
    delete:
      summary: Detiene y elimina una cámara en caliente
      security: [{ ApiKeyAuth: [] }]
      parameters: [{ name: cam_id, in: path, required: true, schema: { type: integer } }]
      responses: { "200": { description: Cámara eliminada }, "404": { description: No encontrada } }

  /store/config:
    get:
      summary: Ajustes persistidos de la tienda (nombre, umbrales, horario)
      security: [{ ApiKeyAuth: [] }]
      responses: { "200": { description: Ajustes de la tienda } }
    put:
      summary: Edita y persiste los ajustes; los umbrales se aplican en caliente
      security: [{ ApiKeyAuth: [] }]
      responses: { "200": { description: Ajustes guardados }, "422": { description: Valor inválido } }

components:
  securitySchemes:
    ApiKeyAuth: { type: apiKey, in: header, name: X-API-Key }
```

**Gateway — gestión de tiendas** (feature `stores-cameras-management`, autenticado por sesión
firmada `shopguard_session`): `GET /api/admin/stores` (lista con api_key enmascarada),
`POST /api/admin/stores` (alta + suscripción al fan-out en caliente),
`PUT /api/admin/stores/{id}` y `DELETE /api/admin/stores/{id}`.

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

---

> Las siguientes features se desarrollaron con el **ciclo completo CP0-CP5** del Ecosistema 360
> (ver [Metodología](#-metodología-ecosistema-360-sdd-con-cp0-cp5)). PR agregado:
> **[msolano/ShopGuard#1](https://github.com/msolano/ShopGuard/pull/1)**.

**Pull Request 4 — feat(config): gestión de cámaras y tiendas desde la UI (persistencia + cifrado)**

*Feature `stores-cameras-management` (CP0-CP5).* La configuración operativa (cámaras, ajustes de
tienda, tiendas del gateway) pasa a **Postgres como fuente de verdad**; `.env`/`stores.yaml` solo
siembran (seed idempotente). CRUD de cámaras **en caliente** (`POST/PUT/DELETE /cameras` +
`camera_manager.add/remove/update` con lock), ajustes de tienda (`GET/PUT /store/config`), alta de
tiendas en el gateway (`/api/admin/stores` + re-suscripción del fan-out) y **cifrado en reposo**
(Fernet) de credenciales RTSP/api_key con enmascarado en API/logs (`app/crypto.py`). UI: pestaña
"Cámaras y tienda" (Streamlit) + vista `#/admin` (SPA). *Checklist:* 80 tests verdes ·
credenciales nunca en claro · degradación segura si cambia la clave.

**Pull Request 5 — feat(detection): ocultamiento de cualquier objeto + fix de detección de personas**

*Feature `concealment-any-object` (CP0-CP5).* Modo configurable `patterns.concealment.mode:
whitelist | any` para detectar el ocultamiento de **cualquier** objeto (tienda que "vende de
todo"). **Fix raíz:** las personas solo las veía el modelo custom (intermitente) → ahora también
se usa `person` del modelo base yolov8n; y los patrones D/E miden **fracción de contención** en vez
de IoU (antes casi no disparaban con objetos pequeños). *Checklist:* validado en vivo (dispara
Patrón D con `remote`/`cell phone`) · 86 tests verdes · diagnóstico opt-in `SHOPGUARD_DEBUG_DETECT`.

**Pull Request 6 — feat(cameras): detección de cámaras USB conectadas**

`GET /cameras/detect` sondea los índices USB y reporta `disponible`/`en_uso`/`no_detectada`; botón
"🔍 Detectar cámaras" en el dashboard para descubrirlas sin adivinar el índice. *Checklist:* salta
los índices en uso para no pelear por el device · verificado con 2 cámaras.

> **Nota de proceso:** cada PR incluye título claro, descripción (qué cambia, por qué, impacto) y
> checklist, y referencia su feature/artefactos CP. Los mensajes de commit siguen *Conventional
> Commits* (`feat(...)`, `fix(...)`, `docs(...)`, `chore(...)`).
