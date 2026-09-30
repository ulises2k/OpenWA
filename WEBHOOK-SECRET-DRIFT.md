# Drift del secreto de webhook — bot mudo, OpenWA "sano"

> Caso real: 2026-09-25 → 2026-09-30. Un bot dejó de responder durante ~4 días.
> Documentado por si vuelve a pasar: el diagnóstico lleva 3 comandos, pero **no se ve
> mirando OpenWA**, que es lo que lo hace traicionero.

## Síntoma

- Un consumidor (bot) **no responde** a mensajes 1-a-1, mientras **otros sí responden**.
- El mensaje **llega**: está en `messages` con `direction=incoming`.
- OpenWA parece sano por todos los indicadores habituales: contenedor `healthy`,
  `restarts=0`, sesiones `ready`, `state=dispatched` en `webhook_outbox_events`.
- El webhook del bot tiene `lastTriggeredAt` **congelado** hace días (ver abajo: es la pista).

## Causa raíz

El **consumidor rotó su `WEBHOOK_SECRET`** (al deployar otra cosa) y **el registro del
webhook en OpenWA nunca se actualizó**. Desde ese momento:

```
OpenWA firma con el secreto VIEJO  →  el worker valida contra el NUEVO  →  401 en cada entrega
```

Es un **drift de configuración entre sistemas**, no un bug de código: `signature.ts` y el
registrador no habían cambiado en meses.

## Por qué es invisible en OpenWA

Tres señales engañosas que hay que saber leer:

| Señal | Qué parece | Qué significa en realidad |
|---|---|---|
| `webhook_outbox_events.state = dispatched` | "se entregó" | sólo que **se intentó** (inline o encolado) |
| `webhooks.lastTriggeredAt` viejo | "no hay tráfico" | **sólo se actualiza con 2xx** → congelado = todo falla |
| `/api/health` + sesiones `ready` | "OpenWA está sano" | OpenWA está sano: **el que rechaza es el consumidor** |

**La tabla que hay que mirar es `webhook_delivery_failures`** (`lastStatusCode`, `lastError`,
`attempts`). Un fallo real vive ahí, no en el outbox.

⚠️ `attempts=3` y `state=dispatched` **no** son contradictorios: el outbox registra el intento,
los 3 reintentos agotados van a `webhook_delivery_failures`. Un 401 no se reintenta solo después.

## Diagnóstico — el test decisivo (firma HMAC)

Prueba en 1 paso **qué secreto espera realmente el worker**, sin leer ningún secreto de
Cloudflare (los secrets son write-only: no se pueden leer de vuelta).

```bash
BODY='{"event":"ping"}'          # payload inocuo: el worker lo ignora, no manda mensajes
sig=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$CANDIDATO" | awk '{print $NF}')
curl -s -o /dev/null -w '%{http_code}\n' -X POST "https://<worker>/webhook" \
  -H "Content-Type: application/json" -H "X-OpenWA-Signature: sha256=$sig" -d "$BODY"
```

- **200** → ése es el secreto que el worker tiene deployado.
- **401** → no lo es.

Correr **siempre con un control** (un secreto inventado) para validar que el 401 significa
"firma inválida" y no otra cosa.

El secreto con el que OpenWA firma se lee de su DB (`webhooks.secret`, en `openwa.sqlite`).
Comparar los dos lados es lo que cierra el caso.

## Fix

Hacer coincidir los dos lados. **Actualizar OpenWA es preferible**: el worker y el panel ya
comparten el valor nuevo, y revertir el worker al viejo se rompería otra vez en el próximo
"Conectar" (ver Prevención).

```
PUT /api/sessions/<sessionId>/webhooks/<webhookId>
{ "url": "<url>", "events": ["message.received"], "secret": "<secreto-nuevo>" }
```

Es **el mismo PUT que hace `ensureWaWebhook`** del panel al tocar "Conectar". Verificar
**en la DB** que `secret` cambió (no alcanza con el `HTTP 200` del PUT).

## Prevención

- **Regla:** al rotar `WEBHOOK_SECRET` en el worker que *consume*, actualizar **también** el
  worker que *registra* (el panel), con **el mismo valor**.
- El secreto **sólo se re-registra solo** cuando alguien toca "Conectar". Si los dos workers
  quedan con secretos distintos, ese click **vuelve a romper** el bot.
- Un secreto rotado en un lado y no en el otro **no rompe nada visible** hasta el próximo
  mensaje real. Por eso: después de cada rotación, mandar **un mensaje 1-a-1 de prueba**.
- Los secretos de Cloudflare **no se pueden leer de vuelta**: si se pierde el valor, hay que
  leerlo de la DB de OpenWA (es el que firma) o **rotar los dos lados a un valor conocido**.

## Regla de oro para el próximo "el bot no responde"

1. ¿El mensaje está en `messages` como `incoming`? → **no es un problema de recepción**.
2. ¿Hay filas en `webhook_delivery_failures` para esa sesión? → **leer `lastStatusCode`**.
   - `401` → **este caso**: drift del secreto.
   - `404`/`5xx`/timeout → problema del worker (URL, deploy, crash).
3. ¿El mismo worker responde a **otra** sesión? Si sí → el problema es **por sesión/webhook**,
   no del worker.
