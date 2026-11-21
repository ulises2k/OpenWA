# Apps que usan OpenWA

Lista de las aplicaciones que consumen esta instancia de OpenWA (self-hosted, LAN privada), con el estado de sus tests unitarios. Generado el 2026-11-21 tras el upgrade a v0.31.0.

**⭐ Antes del PRÓXIMO upgrade de OpenWA, correr la sección [Checklist para el próximo upgrade](#checklist-para-el-próximo-upgrade) completa, en los 4 repos.**

## Resumen

| App | Worker (Cloudflare) | Sesión OpenWA | Tests | Estado |
|---|---|---|---|---|
| [matchwa](#matchwa) | `matchwa` | `matchmgt-*` (multi-tenant) | 434 pass / 11 skip (447) | ✅ 100%, sin warnings |
| [laguiawa](#laguiawa) | `laguiawa` | `laguia` | 183 pass / 6 skip (189) | ✅ 100%, sin warnings |
| [aureo-bot](#aureo-bot) | `aureo-bot` | `aureo` | 114 pass (114) | ✅ 100%, sin warnings |
| [matchmgt-api-worker](#matchmgt-api-worker) | *(no consume OpenWA directo — repo hermano de `matchwa`)* | — | 357 pass (357) | ✅ 100%, sin warnings, 0 vulnerabilidades |

---

## matchwa

**Qué es:** asistente de citas que responde por el WhatsApp personal del usuario (vía OpenWA), impersonándolo, solo con sus contactos activos de MatchMGT. Cloudflare Worker.

- **Repo:** `GitHub/matchmgtapp/matchwa`
- **Worker:** `matchwa` → `<worker-matchwa>.workers.dev`
- **Sesiones OpenWA:** `matchmgt-google-106653513899703590968`, `matchmgt-1d08cb99-4e07-4413-871e-70fdb31df0af` (las otras 2 `matchmgt-google-*` están `disconnected`, nunca se vincularon)
- **Endpoints de OpenWA que usa:** `send-text`, `send-document`, `GET contacts/:id/phone`
- **Filtro de mensajes:** descarta grupos/canales, documentos sin caption se tratan como conversación, notas de voz se transcriben
- **Depende también de:** `matchmgt-api-worker` (repo hermano — API de contactos/usuarios de MatchMGT; `matchwa` lee su `worker.js` directo del filesystem local para un contract test, ver abajo)

**Tests:** `npm test` (Vitest) → **434 pass / 11 skip** de 447. `npm run typecheck` (`tsc --noEmit`) limpio. `npm run build` (`wrangler deploy --dry-run`) construye y lista bindings sin errores. ✅ Sin warnings.

### Fallo que hubo y cómo se arregló (2026-11-21)

`src/matchmgt-contract.test.ts` fallaba: es un *contract test* que **lee el `worker.js` del repo hermano `matchmgt-api-worker` directamente del filesystem** (`readFileSync`, ruta `../../matchmgt-api-worker/worker.js` relativa al checkout) y verifica que siga conteniendo, literalmente, cada motivo de rechazo (`DENY_REASON` en `src/matchmgt.ts`) que `matchwa` sabe traducir. El test decía que el motivo `no-plan` ya no existía en ese `worker.js`.

**Causa real: mi checkout local de `matchmgt-api-worker` estaba desactualizado 4 commits respecto a `origin/main`** (último commit local `fe0f782` del 16/10; `origin/main` tenía hasta `cc0bbeb` del 26/10). El commit `c15ceec` ("feat: trial Pro de 7 dias y bloqueo de acceso al vencer") agregó `denyContext(request, 'no-plan')` al worker — ese commit sí estaba en `origin/main`, solo que mi carpeta local nunca lo había bajado.

⚠️ **Primer intento de fix, INSUFICIENTE (dejó el repo en un estado peor):** actualizar *solo* `worker.js` con `git checkout origin/main -- worker.js`, sin tocar `package.json`/tests. Esto arregló el contract test de `matchwa` pero dejó `matchmgt-api-worker` con código nuevo y **tests viejos** → al correr sus propios tests, pasó de 11 fallos (preexistentes, por el mismo motivo `no-plan`) a **198 fallos** (la mitad de su suite), porque el `worker.js` nuevo requiere fixtures/mocks de features (trial, Polar, etc.) que los tests viejos no conocían.

**Fix real aplicado:** hacer el `pull` completo del repo (no archivo por archivo):
1. `git stash push -u` para preservar sin perder nada los cambios locales sueltos que bloqueaban el fast-forward (bumps de devDependencies de un `ncu` viejo + 2 archivos de test de un pull interrumpido anterior, con mtime de 9 días antes — quedaron en `stash@{0}`, sin aplicar, a la espera de que el usuario decida si los quiere o los descarta).
2. `git checkout HEAD -- worker.js package.json package-lock.json` para descartar el parche parcial.
3. `git merge --ff-only origin/main` — trajo los 4 commits completos (18 archivos: `worker.js`, sus 7 archivos de test, `package.json`/`lock`, `eslint.config.mjs`, docs).
4. `npm install` (el lockfile nuevo trae `eslint-plugin-import-x`, que no estaba instalado).
5. `npm audit fix` — 4 vulnerabilidades moderadas de `minimatch` (transitiva de `eslint`/`eslint-plugin-import-x`, solo devDependencies de lint, no tocan el Worker en producción) → 0 vulnerabilidades.

Resultado: `matchmgt-api-worker` **357/357 tests, 0 warnings, 0 vulnerabilidades**; `matchwa` sigue en 434/434 contra el `worker.js` real y completo.

**Lección para el futuro — la más importante de este documento:** un contract test que lee otro repo del filesystem **no alcanza con actualizar el único archivo que el test mira**. Si el repo hermano tiene varios commits pendientes, hay que traerlos *todos* (código + sus propios tests + sus propias deps) con un `pull` normal, nunca un `checkout` selectivo de un solo archivo — eso deja al repo hermano con código y tests de commits distintos, lo cual rompe su propia suite aunque arregle la de quien lo consume. Antes de tocar nada: `git -C <repo-hermano> fetch && git -C <repo-hermano> log HEAD..origin/main --stat` para ver el alcance completo de lo pendiente.

### Warning que hubo y cómo se arregló

`npm test` mostraba: *"Your Vite config uses features that are unsupported by `configLoader: 'native'`... ESM syntax in a file loaded as CommonJS (vitest.config.ts)"*. Causa: `package.json` no tenía `"type": "module"`, a diferencia de `laguiawa` y `aureo-bot` (que sí lo tienen) — Vite cargaba el `vitest.config.ts` (que usa `import`/`export default`) en modo CommonJS. **Fix:** se agregó `"type": "module"` a `package.json`. No afecta el deploy (un Cloudflare Worker siempre corre en ESM nativo en producción, el campo solo ajusta cómo Node interpreta los archivos en local para Vite/Vitest/tsc). Verificado: test, typecheck y `wrangler deploy --dry-run` siguen limpios tras el cambio.

---

## laguiawa

**Qué es:** asistente de WhatsApp de La Guía de la Zona (FAQ de planes, zonas y contacto) vía OpenWA. Cloudflare Worker.

- **Repo:** `GitHub/LaGuiaDeLaZona/laguiawa`
- **Worker:** `laguiawa` → `<worker-laguiawa>.workers.dev`
- **Sesión OpenWA:** `laguia`
- **Endpoints de OpenWA que usa:** `send-text`, `send-audio`
- **Filtro de mensajes:** bot de FAQ público, responde a cualquiera en 1-a-1; descarta grupos y tarjetas de comercio (`product`/`order`); no filtra por tipo de texto (a propósito, para tolerar variaciones entre motores/versiones de OpenWA)

**Tests:** `npm test` (Vitest) → **183 pass / 6 skip** de 189. ✅ Sin fallos, sin warnings. Nada que arreglar.

---

## aureo-bot

**Qué es:** bot de WhatsApp de Áureo (mega_proyecto) vía OpenWA. Cloudflare Worker.

- **Repo:** `GitHub/Aureo_proyecto/mega_proyecto/apps/bot`
- **Worker:** `aureo-bot` → `<worker-aureo-bot>.workers.dev`
- **Sesión OpenWA:** `aureo`
- **Endpoints de OpenWA que usa:** `send-text`, `GET contacts/:id/phone`, `DELETE /api/sessions/:id`
- **Filtro de mensajes:** lista blanca estricta (solo `type === "chat"` o `"text"` pasa); descarta grupos y canales (`@newsletter`) explícitamente

**Tests:** `npm test` (Vitest) → **114 pass** de 114. ✅ 100%, sin warnings. Nada que arreglar.

---

## matchmgt-api-worker

**Qué es:** API de contactos/usuarios/pagos de MatchMGT. **No consume OpenWA directamente** (no tiene sesión ni webhook propio) — es el repo hermano del que `matchwa` lee el código fuente para su contract test, y por eso entra en este checklist.

- **Repo:** `GitHub/matchmgtapp/matchmgt-api-worker`
- **Rol:** backend de MatchMGT (auth, contactos, pagos/suscripciones, `GET /api/wa/context` que `matchwa` consulta por cada mensaje entrante)

**Tests:** `npm test` (Vitest) → **357 pass** de 357. ✅ 100%, sin warnings, **0 vulnerabilidades** (`npm audit`).

**Estado al 2026-11-21:** el checkout local estaba 4 commits atrás de `origin/main` (ver detalle completo en la sección de `matchwa` arriba). Se actualizó con `git pull --ff-only` tras guardar en `stash@{0}` unos cambios locales sueltos (bumps de devDependencies + 2 tests de un pull anterior interrumpido) que **no se descartaron** — quedan en el stash a la espera de que el usuario decida si los recupera (`git stash show -p stash@{0}`) o los descarta (`git stash drop`).

⚠️ **Pendiente de decisión del usuario:** hay un `stash@{0}` con contenido de un intento de actualización anterior (archivos con mtime de 9 días antes del fix). No se tocó porque no estaba claro si era trabajo en curso. Revisar y resolver antes del próximo upgrade.

---

## Checklist para el próximo upgrade

Repetir esto después de cada upgrade de OpenWA, en los 4 repos, **antes** de dar el upgrade por cerrado:

1. **Actualizar el checkout local de `matchmgt-api-worker` con un PULL COMPLETO, nunca archivo por archivo:**
   ```
   git -C GitHub/matchmgtapp/matchmgt-api-worker fetch origin
   git -C GitHub/matchmgtapp/matchmgt-api-worker log HEAD..origin/main --stat
   ```
   Si hay commits pendientes:
   - Sin cambios locales sueltos → `git -C GitHub/matchmgtapp/matchmgt-api-worker pull --ff-only`.
   - Con cambios locales sueltos que bloquean el fast-forward → **no actualizar un solo archivo con `checkout`** (deja el repo con código y tests de versiones distintas, que es peor que el problema original). En su lugar: `git stash push -u` (preserva todo sin perder nada) → `git checkout HEAD -- <archivos modificados>` → `git merge --ff-only origin/main` → decidir después si recuperar el stash.
   - Tras el pull: `npm install` (el lockfile puede traer paquetes nuevos) y `npm audit fix` si hay vulnerabilidades.
2. **Correr `npm test` en los 4 repos** (`matchwa`, `laguiawa`, `aureo-bot`, `matchmgt-api-worker`) y confirmar **0 fallos y 0 warnings** en la salida. Un warning de `configLoader: 'native'`/ESM-en-CJS indica que a `package.json` le falta `"type": "module"`.
3. **Correr `npm run typecheck`** (si existe el script) en cada repo.
4. **Correr `npm run build`** (equivale a `wrangler deploy --dry-run`) en cada repo — construye y lista bindings sin subir nada, sin necesitar credenciales de Cloudflare.
5. **Correr `npm audit`** en cada repo y resolver con `npm audit fix` lo que no implique un cambio de versión mayor sin revisar antes.
6. Si algún contract test cross-repo falla con "esto ya no existe en el otro lado", **el primer paso siempre es el punto 1** (pull completo del repo hermano) — nunca parchear un solo archivo a mano.
7. Repetir el test de firma HMAC de los webhooks (`laguiawa`, `aureo-bot`) descrito abajo, para confirmar que no hay drift de `WEBHOOK_SECRET` tras el redeploy.

---

## Verificación post-upgrade OpenWA v0.31.0 (2026-11-21)

- Las 4 apps/repos pasan sus tests unitarios: **434 + 183 + 114 + 357 = 1088 tests, 0 fallos, 0 warnings, 0 vulnerabilidades**.
- Se probó la firma HMAC de los webhooks (`X-OpenWA-Signature`) de `laguiawa` y `aureo-bot` contra sus workers reales: el secreto correcto devuelve `200`, un secreto inventado (control) devuelve `401` — confirma que no hay drift de `WEBHOOK_SECRET` entre OpenWA y los workers.
- Las 4 sesiones autenticadas (`laguia`, `aureo`, `matchmgt-google-106653513899703590968`, `matchmgt-1d08cb99`) volvieron a `ready` sin re-escanear QR tras el deploy.
- 0 fallos nuevos en `webhook_delivery_failures`.
- ⚠️ Queda pendiente: `stash@{0}` sin resolver en `matchmgt-api-worker` (ver sección arriba).

Ver `AUDIT-PROMPT.md` y `WEBHOOK-SECRET-DRIFT.md` en este repo para el playbook de diagnóstico si alguna de estas apps deja de responder.
