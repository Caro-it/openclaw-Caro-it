---
name: "4geeks-course-progress"
description: "Resumen numérico del avance de Caro en 4Geeks (tareas asignadas hasta ahora), deduplicado por associated_slug. Solo lectura."
---

# 4geeks-course-progress

Responde "¿cuánto llevo avanzado?", "¿cómo voy en el curso?", "dame un resumen de mi progreso en 4Geeks". Da un **resumen con números**, no una lista de proyectos (eso ya lo hacen las Skills 2 y 3: `4geeks-projects-status` y `4geeks-projects-pending`). Solo lectura. No entrega ni actualiza nada.

## Cuándo usarla
Cuando Caro pida una visión general de avance/progreso del bootcamp. No cubre el detalle de qué proyecto falta o su estado individual (Skills 2 y 3).

## Cómo está construida
Toda la comunicación pasa por el wrapper del sistema `4geeks-get <ruta>` (igual que las Skills 1-3). No usa `curl` directo, no toca variables de entorno de credenciales.

**Regla de trabajo en memoria (importante, agregada 2026-09-27):** las respuestas de la API se procesan solo en memoria durante la ejecución. Nunca se escriben en disco, nunca se guardan en el workspace (que es un repo de git) ni en `memory/`, ni siquiera como archivo temporal de trabajo. Si en algún momento hace falta inspeccionar datos crudos para depurar, se hace con variables en memoria del proceso que corre la skill, no con archivos.

## Reglas que respeta (AGENTS.md 10a)
- Solo GET (lo impone el wrapper).
- Solo datos de la propia cuenta de estudiante de Caro.
- Ninguna credencial se muestra, copia o guarda.
- Nada se persiste en disco (ver regla de trabajo en memoria arriba).
- Si el wrapper responde 401/403: parar y avisar a Caro, sin reintentar.
- Si falla dos veces seguidas (regla 9 AGENTS.md): parar y contar qué pasó.

## Endpoints (dos, con roles distintos)

1. **Tareas**, para todas las métricas de avance:
   ```
   GET /v1/assignment/user/me/task?limit=100&offset=<N>
   ```
   Paginar mientras `next` no sea `null`. **No filtrar por `task_type`**: se traen LESSON + EXERCISE + PROJECT juntas (QUIZ no aparece en los datos reales de Caro a 2026-09-27; si apareciera en el futuro, tratarlo igual que EXERCISE — completado/no completado — y avisar a Caro para revisar si merece su propio tratamiento).

2. **Cohortes activas**, únicamente para la línea de contexto "cohorte(s) activa(s)":
   ```
   GET /v1/admissions/user/me
   ```
   Se usa el array `cohorts[]` de la respuesta, filtrando por `cohort.stage == "STARTED"`. Se muestran solo los nombres (`cohort.name`), sin conteo de tareas ni "principal" (si hay varias con `stage=STARTED` al mismo tiempo, se listan todas por igual).

   **No usar** `GET /v1/admissions/academy/cohort/me`: devuelve 403 porque pide un `academy_id`/header `Academy` que esta cuenta no tiene configurado. Confirmado con datos reales el 2026-09-27.

   **No usar** `educational_status=ACTIVE` como criterio de cohorte activa: en los datos reales de Caro, 23 cohortes distintas tienen `educational_status=ACTIVE` simultáneamente (es un estado de inscripción, no de cursada actual). El campo que sí distingue "la cursás ahora" es `stage=STARTED`.

## Deduplicación (igual criterio que Skills 2 y 3)

Agrupar todos los resultados de `/v1/assignment/user/me/task` por `associated_slug` (estable entre cohortes; el `title` puede repetirse por coincidencia y el `id` cambia si BreatheCode reasigna el módulo a una cohorte nueva). Confirmado con datos reales el 2026-09-27 que el mismo problema de reasignación entre cohortes ocurre también en LESSON y EXERCISE, no solo en PROJECT (38 slugs de EXERCISE y 2 de LESSON tenían más de una instancia en cohortes distintas).

Para cada grupo (tarea real), evaluar el conjunto de instancias:

- **Aprobado:** alguna instancia con `task_status="DONE"` y `revision_status="APPROVED"`.
- **Entregado en revisión:** alguna instancia con `task_status="DONE"` y `revision_status="PENDING"`, y ninguna `APPROVED`.
- **No entregado:** todas las instancias del grupo son `task_status="PENDING"`.
- **A corregir:** alguna instancia `task_status="DONE"` con `revision_status="REJECTED"`, sin hermana `APPROVED`/`PENDING` de revisión. (No se vio ningún caso real a 2026-09-27, pero la categoría queda prevista.)

## Tratamiento especial de LESSON (importante, agregado 2026-09-27)

Las lecciones (`task_type="LESSON"`) no pasan por un flujo de revisión real: en los datos de Caro, todas las lecciones con `task_status="DONE"` tienen `revision_status="PENDING"` (nunca `APPROVED"`), porque nadie las aprueba manualmente. Por eso:

- Para LESSON, **no se distingue "aprobado" de "entregado en revisión"**: cualquier instancia `DONE` (con cualquier `revision_status`) cuenta simplemente como **"completada"**.
- El desglose por tipo muestra Lecciones como `completadas/total`, sin columna de aprobado.
- El **% de aprobado global no incluye lecciones en el denominador ni en el numerador**: se calcula solo sobre PROJECT + EXERCISE, para que no baje artificialmente por algo que estructuralmente nunca se marca como aprobado. Ejemplo real (2026-09-27): 72 aprobados de 134 (proyectos + ejercicios) = 54%, no 72 de 171.
- El **% de "entregado" global** (línea principal de avance) sí incluye las 3 categorías, porque para lecciones "completada" y "entregada" son lo mismo.

## Métricas del reporte

1. **Línea principal — entregado sobre el total de tareas asignadas hasta ahora** (todas las categorías, deduplicadas):
   `Vas por el <entregado_total/total_unico>% de las tareas asignadas hasta ahora (<entregado_total> de <total_unico>).`
   - "de las tareas asignadas hasta ahora", nunca "del curso": el total no incluye módulos futuros aún no asignados.
2. **Aprobado, calculado solo sobre proyectos + ejercicios:**
   `De los proyectos y ejercicios, el <aprobado/(proyectos+ejercicios)>% ya está aprobado (<aprobado> de <proyectos+ejercicios>).`
3. **Desglose por tipo:**
   - Proyectos: `<entregados>/<total>` entregados (`<aprobados>` aprobados)
   - Ejercicios: `<entregados>/<total>` entregados (`<aprobados>` aprobados)
   - Lecciones: `<completadas>/<total>` completadas
4. **Pendientes agregados:** cuántas "no entregadas" + cuántas "a corregir" en total (solo el número, sin nombrarlas — para el detalle están las Skills 2 y 3).
5. **Duplicados excluidos:** conteo de filas crudas descartadas por dedup entre cohortes (mismo estilo que Skill 3), sin listarlas.
6. **Cohorte(s) activa(s):** nombres de las cohortes con `stage="STARTED"`, sin conteo ni jerarquía entre ellas.

### Ejemplo real (2026-09-27, para ilustrar formato, no hardcodear)

```
Vas por el 58% de las tareas asignadas hasta ahora (100 de 171).
De los proyectos y ejercicios, el 54% ya está aprobado (72 de 134).

Por tipo:
- Proyectos: 20/29 entregados (16 aprobados)
- Ejercicios: 67/105 entregados (56 aprobados)
- Lecciones: 13/37 completadas

Pendiente: 71 no entregadas, 0 a corregir.
(Se excluyeron 52 duplicados entre cohortes.)

Cohorte(s) activa(s): Ingeniería de Prompts para Principiantes, spain-aie-pt-4.
```

## Manejo de errores

Última línea `HTTP_STATUS:<código>` de cada llamada al wrapper (igual que las otras skills):
- **200** → seguir con el procesamiento normal.
- **401/403** → parar, avisar a Caro con el código exacto, no reintentar. (Nota: `/v1/admissions/academy/cohort/me` da 403 por diseño según lo explicado arriba; no es este caso de error, simplemente no se usa ese endpoint.)
- **no_token** → avisar que falta configurar la credencial por SSH; no pedirla por chat.
- **curl_error** → error de conexión, reportar sin reintentar en bucle.
- Cualquier otro código → reportar código y cuerpo de respuesta, sin reintentar en bucle.

## Notas

- Esta skill NO entrega proyectos ni actualiza URLs: son escrituras, fuera de alcance.
- Esta skill NO lista proyectos individuales (eso son las Skills 2 y 3); si Caro pide el detalle después de ver el resumen, remitir a esas skills.
- No toca `/v1/activity/me` (tiempo de estudio): es otro tipo de dato, posible skill futura si Caro la pide.
- Si aparece una combinación de estados fuera de las contempladas, o `task_type=QUIZ` en volumen, no inventar una interpretación: mostrar los datos crudos del grupo y avisar a Caro.
- Depende de `/usr/local/bin/4geeks-get`, igual que las otras skills de 4Geeks. Si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearlo.
