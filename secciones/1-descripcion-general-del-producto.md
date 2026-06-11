## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 1. Descripción general del producto

### Descripción general

**ShopGuard** es un sistema de videovigilancia inteligente que detecta comportamientos
sospechosos de robo en tiempo real dentro de comercios. Procesa el vídeo de una o varias
cámaras (USB o IP/RTSP), aplica modelos de visión por computador (YOLOv8 + MediaPipe Pose)
sobre cada frame y, cuando reconoce un patrón de riesgo, genera una alerta y notifica al
responsable por **Telegram** adjuntando la foto del frame sospechoso.

### Problema que resuelve

Las tiendas pequeñas y medianas no pueden permitirse personal dedicado a vigilar pantallas
de CCTV de forma continua; la mayoría de cámaras solo sirven para revisar el robo *después*
de que ocurrió. ShopGuard convierte cámaras pasivas en un sistema de detección proactiva y
de bajo coste, sin hardware especializado.

### Propuesta de valor

- Detección **en tiempo real** sobre hardware común (CPU i5/i7; GPU opcional).
- **Alertas accionables** por Telegram con evidencia fotográfica del incidente.
- **Configurable sin tocar código**: umbrales y patrones en `app/rules.yaml`, recargables
  en caliente vía API.
- **Multicámara** con reconexión automática (backoff exponencial) y panel local por tienda.
- **Escalable a cadena de tiendas** mediante un *gateway* central que agrega N instancias
  detrás de una sola URL con su propia SPA y autenticación.

### Funcionalidades principales

**Detección de patrones de riesgo:**

| Patrón | Nivel | Descripción |
|--------|-------|-------------|
| A | ALTO | Objeto visible → la mano se acerca → el objeto desaparece |
| B | MEDIO | Persona quieta más de N segundos en un área |
| C | MEDIO/ALTO | Postura sospechosa (agachado, espalda a cámara, manos al torso) |
| D | ALTO | Objeto entra al *bounding box* de la persona y no reaparece |
| Custom | ALTO | Clase sospechosa detectada por un modelo entrenado a medida |

**Otras funcionalidades:**

- Streaming de vídeo en vivo (MJPEG) por cámara, embebible en el dashboard.
- Notificaciones por Telegram con foto del frame del incidente.
- Dashboard local por tienda (Streamlit) con vídeo, alertas y estadísticas.
- Gestión multicámara con estado de salud (`ok`, `degraded`, `disconnected`).
- Ajuste de umbrales de detección en caliente vía `rules.yaml` + `POST /config/reload`.
- Gateway central multi-tienda con login, estadísticas globales y alertas en vivo
  multiplexadas de todas las tiendas.
- Autenticación opt-in (API key, token MJPEG firmado HMAC, auth de WebSocket) sin romper
  instalaciones legacy.

### Usuarios objetivo

- **Operador de tienda:** recibe alertas por Telegram y revisa el dashboard local.
- **Supervisor de cadena:** monitoriza todas las tiendas desde el gateway central.

### Stack tecnológico

FastAPI (API + streaming MJPEG) · Streamlit (dashboard local) · SQLite/PostgreSQL vía
SQLAlchemy · YOLOv8 (ultralytics) + MediaPipe Pose + OpenCV · Telegram
(`python-telegram-bot`) · Gateway en FastAPI + httpx + WebSockets con SPA vanilla JS.

---

### Prompts utilizados en esta fase

**Prompt 1: Definición inicial del producto (Product Owner)**
*Contexto: Primer prompt que estableció la visión, el alcance del MVP y el stack del proyecto.*
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
**Respuesta clave del asistente:**
> "ShopGuard será un sistema de detección proactiva sobre hardware común. Para el MVP
> propongo: FastAPI para la API y el streaming MJPEG, YOLOv8 (ultralytics) para detección
> de objetos/personas, MediaPipe Pose para análisis de postura, OpenCV para captura, y
> notificaciones por Telegram. La detección corre 100% en local. Empezaría por una sola
> cámara y un catálogo inicial de patrones de riesgo."

**Impacto en el proyecto:**
Estableció la base conceptual, el stack definitivo (FastAPI + YOLOv8 + MediaPipe + Telegram)
y el enfoque local-first que guió todo el desarrollo posterior.

---

**Prompt 2: Catálogo de patrones y niveles de riesgo**
*Contexto: Concreción de las funcionalidades core — formalizar qué comportamientos detecta el sistema.*
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
**Respuesta clave del asistente:**
> "Catálogo propuesto: Patrón A (objeto visible → mano se acerca → objeto desaparece, ALTO),
> Patrón B (persona quieta > N segundos, MEDIO), Patrón C (postura sospechosa: agachado,
> espalda a cámara, manos al torso, MEDIO/ALTO), Patrón D (objeto entra al bounding box de
> la persona y no reaparece, ALTO) y un Patrón Custom para clases detectadas por un modelo
> entrenado. Los umbrales irían en un `rules.yaml` recargable en caliente."

**Impacto en el proyecto:**
Definió el corazón funcional del producto (los 5 patrones A–D + Custom) y la decisión de
externalizar umbrales a `app/rules.yaml` recargable vía API sin reiniciar.

---

**Prompt 3: Evolución a producto multi-tienda (gateway central)**
*Contexto: Decisión estratégica de escalar de una tienda a una cadena sin romper instalaciones existentes.*
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
**Respuesta clave del asistente:**
> "Propongo un gateway en FastAPI que agrega las tiendas vía httpx/WebSockets y sirve una
> SPA propia. La autenticación de tienda se introduce como opt-in (`AUTH_REQUIRED=false`
> por defecto) para no romper instalaciones legacy, y la migración es rolling: tienda por
> tienda. El gateway enriquece cada alerta con `store_id` y multiplexa los WebSocket de
> todas las tiendas en uno solo."

**Impacto en el proyecto:**
Originó la arquitectura del `gateway/` (agregador multi-tienda con SPA, cookie firmada y
fan-out de WebSockets) y el principio de **auth opt-in** que preserva la compatibilidad
con las tiendas ya desplegadas.
