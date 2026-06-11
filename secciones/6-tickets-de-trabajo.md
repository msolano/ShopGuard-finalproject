## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 6. Tickets de Trabajo

Tres tickets representativos: uno de **detección/backend**, uno de **base de datos** y uno de
**frontend/gateway**.

### SG-001 — Motor de patrones de detección (A–D + custom) con cooldowns

**Tipo:** Backend / Visión por computador · **Estimación:** 13 pts

**Descripción.** Implementar `PatternDetector` que, a partir de las detecciones de YOLO y la
pose de MediaPipe por frame, evalúe los patrones A (ocultamiento), B (permanencia), C
(postura sospechosa) y D (objeto dentro del bbox de persona), además de las clases del
modelo custom. Debe mantener estado temporal por cámara y aplicar cooldowns por patrón.

**Criterios de aceptación:**
- Cada patrón se activa/desactiva con su flag `enabled` en `rules.yaml`.
- Respeta el `cooldown_seconds` por patrón (no emite duplicados del mismo evento).
- El patrón C solo confía en la pose si YOLO ve una persona con confianza ≥
  `require_person_min_confidence` (anti falso-positivo de MediaPipe en escenas vacías).
- `reload_from_disk()` aplica nuevos umbrales sin perder el estado acumulado.

**Definición de hecho:** tests unitarios de A y D con detecciones sintéticas; cooldowns
verificados; sin regresiones al recargar reglas.

### SG-002 — Modelo de datos de alertas + capa de acceso y estadísticas

**Tipo:** Base de datos / Persistencia · **Estimación:** 8 pts

**Descripción.** Diseñar la entidad `alerts` (SQLAlchemy 2.0) y la capa de acceso a datos:
listar con filtros y paginación, resolver (individual y en bulk), servir la evidencia JPG y
calcular estadísticas, funcionando indistintamente sobre SQLite y PostgreSQL.

**Criterios de aceptación:**
- El mismo ORM funciona en SQLite y PostgreSQL cambiando solo `DATABASE_URL`.
- `get_alerts` soporta `level`, `resolved`, `date_from/to`, `limit`, `offset`.
- `bulk_mark_resolved` acepta `alert_ids` o filtros (`level`, fechas).
- `get_stats` calcula totales (hoy, por nivel, sin resolver) y conteo por hora con una
  expresión portable (`extract('hour', ...)`).
- `timestamp` en UTC; evidencia referenciada por ruta con fallback por basename.

**Definición de hecho:** tests de integración con SQLite en memoria; script de migración
SQLite→PostgreSQL operativo.

### SG-003 — SPA del gateway central multi-tienda

**Tipo:** Frontend / Gateway · **Estimación:** 13 pts

**Descripción.** Construir la SPA vanilla del gateway (sin dependencias npm) que muestre la
lista de tiendas, sus estadísticas globales y las alertas en vivo multiplexadas, tras un
login con cookie firmada. Gráficos en SVG inline.

**Criterios de aceptación:**
- Login funcional contra `gateway/auth.py`; rutas protegidas redirigen a `/login` sin sesión.
- Vista de tiendas con estado online/offline y `/api/stats/global` agregado.
- WebSocket `/api/ws/alerts` pinta alertas en vivo etiquetadas con `store_id`.
- Stream MJPEG embebido (modo redirect por defecto).
- 100% vanilla JS/CSS, sin librerías externas; gráficos SVG inline.

**Definición de hecho:** SPA navegable end-to-end con al menos 2 tiendas; sin dependencias
npm; tiendas legacy sin auth siguen visibles.

---

**Prompt 1: Ticket del motor de patrones (backend)**
*Contexto: Convertir la HU de detección en una tarea técnica cerrada.*
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
**Respuesta clave del asistente:**
> "Ticket SG-001 (13 pts): `PatternDetector` con A–D + custom, estado por cámara, cooldowns
> vía `rules.yaml`, gate `require_person_min_confidence` para el patrón C, `reload_from_disk()`
> sin pérdida de estado, y tests unitarios con detecciones sintéticas."

**Impacto en el proyecto:**
Estructuró el desarrollo de `app/patterns.py` y sus umbrales en `rules.yaml`.

**Prompt 2: Ticket del modelo de datos (base de datos)**
*Contexto: Detallar el esquema y la capa de acceso de alertas.*
```
# ROL
Eres Tech Lead de backend que escribe tickets de datos con foco en portabilidad entre
motores y testabilidad.

# CONTEXTO
La capa de datos de alertas es consumida por el dashboard y el gateway. Necesito un ticket
que cubra el esquema y la capa de acceso, funcionando en SQLite y PostgreSQL.

# OBJETIVO
Redactar el ticket del modelo de datos + capa de acceso (DAL) de alertas.

# CONTENIDO REQUERIDO DEL TICKET
- Entidad alerts (SQLAlchemy 2.0) con sus campos/constraints; evidencia como ruta, no blob.
- Funciones: listar con filtros y paginación; obtener por id; resolver individual y en bulk;
  estadísticas (hoy, por nivel, sin resolver, por hora con expresión portable).
- Restricción: mismo ORM en SQLite y PostgreSQL cambiando solo DATABASE_URL.
- Criterios de aceptación verificables.
- Definición de hecho: tests de integración con SQLite en memoria + script de migración
  SQLite→PostgreSQL.
- Estimación en story points.

# ENTREGABLE
El ticket completo (id, tipo, estimación, descripción, criterios, DoD).
```
**Respuesta clave del asistente:**
> "Ticket SG-002 (8 pts): entidad `alerts` + DAL (`get_alerts`, `bulk_mark_resolved`,
> `get_stats`) portable SQLite/PostgreSQL; `extract('hour', ...)` para el conteo por hora;
> tests con `sqlite:///:memory:` y `scripts/migrate_sqlite_to_postgres.py`."

**Impacto en el proyecto:**
Definió el alcance de `app/database.py` y la migración entre motores.

**Prompt 3: Ticket de la SPA del gateway (frontend)**
*Contexto: Acotar el frontend del panel central sin dependencias externas.*
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
**Respuesta clave del asistente:**
> "Ticket SG-003 (13 pts): SPA en `gateway/static/` (index/login/app.js/style.css), login
> contra `auth.py`, consumo de `/api/stores` y `/api/stats/global`, WS `/api/ws/alerts` con
> `store_id`, MJPEG redirect. Sin dependencias npm; gráficos SVG inline."

**Impacto en el proyecto:**
Acotó la SPA vanilla del gateway y reforzó la restricción "sin dependencias npm".
