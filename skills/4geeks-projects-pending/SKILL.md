---
name: "4geeks-projects-pending"
description: "Lista lo que a Caro le falta entregar en 4Geeks, deduplicando por associated_slug entre cohortes. Solo lectura."
---

# 4geeks-projects-pending

Responde específicamente "¿qué me falta entregar?": lista el trabajo pendiente real de Caro en BreatheCode (4Geeks), deduplicado por proyecto real (no por fila cruda de la API). Solo lectura. Una sola responsabilidad: solo lo pendiente real; no clasifica lo ya aprobado/en revisión como tema principal (eso es la Skill 2, `4geeks-projects-status`) y no entrega ni actualiza nada.

## Cuándo usarla
Cuando Caro pregunte "¿qué me falta completar/entregar?", "¿qué tengo pendiente de verdad?", "¿qué proyectos me quedan por hacer?", "¿qué tengo que corregir?". No cubre el estado general de lo ya calificado: eso es la Skill 2.

## Cómo está construida
Toda la comunicación pasa por el wrapper del sistema `4geeks-get <ruta>` (igual que `4geeks-token-check` y `4geeks-projects-status`). No usa `curl` directo, no toca variables de entorno de credenciales, no escribe nada en disco.

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

Igual que en la Skill 2: paginar mientras `next` no sea `null`, no filtrar por `task_status` en la query (se trae todo y se clasifica localmente).

## El problema que resuelve esta skill (importante, no obvio)

BreatheCode reasigna el mismo proyecto a una cohorte nueva (ej. `spain-aie-pt-4`) creando una **fila nueva con `id` distinto y `task_status=PENDING`**, aunque Caro ya lo haya entregado (y quizás aprobado) meses antes en la cohorte original del módulo. El `title` puede repetirse por coincidencia, así que **no sirve como identificador**. El campo confiable, confirmado con datos reales el 2026-09-27, es **`associated_slug`**: es estable para el mismo proyecto sin importar en qué cohorte o con qué `id` aparezca.

Ejemplo real: "Command Line Challenge" aparece como `id 972199` (`PENDING`, cohorte `spain-aie-pt-4`) y como `id 931702` (`DONE`/`APPROVED`, cohorte `Command Line, Git and Github`) — mismo `associated_slug: "exercise-terminal-challenge"`.

## Procedimiento

1. Traer todas las páginas del endpoint (igual que Skill 2).
2. Agrupar todos los resultados por `associated_slug`.
3. Para cada grupo (proyecto real), evaluar el conjunto de (`task_status`, `revision_status`) de todas sus instancias:
   - Si **alguna** instancia tiene `task_status="DONE"` y `revision_status` en (`"APPROVED"`, `"PENDING"`) → **no es pendiente**, se excluye de esta lista: ya está entregado y aprobado, o entregado y esperando que lo revisen.
   - En **cualquier otro caso** → es **pendiente real**, y se subclasifica así para mostrarlo:
     - Si **ninguna** instancia del grupo tiene `task_status="DONE"` (todas `PENDING`) → mostrar como "🔴 no entregado".
     - Si **alguna** instancia tiene `task_status="DONE"` y `revision_status="REJECTED"` (y ninguna instancia del grupo es `APPROVED`/`PENDING` de revisión) → mostrar como "🟠 a corregir" (usar el título, cohorte y `github_url`/feedback de esa instancia rechazada, porque tiene el contexto de la corrección).
   - Usar el título y la cohorte de la instancia más relevante para mostrar cada grupo: la rechazada si existe (caso "a corregir"), si no la de `created_at` más alto entre las `PENDING`.
4. Armar el reporte para Caro:
   - Primera línea: recuento total, ej. `Te faltan 9 proyectos por resolver: 8 sin entregar, 1 a corregir.`
   - Listado agrupado: primero los "🔴 no entregados", después los "🟠 a corregir" (si hay), con `title` y `cohort.name` de cada uno.
   - Última línea, siempre, con el conteo de duplicados excluidos por ya estar entregados en otra cohorte, **sin listarlos**: ej. `(Se excluyeron 7 duplicados ya entregados en otra cohorte — ver detalle con la Skill 2 si hace falta.)`
5. Manejo de errores en la última línea `HTTP_STATUS:<código>` (igual que Skill 2 y `4geeks-token-check`):
   - **200** → seguir con el procesamiento normal.
   - **401/403** → parar, avisar a Caro con el código exacto, no reintentar.
   - **no_token** → avisar que falta configurar la credencial por SSH; no pedirla por chat.
   - **curl_error** → error de conexión, reportar sin reintentar en bucle.
   - Cualquier otro código → reportar código y cuerpo de respuesta, sin reintentar en bucle.

## Notas
- Esta skill NO entrega proyectos (`POST .../deliver`) ni actualiza URLs (`PUT /v1/assignment/task/{id}`): son escrituras, fuera de alcance.
- Esta skill NO es la fuente principal de "qué tengo aprobado/en revisión": eso es la Skill 2 (`4geeks-projects-status`). Acá el foco es exclusivamente lo pendiente (no entregado o a corregir).
- El caso "a corregir" (`DONE`+`REJECTED` sin hermano `APPROVED`/`PENDING`) es la única situación donde esta skill sí muestra algo de contexto extra (feedback/`github_url`) para que Caro sepa qué corregir, ya que no es un simple "no entregado".
- Si aparece un grupo con una combinación rara (por ejemplo instancias con `task_status`/`revision_status` fuera de las vistas), no inventar una interpretación: mostrar los datos crudos del grupo y avisar a Caro.
- Depende de `/usr/local/bin/4geeks-get`, igual que las otras skills de 4Geeks. Si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearlo.
