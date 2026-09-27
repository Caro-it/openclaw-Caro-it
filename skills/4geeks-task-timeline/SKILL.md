---
name: "4geeks-task-timeline"
description: "Días de calendario entre apertura y entrega de un trabajo 4Geeks + estimación del curso (probablemente horas). No es tiempo trabajado. Solo lectura."
---

# 4geeks-task-timeline

Responde "¿cuánto tiempo me llevó [proyecto/ejercicio]?" con los únicos dos datos temporales reales que expone la API para estudiantes: días de calendario entre apertura y entrega, y la estimación que el curso le puso al trabajo al diseñarlo. Ninguno de los dos es tiempo efectivamente trabajado, y la skill lo aclara siempre, en cada respuesta, sin excepción.

## Por qué existe (y qué NO hace)

Se investigó `/v1/activity/me` (con y sin `cohort`, con y sin `date_start`/`date_end`, con `academy=6` explícito) buscando tiempo real por tarea: siempre 403 para cuenta de estudiante ("Missing academy_id" sin el parámetro, o "no capability read_activity for academy 6" con él — es un endpoint de staff/instructor, no de estudiante). También se revisó el campo `assignment_telemetry` en `/v1/assignment/task/{id}`: viene `null` en toda tarea entregada probada. Rutas de telemetría inventadas (`/task/{id}/telemetry`, etc.) dan 404.

**Conclusión que esta skill respeta:** la API no permite saber cuánto tiempo trabajó Caro en algo puntual. Esta skill NUNCA debe calcular ni sugerir esa cifra a partir de fechas — eso sería inventar un dato que no existe. Lo que muestra es otra cosa, explícitamente etiquetada como otra cosa.

## Cuándo usarla

Cuando Caro pregunte "¿cuánto tiempo me llevó X?", "¿cuándo entregué X?", "¿qué estimación tiene X?" sobre un proyecto o ejercicio concreto de 4Geeks. No cubre listados generales (eso son las Skills 2 y 3).

## Cómo está construida

Toda la comunicación pasa por el wrapper `4geeks-get <ruta>` (nunca curl directo, nunca toca la variable de credencial directamente). Solo lectura. Todo se procesa en memoria durante la ejecución: nunca se escribe en disco, ni en el workspace, ni en `memory/`, ni siquiera como archivo temporal.

## Reglas que respeta (AGENTS.md 10a)

- Solo GET (lo impone el wrapper).
- Solo datos de la cuenta de estudiante de Caro.
- Ninguna credencial se muestra, copia o guarda.
- Nada se persiste en disco.
- Si el wrapper responde 401/403: parar y avisar a Caro con el código exacto, sin reintentar.
- Si falla dos veces seguidas (regla 9 AGENTS.md): parar y contar qué pasó.
- Nunca estimar, inventar ni derivar un "tiempo trabajado" a partir de fechas. Si el dato no existe, se dice así.

## Procedimiento

### 1. Traer las tareas candidatas

```
GET /v1/assignment/user/me/task?task_type=PROJECT&limit=100&offset=<N>
GET /v1/assignment/user/me/task?task_type=EXERCISE&limit=100&offset=<N>
```

Paginar con `offset` mientras `next` no sea `null` (igual que la Skill 2 — no asumir que la primera página trae todo).

### 2. Buscar coincidencia por texto

Buscar en `title` y `associated_slug` de todas las tareas traídas, comparación case-insensitive, tolerando coincidencia parcial (ej. "talent pipeline tracker" matchea `associated_slug: ai-eng-milestone-talent-pipeline-tracker` y el `title` "Milestone 3 — Talent Pipeline Tracker").

- **Ninguna coincidencia** → "No encontré ningún trabajo tuyo que coincida con '...'."
- **Coincidencias con distintos `associated_slug`** (trabajos realmente distintos, no duplicados de cohorte) → listarlos y preguntar cuál, no adivinar.
- **Coincidencias que comparten el mismo `associated_slug`** → ir al paso 3 (dedup).

### 3. Deduplicar por `associated_slug`

Cuando el mismo `associated_slug` aparece en más de una instancia (pasa seguido: un cohorte lo tiene `PENDING` sin tocar y otro lo tiene entregado — caso real confirmado con Talent Pipeline Tracker el 2026-09-27), quedarse con la instancia que tenga `delivered_at` no nulo. Si hay más de una con `delivered_at` no nulo, usar la más reciente. Si ninguna tiene `delivered_at`, usar la que tenga `task_status != PENDING`, y si tampoco hay ninguna así, tomar cualquiera y decir explícitamente que el trabajo todavía no fue entregado en ningún cohorte.

### 4. Etiquetar el estado (misma tabla que la Skill 2 — `4geeks-projects-status`)

`APPROVED` y `REJECTED` son valores de `revision_status`, no de `task_status`. No confundir los dos campos.

| task_status | revision_status | Estado a mostrar |
|---|---|---|
| PENDING | (cualquiera) | 🔴 Pendiente (no entregado) |
| DONE | PENDING | 🟡 Entregado, esperando revisión |
| DONE | APPROVED | 🟢 Calificado - Aprobado |
| DONE | REJECTED | 🟠 Calificado - Rechazado (requiere corrección) |
| (otra combinación no vista) | — | Mostrar los valores crudos de ambos campos y avisar que es una combinación nueva no mapeada |

### 5. Fechas — convertir a hora de España antes de mostrar y de contar

`opened_at` y `delivered_at` vienen en UTC (ISO con `Z`). **Convertir ambas a Europe/Madrid (con DST correcto, CEST/CET según la fecha) antes de mostrarlas y antes de calcular la diferencia en días.** Nunca mostrar ni calcular sobre la hora UTC cruda.

- Si falta `opened_at` → "No tiene fecha de apertura registrada."
- Si `delivered_at` es `null` → "Todavía no está entregado (según esta instancia), no tiene fecha de entrega." No calcular días.
- Si están las dos (ya convertidas a España) → calcular la diferencia en **días de calendario**: la diferencia entre la fecha civil (día/mes/año) de entrega y la fecha civil de apertura, ambas ya en hora de España, sin importar la hora del día de cada una. Ejemplo real: abierto el 19/9 22:31 (España), entregado el 23/9 01:14 (España) → 4 días de calendario (19→20→21→22→23), no 3. **No usar "horas completas transcurridas / 24, redondeado hacia abajo"**: eso da un número distinto y menos intuitivo que la diferencia de fecha civil, y fue la causa de una contradicción detectada por Caro el 2026-09-27. Mostrar siempre con la aclaración fija:
  `"Abierto el <fecha España>, entregado el <fecha España> → N días de calendario (no es tiempo trabajado, es tiempo transcurrido entre apertura y entrega)."`

### 6. Estimación del curso (`duration` del asset)

Con el `associated_slug` de la instancia elegida:

```
GET /v1/registry/asset/{associated_slug}
```

Leer el campo `duration`. **No dar por confirmada la unidad.** El valor visto en datos reales (Talent Pipeline Tracker: `duration: 3`) no trae unidad documentada en la respuesta de la API — no asumir "horas" salvo que en algún momento se confirme la unidad en la documentación oficial o la respuesta de la API la incluya explícitamente.

- Si `duration` existe → `"4Geeks lo estima en <valor> (campo duration, probablemente horas)."`
- Si `duration` es `null` o no existe → `"4Geeks no tiene una estimación cargada para este proyecto."`
- Siempre con la aclaración fija: "estimación del curso, no lo que vos tardaste."

### 7. Formato de salida

```
📋 <title>
Cohorte: <cohort.name>
Estado: <etiqueta del paso 4>

📅 Abierto el <fecha España>, entregado el <fecha España> → <N> días de calendario
   (no es tiempo trabajado, es tiempo transcurrido entre apertura y entrega)

⏱️ 4Geeks lo estima en <duration> (campo duration, probablemente horas)
   (estimación del curso, no lo que vos tardaste)
```

Si falta algún dato (fecha o duration), reemplazar esa línea por el mensaje correspondiente del paso 5 o 6, sin inventar ni omitir silenciosamente.

## Manejo de errores

Última línea `HTTP_STATUS:<código>` de cada llamada al wrapper:
- **200** → seguir con el procesamiento normal.
- **401/403** → parar, avisar a Caro con el código exacto, no reintentar.
- **no_token** → avisar que falta configurar la credencial por SSH; no pedirla por chat.
- **curl_error** → error de conexión, reportar sin reintentar en bucle.
- Cualquier otro código → reportar código y cuerpo, sin reintentar en bucle.

## Notas

- Esta skill NO calcula "tiempo trabajado": ese dato no existe en la API para cuentas de estudiante (investigado y confirmado el 2026-09-27). Si en el futuro aparece un campo real de tiempo trabajado, es una revisión explícita de esta skill, no un supuesto.
- La unidad de `duration` no está confirmada; si Caro o alguien más la confirma después (por ejemplo consultando documentación oficial de BreatheCode), actualizar esta skill para decir "horas" con seguridad — mientras tanto, "probablemente horas" es lo más honesto que se puede decir.
- Los "días de calendario" son diferencia de fecha civil en hora de España, no horas completas transcurridas divididas por 24. Confirmado por Caro el 2026-09-27 tras detectar la contradicción en la primera versión de esta skill.
- Depende de `/usr/local/bin/4geeks-get`, igual que las otras skills de 4Geeks. Si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearlo.
- Comparte la tabla de estados con la Skill 2 (`4geeks-projects-status`) a propósito, para que Caro vea siempre las mismas etiquetas en todas las skills de 4Geeks. Si esa tabla cambia allá, debe cambiar acá también.
