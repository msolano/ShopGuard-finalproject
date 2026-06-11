## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 5. Historias de Usuario

Se priorizan 3 historias que cubren el núcleo de valor: **recibir alertas accionables**,
**vigilar en vivo** y **supervisar la cadena**.

### HU-01 — Alerta de comportamiento sospechoso en tiempo real

> **Como** operador de tienda
> **quiero** recibir una notificación inmediata por Telegram con la foto del momento cuando
> el sistema detecta un comportamiento sospechoso
> **para** poder reaccionar antes de que el robo se complete.

**Criterios de aceptación:**
- Cuando un patrón de nivel `ALTO` se dispara, llega un mensaje de Telegram en < 5 s con la
  foto del frame, el patrón, la cámara y la hora.
- La alerta se persiste en la BD y se empuja por WebSocket al dashboard.
- Cada patrón respeta un *cooldown* para no enviar mensajes repetidos del mismo evento.
- Las alertas `MEDIO` no envían Telegram por defecto (`telegram_only_high: true`).

### HU-02 — Monitoreo de cámaras en vivo desde el dashboard

> **Como** operador de tienda
> **quiero** ver el vídeo en directo de todas las cámaras y el listado de alertas recientes
> en un único panel
> **para** vigilar la tienda y revisar/resolver incidentes sin tocar la consola.

**Criterios de aceptación:**
- El dashboard muestra el stream MJPEG de cada cámara con su estado de salud (`ok`,
  `degraded`, `disconnected`).
- Lista las alertas recientes con su evidencia y permite marcarlas como resueltas
  (individual y en bloque).
- Se autorrefresca cada 3 s sin recargar la página completa.
- Si una cámara se cae, se reconecta automáticamente (backoff exponencial) y el panel lo
  refleja.

### HU-03 — Supervisión centralizada de varias tiendas

> **Como** supervisor de cadena
> **quiero** un panel único con login que agregue todas las tiendas, sus estadísticas
> globales y las alertas en vivo de todas
> **para** monitorizar la operación sin abrir el dashboard de cada sede.

**Criterios de aceptación:**
- Tras login (cookie firmada), el supervisor ve la lista de tiendas con su estado online/offline.
- `/api/stats/global` muestra los totales agregados de toda la cadena.
- Un WebSocket único entrega las alertas de todas las tiendas, cada una etiquetada con su
  `store_id`.
- Las tiendas sin autenticación (legacy) siguen funcionando con su dashboard local sin cambios.

---

**Prompt 1: Redacción de historias de usuario priorizadas**
*Contexto: Traducir la visión del producto a HU accionables y acotadas al MVP.*
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
**Respuesta clave del asistente:**
> "3 HU: (HU-01) alerta en tiempo real por Telegram con foto para el operador; (HU-02)
> dashboard de monitoreo en vivo con resolución de alertas; (HU-03) panel central
> multi-tienda con login para el supervisor. Cubren el 80% del valor con 2 roles."

**Impacto en el proyecto:**
Fijó el alcance funcional del MVP y los dos roles (operador / supervisor).

**Prompt 2: Criterios de aceptación verificables**
*Contexto: Convertir cada HU en criterios objetivos y testeables.*
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
**Respuesta clave del asistente:**
> "Criterios medibles por HU: latencia de Telegram, persistencia + WebSocket, cooldown por
> patrón, autorefresh de 3 s, reconexión con backoff, login con cookie y etiquetado de
> `store_id`, además de no romper instalaciones sin auth."

**Impacto en el proyecto:**
Generó los criterios de aceptación que alimentan la suite de pruebas y los tickets.

**Prompt 3: Mapeo de historias a componentes**
*Contexto: Conectar cada HU con los módulos que la implementan.*
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
**Respuesta clave del asistente:**
> "HU-01 → `patterns.py` + `alerts.py` + `/ws/alerts` + Telegram. HU-02 → `streamlit_app.py`
> + `/cameras`, `/video/stream`, `/alerts`. HU-03 → `gateway/` (`ws_fanout.py`, `cache.py`,
> `/api/stats/global`, SPA)."

**Impacto en el proyecto:**
Trazó la relación HU↔componentes que estructuró los tickets de la sección 6.
