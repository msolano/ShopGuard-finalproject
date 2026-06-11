## Índice

1. [Descripción general del producto](1-descripcion-general-del-producto.md)
2. [Arquitectura del sistema](2-arquitectura-del-sistema.md)
3. [Modelo de datos](3-modelo-de-datos.md)
4. [Especificación de la API](4-especificaciones-de-la-api.md)
5. [Historias de usuario](5-historias-de-usuario.md)
6. [Tickets de trabajo](6-tickets-de-trabajo.md)
7. [Pull requests](7-pull-requests.md)

---

## 7. Pull Requests

> Nota: estos PRs documentan los tres bloques de cambio más representativos del desarrollo,
> redactados con la plantilla que se usaría al abrirlos.

### PR-1 — feat: autenticación opt-in (API key + token MJPEG + WS)

**Resumen.** Introduce seguridad opcional en la API de tienda sin romper instalaciones
legacy. `AUTH_REQUIRED=false` por defecto.

**Cambios principales:**
- `app/auth.py`: `require_api_key` (X-API-Key, timing-safe), `mint/verify_mjpeg_token`
  (HMAC corto, TTL 60 s) y `ws_authenticate` (`?api_key=`).
- `app/config.py`: `API_KEY`, `AUTH_REQUIRED`, `MJPEG_TOKEN_SECRET`, `MJPEG_TOKEN_TTL` +
  fail-fast si `AUTH_REQUIRED=true` sin key.
- `app/main.py`: `Depends(require_api_key)` en REST, `require_mjpeg_token` en MJPEG,
  endpoint `GET /cameras/{id}/stream/token`, auth en `/ws/alerts`.

**Checklist:** ✅ compatibilidad legacy · ✅ comparaciones timing-safe · ✅ `/health` y `/`
exentos · ✅ dashboard inyecta la key automáticamente.

### PR-2 — feat: gateway central multi-tienda con SPA

**Resumen.** Añade el paquete `gateway/` que agrega N tiendas detrás de una sola URL, con
login, estadísticas globales y alertas en vivo multiplexadas.

**Cambios principales:**
- `gateway/main.py`: SPA + `/api/stores*`, `/api/stats/global` (`asyncio.gather`),
  `/api/ws/alerts` (multiplexado), login con cookie firmada.
- `gateway/ws_fanout.py`: 1 task WS por tienda con reconexión exponencial + `store_id`.
- `gateway/cache.py`: caché TTL=3 s con `asyncio.Lock` por key (anti-stampede).
- `gateway/auth.py` + `scripts/hash_password.py`: sesión pbkdf2_sha256 + itsdangerous.
- `gateway/static/`: SPA 100% vanilla (gráficos SVG inline, sin npm).
- `stores.yaml`: catálogo `{id,name,api_url,api_key}` con interpolación `${VAR}`.

**Checklist:** ✅ sin dependencias npm · ✅ arranque fail-fast (`validate_or_die`) · ✅
modos MJPEG redirect/proxy · ✅ tiendas legacy siguen funcionando.

### PR-3 — feat: soporte PostgreSQL + migración desde SQLite

**Resumen.** Permite usar PostgreSQL para instalaciones con más cámaras/volumen, manteniendo
SQLite como default de piloto.

**Cambios principales:**
- `app/database.py`: `_normalize_database_url` (driver `psycopg`), `pool_pre_ping`,
  `connect_args` solo para SQLite; estadísticas con `extract('hour', ...)` compatibles.
- `docker-compose.yml`: servicio `postgres:16-alpine` con healthcheck y volumen persistente.
- `scripts/migrate_sqlite_to_postgres.py`: copia de la tabla `alerts` entre motores.
- `.env.postgres.example` + `docs/postgres-local.md`.

**Checklist:** ✅ SQLite sigue siendo el default · ✅ cambio de motor solo con
`DATABASE_URL` · ✅ migración de datos sin pérdida · ✅ mismas consultas en ambos motores.

---

**Prompt 1: Plantilla y primer PR (auth opt-in)**
*Contexto: Documentar el bloque de seguridad como un PR revisable.*
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
**Respuesta clave del asistente:**
> "PR-1 'feat: auth opt-in' con cambios en `auth.py`, `config.py` y `main.py`; checklist con
> compatibilidad legacy, `compare_digest`, endpoints exentos y la inyección automática de la
> key en el dashboard."

**Impacto en el proyecto:**
Estableció la plantilla de PR y documentó el bloque de autenticación.

**Prompt 2: PR del gateway multi-tienda**
*Contexto: Empaquetar la capa de agregación como un PR autocontenido.*
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
**Respuesta clave del asistente:**
> "PR-2 'feat: gateway multi-tienda' con `gateway/*` y `stores.yaml`; checklist de no-npm,
> `validate_or_die`, modos MJPEG y convivencia con tiendas legacy."

**Impacto en el proyecto:**
Documentó la incorporación del gateway y reforzó las restricciones del proyecto.

**Prompt 3: PR de soporte PostgreSQL**
*Contexto: Documentar el salto de SQLite a Postgres con migración.*
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
**Respuesta clave del asistente:**
> "PR-3 'feat: soporte PostgreSQL' con cambios en `database.py`, `docker-compose.yml` y
> `scripts/migrate_sqlite_to_postgres.py`; checklist de SQLite-por-defecto, cambio solo por
> `DATABASE_URL` y migración sin pérdida de datos."

**Impacto en el proyecto:**
Documentó el soporte multi-motor de BD y la ruta de migración.
