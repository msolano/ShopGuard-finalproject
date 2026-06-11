> Detalla en esta sección los prompts principales utilizados durante la creación del proyecto, que justifiquen el uso de asistentes de código en todas las fases del ciclo de vida del desarrollo. Esperamos un máximo de 3 por sección, principalmente los de creación inicial o los de corrección o adición de funcionalidades que consideres más relevantes.
> Puedes añadir adicionalmente la conversación completa como link o archivo adjunto si así lo consideras


## Índice

1. [Descripción general del producto](#1-descripción-general-del-producto)
2. [Arquitectura del sistema](#2-arquitectura-del-sistema)
3. [Modelo de datos](#3-modelo-de-datos)
4. [Especificación de la API](#4-especificación-de-la-api)
5. [Historias de usuario](#5-historias-de-usuario)
6. [Tickets de trabajo](#6-tickets-de-trabajo)
7. [Pull requests](#7-pull-requests)

> Los prompts siguen una estructura experta: **ROL · CONTEXTO · OBJETIVO · RESTRICCIONES ·
> ENTREGABLES · MÉTODO**. El contenido resultante de cada sección está documentado en
> `readme.md`.

---

## 1. Descripción general del producto

**Prompt 1: Definición inicial del producto (Product Owner)**
```
# ROL
Actúa simultáneamente como (a) Product Owner senior especializado en retail y prevención
de pérdidas (loss prevention) y (b) consultor técnico de productos de visión por computador.

# CONTEXTO
Quiero crear "ShopGuard", un sistema de videovigilancia inteligente para tiendas pequeñas y
medianas que NO pueden permitirse personal dedicado a vigilar pantallas de CCTV. Hoy sus
cámaras solo sirven para revisar el robo después de que ocurrió. Quiero convertir esas
cámaras pasivas en detección proactiva en tiempo real.

# OBJETIVO
Definir la visión de producto y delimitar el alcance del MVP.

# RESTRICCIONES (no negociables)
- Inferencia 100% en local, sin servicios de pago en la nube (privacidad + coste operativo).
- Stack Python.
- Hardware objetivo: PC con CPU i5/i7; GPU NVIDIA opcional.
- Cámaras comunes: USB y/o IP por RTSP.
- Notificación al responsable por un canal que ya use (p. ej. Telegram) con evidencia visual.
- El diseño debe poder crecer a multi-cámara y, más adelante, a multi-tienda.

# ENTREGABLES (en este orden)
1. Problema que resuelve y propuesta de valor (1 párrafo).
2. Alcance del MVP: dentro y fuera de alcance (lista explícita).
3. Stack tecnológico recomendado, justificando cada elección con su trade-off.
4. Catálogo inicial de patrones de comportamiento a detectar, con su nivel de riesgo.
5. Limitaciones y riesgos conocidos del enfoque.

# MÉTODO
Antes de responder, hazme las preguntas que necesites para acotar el problema. Razona las
decisiones técnicas y señala explícitamente los compromisos (precisión vs. rendimiento,
falsos positivos vs. cobertura). No te limites a listar tecnologías: justifícalas.
```

**Prompt 2: Catálogo de patrones y niveles de riesgo**
```
# ROL
Eres ingeniero de visión por computador especializado en detección de comportamiento
(behavior analytics) sobre vídeo en tiempo real.

# CONTEXTO
Partimos del MVP de ShopGuard ya definido (FastAPI + YOLOv8 + MediaPipe Pose + OpenCV,
inferencia local). Necesito formalizar QUÉ comportamientos sospechosos detecta el sistema,
de forma que sea implementable y, sobre todo, ajustable en operación sin redeploys.

# OBJETIVO
Diseñar un catálogo de patrones de detección listo para implementar.

# REQUISITOS
Para CADA patrón especifica:
- Identificador y nombre.
- Descripción del comportamiento humano que representa.
- Nivel de riesgo (ALTO / MEDIO) y su criterio.
- Señales de CV que lo disparan (detección de objetos YOLO, pose MediaPipe, tracking
  multi-frame, IoU/solapamiento, temporizadores).
- Parámetros umbral configurables y su valor inicial sugerido.
- Falsos positivos esperados y cómo mitigarlos.

# ALCANCE
Cubre tanto lo detectable con el modelo base COCO como los casos que COCO NO conoce y
requerirían un modelo entrenado a medida (clase "sospechosa").

# RESTRICCIÓN DE DISEÑO
Todos los umbrales, cooldowns y flags de activación deben vivir en configuración externa
(YAML) y poder recargarse en caliente, sin tocar código ni reiniciar el proceso.
```

**Prompt 3: Evolución a producto multi-tienda (gateway central)**
```
# ROL
Actúa como arquitecto de producto/software con experiencia en sistemas distribuidos y en
estrategias de evolución de producto sin romper la base instalada.

# CONTEXTO
ShopGuard funciona hoy como una instalación autónoma por tienda (proceso de detección +
API + dashboard local). Ya hay tiendas en operación que NO usan autenticación.

# OBJETIVO
Definir la evolución a un producto para CADENAS de tiendas, manteniendo retrocompatibilidad
total con las instalaciones existentes.

# REQUISITOS DEL PRODUCTO MULTI-TIENDA
- Un panel central (gateway) que agregue N tiendas detrás de una sola URL.
- Login propio del supervisor de cadena.
- Vista de estadísticas globales agregadas de toda la cadena.
- Alertas en vivo de TODAS las tiendas en un único flujo en tiempo real.
- Cada instalación de tienda sigue siendo autónoma (si el gateway cae, la tienda funciona).

# RESTRICCIÓN CRÍTICA
Las tiendas existentes sin autenticación NO deben romperse. La seguridad debe poder
adoptarse de forma incremental, tienda por tienda (migración rolling).

# ENTREGABLES
1. Propuesta de valor de la versión multi-tienda.
2. Modelo de convivencia tiendas-legacy ↔ gateway, y la estrategia de migración rolling.
3. Cómo se identifica el origen de cada alerta sin imponer cambios de esquema a las tiendas.
Razona los trade-offs de cada decisión.
```

---

## 2. Arquitectura del Sistema

### 2.1. Diagrama de arquitectura

**Prompt 1: Diseño de la arquitectura base**
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

**Prompt 2: Diagrama del gateway multi-tienda**
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

**Prompt 3: Estrategia de configuración en caliente**
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

### 2.2. Descripción de componentes principales

**Prompt 1: Aislamiento de estado por cámara**
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

**Prompt 2: Motor de patrones con estado y cooldowns**
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

**Prompt 3: Caché anti-stampede del gateway**
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

### 2.3. Descripción de alto nivel del proyecto y estructura de ficheros

**Prompt 1: Estructura inicial del repositorio**
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

**Prompt 2: Dónde colocar el gateway sin acoplar**
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

**Prompt 3: Externalizar rutas y home del proyecto**
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

### 2.4. Infraestructura y despliegue

**Prompt 1: Scripts de arranque multiplataforma**
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

**Prompt 2: PostgreSQL local con Docker**
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

**Prompt 3: Migración de SQLite a PostgreSQL**
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

### 2.5. Seguridad

**Prompt 1: Autenticación opt-in sin romper lo existente**
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

**Prompt 2: Streaming MJPEG autenticado en el navegador**
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

**Prompt 3: Sesión del gateway con cookie firmada**
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

### 2.6. Tests

**Prompt 1: Estrategia de pruebas**
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

**Prompt 2: Tests de las heurísticas de patrones**
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

**Prompt 3: Tests de integración de la API**
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

---

## 3. Modelo de Datos

**Prompt 1: Diseño del esquema de alertas**
```
# ROL
Actúa como data engineer especializado en modelado relacional con SQLAlchemy 2.0 y en
portabilidad entre motores SQL.

# CONTEXTO
ShopGuard necesita registrar cada detección sospechosa (alerta). NO hay usuarios, roles ni
inventario en BD: solo eventos. La evidencia es una imagen JPG que se guarda en disco
(evidence/), no como blob en la base de datos.

# OBJETIVO
Diseñar el modelo de datos de persistencia (la entidad alerta).

# REQUISITOS DEL ESQUEMA
Campos: nivel de riesgo (ALTO/MEDIO), patrón que la disparó (A/B/C/D/custom), descripción
legible, cámara de origen, ruta de la imagen de evidencia (nullable), timestamp y flag de
resuelta.

# RESTRICCIONES
- SQLAlchemy 2.0 (DeclarativeBase). Una sola clase ORM, mínima.
- El MISMO modelo debe funcionar en SQLite (piloto) y PostgreSQL (producción) cambiando solo
  DATABASE_URL.
- timestamp en UTC, con default e indexado para consultas por fecha.
- Tipos, longitudes, nullability y defaults explícitos.

# ENTREGABLE
La clase ORM Alert con sus columnas/constraints y la función init_db() para crear el esquema.
Justifica brevemente por qué una sola tabla es suficiente para el MVP.
```

**Prompt 2: Consultas, filtros y estadísticas**
```
# ROL
Eres ingeniero backend especializado en capas de acceso a datos (DAL) y consultas SQL
portables.

# CONTEXTO
El dashboard local y el gateway necesitan consultar y agregar las alertas de la tabla alerts.
La misma capa debe funcionar en SQLite y PostgreSQL sin SQL específico de motor.

# OBJETIVO
Implementar las funciones de consulta y agregación sobre la tabla alerts.

# REQUISITOS
- Listar con paginación (limit/offset) y filtros: level, resolved, rango de fechas.
- Obtener una alerta por id.
- Marcar resuelta: individual y en bulk (por lista de ids O por filtros level+fechas).
- Estadísticas: total de hoy, total por nivel, total sin resolver, y conteo por hora del día.

# RESTRICCIÓN DE PORTABILIDAD
El conteo por hora debe usar una expresión que funcione igual en SQLite y PostgreSQL
(p. ej. extract('hour', ...)); evita funciones propietarias de un solo motor.

# ENTREGABLE
Las funciones get_alerts, get_alert, mark_resolved, bulk_mark_resolved y get_stats, con
manejo de sesión y cierre seguro.
```

**Prompt 3: store_id sin migración de esquema**
```
# ROL
Actúa como arquitecto de datos que decide cuándo evolucionar un esquema y cuándo evitarlo
(YAGNI vs. deuda técnica).

# CONTEXTO
En la versión multi-tienda, cada tienda tiene su PROPIA base de datos con su tabla alerts.
El gateway agrega varias tiendas. Quiero poder saber de qué tienda viene cada alerta, pero
añadir una columna store_id a la tabla de cada tienda obligaría a migrar N bases de datos
para un dato que, dentro de la tienda, es constante.

# OBJETIVO
Decidir dónde y cómo se materializa la identidad de la tienda (store_id) en cada alerta.

# PREGUNTAS A RESOLVER
- ¿Puede el gateway etiquetar las alertas con su store_id en tiempo de agregación, sin tocar
  el esquema de las tiendas? ¿En REST y en WebSocket?
- ¿Qué trade-offs tiene enriquecer en el payload vs. persistir la columna?
- ¿Bajo qué condición futura SÍ valdría la pena migrar el esquema y añadir store_id como
  columna (y exponer un filtro por tienda en /alerts)?

# ENTREGABLE
Recomendación argumentada con su disparador de decisión (cuándo cambiar de estrategia).
```

---

## 4. Especificación de la API

**Prompt 1: Diseño de la API REST de tienda**
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

**Prompt 2: WebSocket de alertas en tiempo real**
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

**Prompt 3: API de agregación del gateway**
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

---

## 5. Historias de Usuario

**Prompt 1: Redacción de historias de usuario priorizadas**
```
# ROL
Actúa como Product Owner senior experto en redacción de historias de usuario y en
priorización de backlog (impacto vs. esfuerzo).

# CONTEXTO
Ya está definida la descripción de ShopGuard (detección de robos en tiempo real + alertas
Telegram + dashboard local + gateway multi-tienda). Necesito convertir esa visión en
historias de usuario accionables para el MVP.

# OBJETIVO
Redactar las 3 historias de usuario MÁS importantes.

# REQUISITOS
- Formato estándar: "Como [rol] quiero [objetivo] para [beneficio]".
- Identifica los roles REALES del sistema (operador de tienda y supervisor de cadena); no
  inventes roles que el producto no tiene.
- Prioriza por impacto de negocio: (1) detección/notificación de comportamiento sospechoso,
  (2) vigilancia en vivo y gestión de alertas, (3) supervisión centralizada multi-tienda.
- Mantén el alcance del MVP: nada de funcionalidades nice-to-have.

# ENTREGABLE
3 historias de usuario, cada una con su rol, objetivo y beneficio, ordenadas por prioridad.
```

**Prompt 2: Criterios de aceptación verificables**
```
# ROL
Eres Product Owner/QA con experiencia en escribir criterios de aceptación testeables (estilo
Gherkin / Given-When-Then) que un equipo de QA pueda automatizar.

# CONTEXTO
Tengo 3 historias de usuario de ShopGuard (alerta en tiempo real, monitoreo en vivo,
supervisión multi-tienda). Necesito que cada una tenga criterios objetivos, no ambiguos.

# OBJETIVO
Escribir los criterios de aceptación de cada una de las 3 historias.

# REQUISITOS — los criterios deben ser MEDIBLES e incluir
- Tiempos concretos (p. ej. notificación Telegram en < 5 s).
- Estados de cámara observables (ok / degraded / disconnected) y reconexión con backoff.
- Comportamiento de cooldown anti-spam por patrón.
- Persistencia en BD + push por WebSocket de cada alerta.
- Autorefresh del dashboard (3 s) sin recargar la página.
- Compatibilidad: las tiendas legacy SIN auth siguen funcionando.

# RESTRICCIÓN
Cada criterio debe poder traducirse directamente a un caso de test (unitario, integración o
E2E). Evita criterios subjetivos ("debe ser rápido/intuitivo").

# ENTREGABLE
Lista de criterios de aceptación por historia, redactados de forma verificable.
```

**Prompt 3: Mapeo de historias a componentes**
```
# ROL
Actúa como tech lead que practica trazabilidad requisito→implementación (requirements
traceability) antes de planificar el trabajo.

# CONTEXTO
Tengo 3 historias de usuario de ShopGuard y una arquitectura ya definida (backend app/,
dashboard Streamlit, gateway/). Antes de crear los tickets, quiero saber qué toca cada
historia.

# OBJETIVO
Mapear cada historia de usuario a los componentes y contratos que la realizan.

# REQUISITOS
Para cada historia, indica:
- Módulos del backend implicados (p. ej. patterns.py, alerts.py, database.py).
- Componentes del dashboard local y/o del gateway.
- Endpoints REST y/o eventos WebSocket que consume o expone.
- Dependencias entre historias (qué debe existir antes).

# ENTREGABLE
Una matriz historia ↔ componentes ↔ endpoints/eventos, que sirva de base para desglosar los
tickets de trabajo de la sección siguiente.
```

---

## 6. Tickets de Trabajo

**Prompt 1: Ticket del motor de patrones**
```
# ROL
Actúa como Tech Lead que escribe tickets de ingeniería listos para que un desarrollador los
implemente sin ambigüedad (descripción, criterios, DoD, estimación).

# CONTEXTO
Parto de la historia HU-01 "alerta de comportamiento sospechoso en tiempo real". El corazón
técnico es el motor de patrones que evalúa cada frame.

# OBJETIVO
Redactar el ticket técnico del motor de patrones (PatternDetector).

# CONTENIDO REQUERIDO DEL TICKET
- Descripción técnica: heurísticas A (ocultamiento), B (permanencia), C (postura con
  MediaPipe), D (objeto en bbox de persona) y el patrón custom del modelo entrenado.
- Estado temporal por cámara y cooldowns por patrón configurables en rules.yaml.
- Gate anti falso-positivo del patrón C (confiar en la pose solo si YOLO ve persona con
  confianza ≥ umbral).
- Recarga de umbrales en caliente sin perder estado.
- Criterios de aceptación verificables.
- Definición de hecho (Definition of Done), incluyendo tests unitarios.
- Estimación en story points.

# ENTREGABLE
El ticket completo con id, tipo, estimación, descripción, criterios y DoD.
```

**Prompt 2: Ticket de API + persistencia**
```
# ROL
Eres Tech Lead de backend que escribe tickets de API con foco en contratos estables y
testabilidad.

# CONTEXTO
La API de alertas es consumida por el dashboard y el gateway. Necesito un ticket que cubra el
ciclo de vida completo de las alertas sobre la tabla alerts, más el push en tiempo real.

# OBJETIVO
Redactar el ticket de la API de alertas + persistencia + WebSocket.

# CONTENIDO REQUERIDO DEL TICKET
- Endpoints: listar con filtros (level, resolved, fechas) y paginación; resolver individual;
  resolver en bulk (por ids o filtros); servir evidencia JPG; estadísticas; WS /ws/alerts.
- Detalles técnicos: orden literal-antes-que-paramétrica (/alerts/bulk/resolve antes de
  /alerts/{id}/resolve) y fallback de evidencia por basename tras migración.
- Criterios de aceptación verificables, incluido: con AUTH_REQUIRED=true, request sin
  X-API-Key → 401.
- Definición de hecho: tests de integración con TestClient + SQLite en memoria; OpenAPI en
  /docs.
- Estimación en story points.

# ENTREGABLE
El ticket completo (id, tipo, estimación, descripción, criterios, DoD).
```

**Prompt 3: Ticket de la SPA del gateway**
```
# ROL
Actúa como Tech Lead de frontend que escribe tickets respetando restricciones técnicas
estrictas del proyecto.

# CONTEXTO
El gateway sirve una SPA al supervisor de cadena. RESTRICCIÓN DURA del proyecto: 100% vanilla
JS/CSS, SIN npm ni librerías externas; los gráficos se hacen con SVG inline (sin Chart.js ni
similares).

# OBJETIVO
Redactar el ticket de la SPA del gateway central.

# CONTENIDO REQUERIDO DEL TICKET
- Login con cookie firmada; rutas protegidas redirigen a /login sin sesión.
- Vista de tiendas con estado online/offline.
- Estadísticas globales agregadas (/api/stats/global).
- Alertas en vivo vía WebSocket /api/ws/alerts multiplexado, etiquetadas con store_id.
- Stream MJPEG embebido (modo redirect por defecto).
- Criterios de aceptación verificables.
- Definición de hecho.
- Estimación en story points.

# RESTRICCIÓN A REPETIR EN EL TICKET
Sin dependencias npm; gráficos SVG inline. Cualquier librería externa es motivo de rechazo
en review.

# ENTREGABLE
El ticket completo (id, tipo, estimación, descripción, criterios, DoD).
```

---

## 7. Pull Requests

**Prompt 1: Plantilla y primer PR (auth opt-in)**
```
# ROL
Actúa como ingeniero DevOps/plataforma que define el estándar de Pull Requests del equipo y
escribe descripciones de PR ejemplares.

# CONTEXTO
ShopGuard no tenía proceso formal de PR. Quiero establecer una plantilla y estrenarla con el
primer cambio relevante: la autenticación opt-in de la API de tienda.

# OBJETIVO
1) Definir una plantilla de PR reutilizable. 2) Aplicarla al PR de auth opt-in.

# REQUISITOS DE LA PLANTILLA
- Resumen del cambio, lista de cambios principales por archivo, y checklist de calidad.

# PUNTOS A DESTACAR EN EL PR DE AUTH
- No rompe instalaciones legacy (AUTH_REQUIRED=false por defecto).
- Comparaciones de credenciales timing-safe (hmac.compare_digest).
- Fail-fast si AUTH_REQUIRED=true sin API key.
- /health y / quedan EXENTOS para que el gateway detecte tiendas offline.

# ENTREGABLE
La plantilla de PR + el PR-1 redactado con ella (resumen, cambios, checklist).
```

**Prompt 2: PR del gateway multi-tienda**
```
# ROL
Eres ingeniero que documenta un PR grande (feature completa) de forma que sea revisable por
partes.

# CONTEXTO
El gateway central es una funcionalidad nueva y autocontenida (paquete gateway/). Voy a
abrirla como un PR y necesito que el revisor entienda cada pieza rápido.

# OBJETIVO
Redactar el PR del gateway central multi-tienda.

# CONTENIDO REQUERIDO
- Resumen del valor que aporta (agregación de N tiendas con SPA, login, stats globales,
  alertas en vivo).
- Lista de archivos nuevos con una línea por cada uno: main, ws_fanout, cache, auth, proxy,
  stores, static, scripts/hash_password, stores.yaml.
- Checklist específico del proyecto:
  · Sin dependencias npm (SPA 100% vanilla, gráficos SVG inline).
  · Arranque fail-fast (validate_or_die) si faltan PASS_HASH o SECRET.
  · Modos MJPEG redirect/proxy seleccionables.
  · Compatibilidad: tiendas legacy sin auth siguen funcionando.

# ENTREGABLE
El PR-2 redactado con la plantilla (resumen, cambios principales, checklist).
```

**Prompt 3: PR de soporte PostgreSQL**
```
# ROL
Eres ingeniero que documenta un PR de infraestructura/datos con foco en compatibilidad hacia
atrás y reversibilidad.

# CONTEXTO
Añado soporte PostgreSQL a ShopGuard para instalaciones grandes, pero las pequeñas deben
seguir en SQLite sin cambios. El cambio de motor debe ser solo configuración.

# OBJETIVO
Redactar el PR que añade soporte PostgreSQL manteniendo SQLite como default.

# CONTENIDO REQUERIDO
- Resumen: por qué (concurrencia/volumen) y qué se mantiene (SQLite por defecto).
- Cambios principales:
  · database.py: normalización de DATABASE_URL al driver psycopg, pool_pre_ping,
    connect_args solo para SQLite, estadísticas portables (extract hour).
  · docker-compose.yml: servicio postgres:16-alpine con healthcheck y volumen persistente.
  · scripts/migrate_sqlite_to_postgres.py: migración de la tabla alerts.
  · .env.postgres.example y docs/postgres-local.md.
- Checklist que garantice:
  · SQLite sigue siendo el default.
  · Cambiar de motor es SOLO definir DATABASE_URL (sin tocar código).
  · Migración de datos sin pérdida; mismas consultas en ambos motores.

# ENTREGABLE
El PR-3 redactado con la plantilla (resumen, cambios principales, checklist).
```
