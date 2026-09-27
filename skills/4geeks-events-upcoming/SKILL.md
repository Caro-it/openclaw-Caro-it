---
name: "4geeks-events-upcoming"
description: "Lista próximos eventos 4Geeks: online de cualquier academia + presenciales solo de Madrid (id 6). Hora en España. Solo lectura."
---

# 4geeks-events-upcoming

Responde "¿qué eventos vienen?", "¿hay algún workshop de 4Geeks próximo?", "avisame de charlas", "no quiero perderme nada del campus". Solo lectura. No entrega ni actualiza nada.

## Cuándo usarla

Cuando Caro pida ver eventos/workshops/charlas próximos de 4Geeks. No cubre tareas ni proyectos (eso son las Skills 1-4).

## Cómo está construida

Toda la comunicación pasa por el wrapper del sistema `4geeks-get <ruta>` (igual que las otras skills de 4Geeks). No usa `curl` directo, no toca variables de entorno de credenciales.

**Regla de trabajo en memoria (igual que Skill 4):** las respuestas de la API se procesan solo en memoria durante la ejecución. Nunca se escriben en disco, nunca se guardan en el workspace ni en `memory/`, ni siquiera como archivo temporal de trabajo.

## Reglas que respeta (AGENTS.md 10a)

- Solo GET (lo impone el wrapper).
- Solo datos visibles para la cuenta de estudiante de Caro (la API ya filtra qué eventos puede ver).
- Ninguna credencial se muestra, copia o guarda.
- Nada se persiste en disco.
- Si el wrapper responde 401/403: parar y avisar a Caro, sin reintentar.
- Si falla dos veces seguidas (regla 9 AGENTS.md): parar y contar qué pasó.

## Endpoint

```
GET /v1/events/all?upcoming=true&limit=100
```

Sin filtro `academy` en la llamada a la API (el filtro por academia se aplica después, en memoria — ver siguiente sección). Paginar con `limit`/`offset` si `count` supera el `limit` pedido (mismo patrón que Skill 4).

## Filtro de relevancia (confirmado por Caro el 2026-09-27, no es un supuesto)

La API centraliza los eventos públicos en una academia global (`academy.id=47`, "4Geeks.com"), no en la de Caro, así que la llamada trae todos los `upcoming=true` sin filtrar por `academy`. Pero traer *todos* los presenciales sin más metería sedes donde Caro no puede estar físicamente. Por eso, sobre el resultado ya traído, se aplica este filtro en memoria antes de mostrar:

- **Si `online_event == true`:** se incluye siempre, sin importar `academy.id`.
- **Si `online_event == false` (presencial):** se incluye **solo si** `academy.id == 6` (4Geeks Madrid, el `academy_id` real de Caro, confirmado en `cohort.academy.id` de `/v1/admissions/user/me` — nunca hardcodear este número sin volver a confirmarlo si cambia el contexto de Caro).

Eventos presenciales de otras academias se descartan silenciosamente (no se listan ni se mencionan como "descartados"; simplemente no son relevantes para Caro).

Si en el futuro esto trae demasiado ruido de eventos online irrelevantes en volumen, avisar a Caro para reconsiderar — no cambiar el criterio por cuenta propia.

## Deduplicación

Deduplicar por `id` del evento (no debería haber duplicados en este endpoint, pero por las dudas ante paginación).

## Conversión de hora

`starting_at` y `ending_at` vienen en UTC (formato ISO con `Z`). Convertir siempre a hora de España (Europe/Madrid, con DST correcto: CEST/CET según la fecha) para mostrarlas. Nunca mostrar la hora en UTC sin convertir.

## Qué mostrar por evento

- **Título** (`title`)
- **Fecha y hora en España**, rango si hay `ending_at` (ej. "miércoles 7/10 · 19:00 a 20:15 España")
- **Tipo** (`event_type.name`, si viene)
- **Modalidad**: "online" si `online_event=true`, o el nombre del `venue`/ciudad de la academia si es presencial
- **Enlace**: `url` si no es null; si es null, decir explícitamente "sin enlace en la API — probablemente hay que anotarse desde la plataforma del campus" (nunca inventar una URL)

## Si no hay eventos próximos

Después de aplicar el filtro de relevancia, si la lista queda vacía: "No hay eventos próximos de 4Geeks por ahora." No inventar eventos ni mostrar eventos pasados o descartados por el filtro como si fueran próximos.

## Manejo de errores

Última línea `HTTP_STATUS:<código>` de cada llamada al wrapper (igual que las otras skills):
- **200** → seguir con el procesamiento normal.
- **401/403** → parar, avisar a Caro con el código exacto, no reintentar.
- **no_token** → avisar que falta configurar la credencial por SSH; no pedirla por chat.
- **curl_error** → error de conexión, reportar sin reintentar en bucle.
- Cualquier otro código → reportar código y cuerpo de respuesta, sin reintentar en bucle.

## Notas

- Esta skill NO entrega ni actualiza nada: es de solo lectura.
- El filtro de relevancia (online: todas; presencial: solo Madrid) es una decisión explícita de Caro (2026-09-27); si algún día hay que ajustarlo, debe ser otra decisión explícita suya, no un cambio silencioso.
- Depende de `/usr/local/bin/4geeks-get`, igual que las otras skills de 4Geeks. Si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearlo.

## Ejemplo real (2026-09-27, para ilustrar formato, no hardcodear)

```
Próximos eventos de 4Geeks:

📅 El mapa real del AI Engineer
Miércoles 7/10 · 19:00 a 20:15 España
Tipo: Tendencias de IA (Público)
Modalidad: online
Enlace: sin enlace en la API — probablemente hay que anotarse desde la plataforma del campus.
```

(Este evento es `academy.id=47`, pero como es online se incluye igual, por el criterio confirmado arriba.)
