---
name: "4geeks-projects-status"
description: "Lista los proyectos asignados a Caro en 4Geeks/BreatheCode con su estado (pendiente, entregado, calificado). Solo lectura."
---

# 4geeks-projects-status

Lista los proyectos asignados a Caro en la API de BreatheCode (4Geeks) junto con su estado actual (pendiente, entregado, calificado). Solo lectura. Una sola responsabilidad: listar y clasificar el estado; no entrega, no actualiza, no toca nada.

## Cuándo usarla
Cuando Caro pida ver el estado de sus proyectos del bootcamp (4Geeks), o pregunte "¿cómo van mis proyectos?", "¿qué tengo calificado?", "¿cuántos proyectos aprobados tengo?". No cubre "¿qué me falta entregar?" ni listados de trabajo pendiente por hacer: eso corresponde a la Skill 3 (trabajo pendiente), para no solapar responsabilidades.

## Cómo está construida
Toda la comunicación pasa por el wrapper del sistema `4geeks-get <ruta>` (igual que la skill `4geeks-token-check`). No usa `curl` directo, no toca ninguna variable de entorno de credenciales, no escribe nada en disco.

## Reglas que respeta (AGENTS.md 10a)
- Solo GET (lo impone el wrapper).
- Solo datos de la propia cuenta de estudiante de Caro.
- Ninguna credencial se muestra, copia o guarda.
- Si el wrapper responde 401/403: parar y avisar a Caro, sin reintentar.
- Si falla dos veces seguidas (regla 9 AGENTS.md): parar y contar qué pasó.

## Endpoint

```
GET /v1/assignment/user/me/task?task_type=PROJECT&limit=100&offset=<N>
```

- `task_type=PROJECT` filtra para traer solo proyectos (no lecciones, ejercicios ni quizzes).
- `limit`/`offset` para paginar. Usar un `limit` alto (por ejemplo 100) para minimizar llamadas, pero **hay que recorrer todas las páginas**: mientras el campo `next` de la respuesta no sea `null`, seguir pidiendo con el siguiente `offset`. No asumir que la primera página trae todo (confirmado con datos reales: con `limit=5` hay `count:36` y `next` no nulo).
- No pasar `task_status` en la query: se trae todo y se clasifica localmente con la tabla de abajo, para no perder ningún proyecto por un filtro mal armado.

## Procedimiento

1. Ejecutar, incrementando `offset` mientras `next` no sea `null`:
   ```sh
   4geeks-get "/v1/assignment/user/me/task?task_type=PROJECT&limit=100&offset=0"
   ```
2. Por cada proyecto en `results`, clasificar el estado combinando `task_status` y `revision_status` (confirmado con datos reales de la cuenta de Caro el 2026-09-27):

   | task_status | revision_status | Estado a mostrar |
   |---|---|---|
   | PENDING | (cualquiera) | 🔴 Pendiente (no entregado) |
   | DONE | PENDING | 🟡 Entregado, esperando revisión |
   | DONE | APPROVED | 🟢 Calificado - Aprobado |
   | DONE | REJECTED | 🟠 Calificado - Rechazado (requiere corrección) |
   | (otra combinación no vista) | — | Mostrar los valores crudos de ambos campos y avisar que es una combinación nueva no mapeada |

3. Armar el reporte para Caro en dos partes:
   - **Recuento por estado primero**, en una sola línea, por ejemplo: `🔴 5 · 🟡 3 · 🟢 26 · 🟠 2`.
   - **Después, el listado agrupado por estado** (todos los 🔴 juntos, luego los 🟡, luego los 🟢, luego los 🟠), mostrando por cada proyecto: `title` y `cohort.name`. No mostrar `description` completa (puede tener feedback largo del instructor) salvo que Caro pida el detalle de un proyecto puntual.
4. Si Caro pide filtrar (por ejemplo "solo los aprobados" o "solo los de tal cohorte"), filtrar sobre los datos ya traídos, no volver a llamar a la API con un query distinto salvo que haga falta releer todo.
5. Manejo de errores en la última línea `HTTP_STATUS:<código>` (igual que en `4geeks-token-check`):
   - **200** → seguir con el procesamiento normal.
   - **401/403** → parar, avisar a Caro con el código exacto, no reintentar.
   - **no_token** → avisar que falta configurar la credencial por SSH; no pedirla por chat.
   - **curl_error** → error de conexión, reportar sin reintentar en bucle.
   - Cualquier otro código → reportar código y cuerpo de respuesta, sin reintentar en bucle.

## Notas
- Esta skill NO entrega proyectos (`POST .../deliver`) ni actualiza URLs (`PUT /v1/assignment/task/{id}`). Si Caro pide eso, es una skill nueva y distinta, con su propia revisión (esas son escrituras, no lectura).
- Esta skill NO pide el detalle completo de una tarea (`GET /v1/assignment/task/{task_id}`) salvo que se agregue como paso explícito más adelante; por ahora se queda con lo que trae el listado.
- Esta skill NO responde "¿qué me falta entregar?": esa pregunta es responsabilidad de la Skill 3 (trabajo pendiente), para evitar solapamiento.
- Depende de `/usr/local/bin/4geeks-get` igual que `4geeks-token-check`. Si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearlo.
- Si aparece una combinación de `task_status`/`revision_status` no vista en la tabla, no inventar una interpretación: mostrar los valores crudos y avisar a Caro para actualizar la tabla junto con ella.
