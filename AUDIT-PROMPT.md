# Auditoría de errores — OpenWA + consumidores

> Prompt de diagnóstico. Pegar entero en una sesión fresca (o usar `@AUDIT-PROMPT.md`).
> Última actualización: 2026-09-13 (post-deploy v0.23.4).

## Objetivo

Revisar de punta a punta dónde hay errores, y para cada hallazgo dar evidencia
y en QUÉ CAPA está la falla (infra / OpenWA / entrega / worker / config).
No propongas fixes hasta tener la atribución.

## Estado conocido 2026-09-13 — NO rehacer

- OpenWA **v0.23.4** desplegado y verificado. Pi en `e83c7633`, container `healthy`,
  `restarts=0`, 4/4 sesiones `ready` sin re-link, 32 migraciones, 0 fallos de entrega.
- **OpenWA está sano**: recibe y despacha webhooks OK (`state=dispatched`,
  `attempts=0`, 0 `webhook_delivery_failures` ⇒ el receptor devuelve 2xx).
- **Estado de los consumidores (2026-09-13, POST-deploy y verificado):**
  - `laguiawa` — **ARREGLADO Y PROBADO.** Tenía `CLAUDE_MODEL = "claude-sonnet-4-6"` (modelo
    inválido) en `wrangler.toml` **y** en el fallback `DEFAULT_MODEL` de `src/claude.ts`.
    Corregido a `claude-sonnet-5`, deployado, y verificado end-to-end: mensaje real →
    webhook 16:01:40Z → respuesta 16:01:56Z (`status=read`).
  - `matchwa` — **NUNCA ESTUVO ROTO.** El modelo ya estaba bien (commit `3eb8412`) y responde
    normalmente. Ver la advertencia de método más abajo.
  - `aureo-bot` — **PROBADO Y FUNCIONANDO.** Modelo correcto (`a502d83`); verificado end-to-end el
    2026-09-13 (mensaje real → webhook `16:06:28Z` → respuesta `16:06:39Z`, `delivered`, **12s**).
    ⭐ **Sus 158 envíos `status=failed` iban TODOS a `120363163437200334@newsletter`** — a un canal
    de WhatsApp **no se le puede responder**, así que fallan por diseño, no por bug. El silencio de
    15 días era **falta de tráfico 1-a-1** (solo le llegaban noticias de canales), no una falla.
    ✅ **Filtro implementado** (commit `d05d4a9`, `apps/bot/src/filters.ts`): `isNewsletterJid` +
    chequeo en `reasonToSkip`. Ojo con el detalle: la guarda `non-user` **no** los atrapaba porque
    `normalizeNumber()` deja los dígitos y el id de un canal es numérico — hizo falta el sufijo
    explícito, igual que con `@g.us`. ⚠️ **Pendiente: deployar** (el código está pusheado, no en
    producción).
  - `control-de-gastos` — modelo correcto y hardcodeado (`worker.js:7500`); no requiere cambios.

- ⚠️ **CORRECCIÓN DE UN FALSO POSITIVO (no repetir):** el 2026-09-13 se reportó "los CF workers
  reciben pero no responden" contando *inbound sin responder*. **Era un artefacto de método.**
  Al filtrar por remitente: los "23 inbound / 0 outbound" de matchwa eran **22 mensajes de grupos
  `@g.us`** + 1 de canal, y los "296 sin responder" de laguia eran **302 de 303 mensajes de grupos**.
  Ningún bot responde en grupos ni canales, así que **0 respuestas era el comportamiento correcto**.
  El único bug real era el modelo inválido de laguiawa.

## Accesos y gotchas de entorno

- Pi: `ssh -o BatchMode=yes <usuario>@<ip-pi>` (interfaz cableada; la Wi-Fi es inestable).
- **⚠️ OpenWA corre en docker ROOTLESS.** Poner en TODO comando docker:
  `export DOCKER_HOST=unix:///run/user/1000/docker.sock`
  Sin eso `docker ps` sale vacío → **falso "se borró"**. Usá `docker -c rootless` como alternativa.
- **⚠️ El volumen pertenece a root (uid 100996, modo 0600).** Para leerlo/tar: `sudo -n`.
- DBs en el volumen `openwa_openwa-data`:
  `main.sqlite` (auth, **api_keys**) y `openwa.sqlite` (data: sessions, webhooks, messages, **migrations**).

## Capas a auditar (aguas arriba → abajo)

1. **Infra Pi**: `docker inspect openwa-api` (health, RestartCount), `docker logs openwa-api`,
   disco (`df -h`), y buscá `"level":"error"` en los logs.
2. **Sesiones** (`sessions`): `status`, `phone`, `connectedAt`.
   ⚠️ `lastActiveAt` se COMPARTE entre sesiones = heartbeat, **NO** es actividad real.
3. **Webhooks** (`webhooks`): `active`, `events`, `length(secret)`, `filters`, `lastTriggeredAt`.
4. **Entrega** (`webhook_outbox_events` + `webhook_delivery_failures`): `state`, `attempts`, `createdAt`.
   `dispatched` = entregado inline o encolado; un fallo real va a `webhook_delivery_failures`.
   Revisá también filas `pending` viejas (entregas trabadas).
5. **Mensajes** (`messages`): agrupá por `sessionId` + `direction`.
   ⚠️ Ordená y filtrá por **`createdAt` o `rowid`**, NUNCA por `timestamp`
   (esa es la de WhatsApp, en SEGUNDOS: envenena cualquier filtro por fecha).
6. **Migraciones**: tabla `migrations` de **`openwa.sqlite`**.
   ⚠️ NO están en `main.sqlite` — consultarlo ahí da un falso "no such table/MISSING".
7. **API keys** (`main.sqlite`.`api_keys`.`lastUsedAt`): muestra qué worker realmente llamó.
8. **Config**: `docker exec openwa-api env` (si te preocupan secretos, sacá solo los NOMBRES);
   y `docker-compose.override.yml` en la Pi (`ENGINE_TYPE=baileys`, `RESOLVE_LID_TO_PHONE`,
   `TRUSTED_PROXIES`, `AUTO_START_SESSIONS`). Ese archivo es **untracked**: no lo pises.
9. **Consumidores** (uno por uno: ¿deployado? ¿logs? ¿token wrangler válido? ¿modelo Claude vigente?):
   - `matchwa` → `<worker-matchwa>.workers.dev` — 4 webhooks, sesión `matchmgt-*`
   - `laguiawa` → `<worker-laguiawa>.workers.dev` — sesión `laguia`
   - `aureo-bot` → `<worker-aureo-bot>.workers.dev` — sesión `aureo`
   - `control-de-gastos` → **INDIRECTO**: repo `GitHub/control-gastos_nueva/`. NO tiene webhook
     ni sesión propia; manda WhatsApp vía `MATCHWA_URL` + `MATCHWA_SEND_SECRET`.
     ⇒ su estado de WhatsApp es el de matchwa.

## Método de diagnóstico (OBLIGATORIO antes de concluir "el bot no responde")

**Regla 1 — Filtrá SIEMPRE por tipo de remitente antes de contar nada.**
Un "inbound sin responder" **no** significa que el bot falló. Contar todo junto da
falsos positivos enormes (pasó el 2026-09-13: se reportaron "296 sin responder"
que eran 302 mensajes de grupos). Discriminá por el campo `"from"`:

| Sufijo / tipo | Qué es | ¿Se espera respuesta? |
|---|---|---|
| `@g.us` | Grupo | **No** (los bots son 1-a-1) |
| `@newsletter` / `@broadcast` | Canal / difusión | **No** |
| `@c.us` o teléfono, `type=text` | **Contacto 1-a-1 real** | **SÍ** ← el único que cuenta |
| `type` = image/video/audio y `body` vacío | Media sin texto | Depende del bot |

Recién con el subconjunto "1-a-1 + `type=text`" podés decir si un bot está fallando.

**Regla 2 — Compará una ventana ANTERIOR al cambio.** Si el patrón ya existía antes,
no es regresión de ese cambio. Pero aplicá la Regla 1 **dentro** de la ventana.

**Regla 3 — Ojo con el ruido de los tests propios.** Los mensajes que manda el usuario
desde su teléfono aparecen como **`outbound`** en la sesión de su WhatsApp personal
(`matchmgt-*`, la sesión personal) y como **`inbound`** en la sesión del bot. Un par
"outbound en una sesión + inbound en otra, al mismo segundo" suele ser un test humano,
no tráfico de producción.

**Regla 4 — La prueba definitiva es un mensaje real 1-a-1.** Si no hay tráfico de ese
tipo, no podés verificar nada: decilo y pedí autorización para mandar uno (nunca a un
tercero; sí al número del propio bot desde el teléfono del usuario).

## Inferir si un deploy se ejecutó (sin credenciales de Cloudflare)

El mtime de `.wrangler/tmp` en el repo del worker da la hora aproximada del último
`wrangler deploy`. Sirve para saber si el código nuevo llegó a producción **antes o
después** de un mensaje dado — que es la diferencia entre "el fix no funcionó" y
"ese mensaje era anterior al fix".

## Reglas

- **Verificá por EFECTO**, no por exit code ni por lo que dice un script:
  el `nohup` por SSH cuelga y deja logs de 0 bytes, y `pgrep -f deploy-clean` se auto-matchea.
- **No mandes mensajes de WhatsApp reales** a contactos sin autorización explícita. Si hace falta, pedilo.
- Un worker que devuelve 404 en GET **no** prueba que esté vivo: un subdominio
  inexistente también da 404. No uses eso como evidencia de liveness.
- Separá **artefacto** de **falla real**. En Windows son artefacto: chmod (`0o700`/`0o600`),
  separadores de path, `ps`/kill, y **CRLF** (un test que hace `indexOf('\n  servicio:\n')`
  falla con CRLF aunque el contenido sea correcto).
- No borres, reinicies ni deployes nada sin confirmar. No toques `RASPBERRY.md`
  (credenciales). Nunca "commit/push all" en `GitHub/` (~32 repos, binarios y basura).

## Salida esperada

Tabla: `capa | qué falla | evidencia (comando + resultado) | ¿es regresión? | severidad`.
Al final, aparte: **qué NO pudiste determinar y por qué** (p. ej. requiere un mensaje real).
