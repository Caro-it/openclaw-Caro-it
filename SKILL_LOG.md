# SKILL_LOG — Mi Asistente 4Geeks (Alessia / OpenClaw)

Agente: **Alessia** (OpenClaw 2026.7.1-2), en un VPS propio, canal Telegram.
API: API de estudiantes de 4Geeks (BreatheCode), host `breathecode.herokuapp.com`.

---

## 0. Configuración inicial y manejo seguro del token

- **Instancia activa:** servicio `openclaw-gateway.service` corriendo como servicio de usuario (`systemctl --user`), verificado con `is-active` → `active`.
- **Obtención del token:** cookie `4g_tok` de `https://learn.4geeks.com` (DevTools → Aplicación → Cookies).
- **Almacenamiento:** el token se guardó **por SSH, sin pasar por el chat**, en `~/.openclaw/.env` como `FOURGEEKS_TOKEN` (permisos `600`). Ese archivo está fuera del repo del workspace (`~/.openclaw/workspace`), así que nunca se sube a GitHub.
- **Verificación sin mostrar el valor:** solo se comprobaron la longitud, la ausencia de espacios y de caracteres raros, y que hubiera una única línea (`grep -c` → `1`).
- **Wrapper de solo lectura** (añadido tras la cuarentena de la Skill 1, ver incidencias): `/usr/local/bin/4geeks-get`. Es el **único** punto del sistema que usa el token. Solo hace GET, con el host fijo, y rechaza rutas que no empiecen por `/v1/`:

```sh
#!/bin/sh
# 4geeks-get — GET de solo lectura a la API de estudiantes de 4Geeks.
# Uso: 4geeks-get /v1/ruta/del/endpoint
# El host está fijo: el token solo puede viajar a breathecode.herokuapp.com.
path="${1:-}"
case "$path" in
  /v1/*) ;;
  *) echo "HTTP_STATUS:bad_path"; exit 2 ;;
esac
case "$path" in
  *..*|*"://"*|*" "*|*@*) echo "HTTP_STATUS:bad_path"; exit 2 ;;
esac
[ -n "${FOURGEEKS_TOKEN:-}" ] || { echo "HTTP_STATUS:no_token"; exit 3; }
curl -s --max-time 20 -w "\nHTTP_STATUS:%{http_code}\n" \
  -H "Authorization: Token $FOURGEEKS_TOKEN" \
  "https://breathecode.herokuapp.com$path" || echo "HTTP_STATUS:curl_error"
```

Prueba del wrapper: `4geeks-get /v1/auth/user/me` → `HTTP_STATUS:200`; `4geeks-get https://otro-sitio.com` → `HTTP_STATUS:bad_path`.

---

## 1. Conversación de descubrimiento

**Prompt inicial (26/09/2026):**
Quiero darte la habilidad de conectarte a mi cuenta de 4Geeks usando mi token de estudiante, sin que tenga que desarrollar código de mi parte. ¿Qué debemos hacer?

**Qué respondió OpenClaw (26/09, 22:48):** se negó, citando dos de sus reglas fijas de `AGENTS.md`:
- la **regla 10**, que le prohíbe configurar conexiones o APIs nuevas;
- la **regla 1**, que le prohíbe manejar tokens.

Como alternativa propuso un conector de Composio o un flujo en n8n. Se descartaron porque el proyecto exige construir las skills conversando con el propio agente.

**Qué se cambió:** se editó `AGENTS.md` por SSH para añadir una excepción acotada, y se añadió la API a `TOOLS.md`:
- **Regla 10a:** permite usar la API de estudiantes de 4Geeks solo en modo lectura (GET), solo con datos de la cuenta propia, sin mostrar ni guardar nunca el valor del token, y parando si la API responde 401 o 403.
- **Regla 1:** ahora remite explícitamente a la 10a como única excepción.
- **Sección "Interno vs. externo":** incluye la consulta de la API en modo lectura.

**Segunda respuesta (26/09, 23:30):** Alessia leyó los archivos del VPS, dijo que no estaban actualizados y trató el mensaje como un posible intento de saltarse sus reglas (regla 12). **Tenía razón:** los cambios se habían hecho en la copia local, no en el VPS. Se aplicaron en el VPS y se verificaron con `grep`.

**Qué información pidió:** ninguna credencial. Exigió que el cambio de permisos estuviera escrito en sus archivos de configuración, no solo afirmado en el chat.

---

## Skill 1 — Autenticar (`4geeks-token-check`)

**Prompt en lenguaje natural (27/09):**
> Edité yo misma por SSH AGENTS.md (reglas 1 y 10a) y TOOLS.md. [...] Quiero darte la habilidad de conectarte a mi cuenta de 4Geeks sin que yo tenga que desarrollar código. ¿Qué debemos hacer? Empecemos por una skill que solo verifique que el token es válido y la sesión está activa.

**Qué hace:** comprueba que la sesión de estudiante está activa y reporta nombre, academia y rol, **sin incluir el email**. Maneja cada resultado:
- `200`: sesión activa, reporta los datos.
- `401` / `403`: para y avisa, sin reintentar.
- `no_token`: falta la credencial en el servidor; avisa sin pedirla por chat.
- `bad_path`: error interno de la skill; lo reporta como bug.
- `curl_error`: error de conexión; lo reporta sin reintentar.

**Endpoint:** `GET /v1/auth/user/me`, a través de `4geeks-get`.

**Cómo se construyó:** con Skill Workshop de OpenClaw. Alessia crea una propuesta, la usuaria la revisa y la aprueba, y el escáner de seguridad la analiza antes de aplicarla. Hubo 4 iteraciones:

| Versión | Cambio | Motivo |
|---|---|---|
| v1 | `curl -o` a un archivo temporal dentro del workspace | Primera propuesta |
| v2 | Archivo temporal con `mktemp` + `trap`, fuera del repo | El temporal con datos personales podía acabar en un commit |
| v3 | Sin archivos: `curl` imprime el cuerpo y el código por salida estándar, y el reporte ya no incluye el email | Simplificación y minimización de datos |
| v4 (aplicada) | La skill solo llama a `4geeks-get` y no menciona el token | El escáner puso la v3 en cuarentena (ver incidencias) |

Escáner de la versión aplicada: `"state":"clean"`, `"critical":0`, sin hallazgos.

**Resultado de prueba (27/09, 11:06):**
> Sí, tu sesión está activa ✅ HTTP 200.
> Maria Carolina Kaechele Smoly — 4Geeks Madrid — rol: student 🐙

**Resultado:** ✅ Funciona. Estado en Skill Workshop: `applied`.

---

## Skill 2 — Obtener mis proyectos

## Skill 2 — Obtener mis proyectos (`4geeks-projects-status`)

**Prompt en lenguaje natural (27/09):**
> Skill 1 lista y subida. Vamos con la Skill 2: quiero que puedas recuperar la lista de proyectos que tengo asignados en 4Geeks con su estado actual (pendiente, entregado, calificado). El mapa oficial de endpoints está en `docs/4geeks-api-map.md` del workspace. Buscá ahí el endpoint correcto, probalo con `4geeks-get` (igual que la Skill 1) y mostrame la propuesta completa antes de aplicarla. Una sola responsabilidad: solo listar proyectos y su estado.

**Qué hace:** lista todos los proyectos asignados con su estado. Empieza con un recuento por estado y después los agrupa. Recorre todas las páginas de la API (`next` hasta `null`). Si aparece una combinación de estados desconocida, muestra los valores crudos en vez de inventar una interpretación.

**Endpoint:** `GET /v1/assignment/user/me/task?task_type=PROJECT&limit=100&offset=<N>`, a través de `4geeks-get`.

**Hallazgo con datos reales:** el estado no sale de un solo campo, sino de cruzar dos:

| task_status | revision_status | Estado |
|---|---|---|
| PENDING | (cualquiera) | 🔴 Pendiente |
| DONE | PENDING | 🟡 Entregado, esperando revisión |
| DONE | APPROVED | 🟢 Calificado - Aprobado |
| DONE | REJECTED | 🟠 Calificado - Rechazado |

**Revisión antes de aplicar (v1 → v2):**
- Se quitó "¿qué me falta entregar?" de los disparadores, porque es la pregunta de la Skill 3. Ahora la skill aclara explícitamente que no la cubre.
- Se añadió un recuento por estado al principio, porque 36 proyectos seguidos en Telegram eran ilegibles.

Escáner: `"state":"clean"`, `"critical":0`. Estado: `applied`.

**Resultado de prueba (27/09), extracto:**
> 🔴 16 · 🟡 4 · 🟢 16 · 🟠 0
>
> 🔴 Pendientes (no entregados):
> • Command Line Challenge — spain-aie-pt-4
> • Backend Architecture Proposal — Backend development with Coding Agents
> • …
>
> 🟡 Entregados, esperando revisión:
> • Talk to the Machine (chat con IA real) — Frontend development with Coding Agents
> • …
>
> 🟢 Calificados - Aprobados:
> • A simple Dashboard with Tailwind CSS — Web UI fundamentals with Tailwind
> • …

**Resultado:** ✅ Funciona.

**Observación para la Skill 3:** varios proyectos aparecen dos veces, pendientes en la cohorte general (`spain-aie-pt-4`) y entregados o aprobados en la cohorte del módulo. La Skill 2 los muestra tal como los devuelve la API. La Skill 3 tendrá que deduplicarlos para no dar como pendiente algo ya entregado.

## Skill 3 — Obtener trabajo pendiente

## Skill 3 — Obtener trabajo pendiente (`4geeks-projects-pending`)

**Prompt en lenguaje natural (27/09):**
> Skill 2 lista. Vamos con la Skill 3: quiero que me digas específicamente qué me falta completar. Ojo con algo que vimos en la Skill 2: varios proyectos aparecen dos veces (pendiente en spain-aie-pt-4 y entregado o aprobado en la cohorte del módulo). Revisá en la respuesta real qué campo identifica que son el mismo proyecto (no solo el título), y que la skill no cuente como pendiente algo que ya está entregado en otra cohorte. Una sola responsabilidad: solo lo pendiente. Mostrame la propuesta antes de aplicarla.

**Qué hace:** lista solo el trabajo pendiente real. Agrupa las copias de un mismo proyecto por `associated_slug` y lo excluye si alguna copia está entregada o aprobada. Los rechazados se muestran como "🟠 a corregir", con su feedback. Termina indicando cuántos duplicados excluyó.

**Endpoint:** `GET /v1/assignment/user/me/task?task_type=PROJECT&limit=100&offset=<N>`, a través de `4geeks-get`. Es el mismo endpoint que la Skill 2, pero con una responsabilidad distinta: la Skill 2 muestra el estado de todo; la Skill 3, solo lo que falta.

**Problema real descubierto:** 4Geeks reasigna el mismo proyecto a la cohorte general (`spain-aie-pt-4`) con un `id` nuevo y `task_status=PENDING`, aunque ya esté entregado en la cohorte del módulo. Sin deduplicar, salían 16 pendientes; los reales son 9. El título no sirve para identificarlos; el campo fiable es `associated_slug`.

**Revisión antes de aplicar (v1 → v2):** la v1 excluía cualquier proyecto con alguna copia en `DONE`, pero así un proyecto `DONE` + `REJECTED` (que hay que corregir) desaparecía de la lista. En la v2, solo se excluye si alguna copia está `DONE` + `APPROVED` o `DONE` + `PENDING`. Además se añadió la línea de duplicados excluidos.

Escáner: `"state":"clean"`, `"critical":0`.

**Resultado de prueba (27/09, 12:24):**
> Te faltan 9 proyectos por resolver: 9 sin entregar, 0 a corregir.
> 1. Backend Architecture Proposal — Backend development with Coding Agents
> 2. Company Monorepo Containerization — Container applications with Docker
> 3. Securing the API: Authentication and Route Restriction in FastAPI — Authentication in web applications
> 4. EduTrack Data Audit — Managing relational databases with FastAPI
> 5. Centralized Incident Manager — Error handling, debugging and testing
> 6. Error Handling — Error handling, debugging and testing
> 7. My 4Geeks Assistant — Teaching OpenClaw to Track Your Progress — Advanced personal assistants with Openclaw
> 8. Enhacing development with agent skills - Financial dashboard — Working with AI coding agents
> 9. Todo List CLI with Python — spain-aie-pt-4
>
> (Se excluyeron 7 duplicados ya entregados en otra cohorte.)

**Resultado:** ✅ Funciona. Los números coinciden con la Skill 2: 16 filas pendientes menos 7 duplicados = 9.

## Skill 4 — Obtener resumen de progreso

## Skill 4 — Obtener resumen de progreso (`4geeks-course-progress`)

**Prompt en lenguaje natural (27/09):**
> Skill 3 lista. Vamos con la Skill 4: quiero una visión general de cuánto he avanzado en el curso. Revisá en `docs/4geeks-api-map.md` qué endpoint(s) sirven (por ejemplo, las tareas de todos los tipos, no solo proyectos, o mi cohorte activa) y proponeme qué métricas mostrar. Tiene que ser un resumen con números, no una lista de proyectos (eso ya lo hacen las Skills 2 y 3), y aplicando la misma deduplicación por `associated_slug`. Mostrame la propuesta antes de aplicarla.

**Qué hace:** da un resumen numérico del avance:
- porcentaje entregado sobre las tareas asignadas hasta ahora;
- porcentaje aprobado sobre proyectos y ejercicios;
- desglose por tipo de tarea;
- pendientes totales;
- duplicados excluidos;
- nombre de las cohortes activas.

No lista tareas individuales, porque eso lo hacen las Skills 2 y 3.

**Endpoints:**
- `GET /v1/assignment/user/me/task`, paginado y sin filtrar por tipo, para las tareas.
- `GET /v1/admissions/user/me`, solo para los nombres de las cohortes con `stage=STARTED`.

Ambos a través de `4geeks-get`.

**Decisiones tomadas con datos reales:**
- La deduplicación por `associated_slug` funciona en todos los tipos: 223 filas se quedan en 171 tareas únicas (52 duplicados).
- No hay quizzes en los datos, así que se excluyen.
- `/v1/admissions/academy/cohort/me` devolvía 403. Además, 23 cohortes figuraban como `educational_status=ACTIVE`, así que ese campo no identifica la cohorte actual. Se usa `stage=STARTED`, con el que hoy salen 2.
- El total son las tareas **asignadas hasta ahora**, no el curso completo, porque los módulos futuros aún no aparecen.
- Se separa "entregado" de "aprobado". Las lecciones cuentan solo como "completadas", porque ninguna aparece como aprobada (0 de 13 entregadas). Es una deducción a partir de los datos, no algo confirmado por 4Geeks. Por eso el porcentaje de aprobado se calcula solo sobre proyectos y ejercicios.
- Todo se procesa en memoria, sin escribir en disco (ver incidencia 6).

Escáner: `"state":"clean"`, `"critical":0`. Estado: `applied`.

**Resultado de prueba (27/09, 12:49):**
> Vas por el 58% de las tareas asignadas hasta ahora (100 de 171).
> De los proyectos y ejercicios, el 54% ya está aprobado (72 de 134).
>
> Por tipo:
> • Proyectos: 20/29 entregados (16 aprobados)
> • Ejercicios: 67/105 entregados (56 aprobados)
> • Lecciones: 13/37 completadas
>
> Pendiente: 71 no entregadas, 0 a corregir.
> (Se excluyeron 52 duplicados entre cohortes.)
>
> Cohorte(s) activa(s): Ingeniería de Prompts para Principiantes, spain-aie-pt-4.

**Resultado:** ✅ Funciona. Los proyectos coinciden con las Skills 2 y 3: 29 únicos, 9 pendientes.

## Skill 5 — [extendida] ...
- **Necesidad que la motivó:**

## Skill 5 — [extendida] Próximos eventos (`4geeks-events-upcoming`)

**Necesidad que la motivó:** no quiero perderme workshops por no revisar el campus.

**Prompt en lenguaje natural (27/09):**
> Skill 4 lista. Ahora las extendidas. Skill 5: no quiero perderme workshops por no revisar el campus. Quiero saber qué eventos de 4Geeks vienen (workshops, charlas). Mirá en `docs/4geeks-api-map.md` el endpoint de eventos (`/v1/events/all`, con `upcoming` y `academy`). Sacá mi ID de academia de mis datos reales, sin suponerlo. Mostrá solo los próximos, con título, fecha y hora convertidas a hora de España, y el tipo o enlace si viene. Si no hay eventos próximos, decilo claramente. Solo lectura, en memoria, sin guardar nada en disco. Probalo con datos reales y mostrame la propuesta antes de aplicarla.

**Qué hace:** lista los próximos eventos de 4Geeks con título, fecha y hora en España (de UTC a Europe/Madrid, teniendo en cuenta el cambio de hora), tipo, modalidad y enlace, o avisa si no lo hay. Si no hay eventos, lo dice claramente.

**Endpoint:** `GET /v1/events/all?upcoming=true`, a través de `4geeks-get`.

**Hallazgo con datos reales:** filtrando por mi academia (Madrid, `id=6`, obtenido de mis datos) salían **0 eventos**, porque 4Geeks publica los eventos online en una academia global (`id=47`). Por eso se pide sin filtro y se filtra en memoria:
- **Online:** de cualquier academia.
- **Presencial:** solo si es de Madrid.

Esta regla la decidí yo, y la skill indica que no puede cambiarse sin mi confirmación.

Escáner: `"state":"clean"`, `"critical":0`. Estado: `applied`.

**Resultado de prueba (27/09, 13:03):**
> Próximos eventos de 4Geeks:
> 📅 El mapa real del AI Engineer
> Miércoles 7/10 · 19:00 a 20:15 España
> Tipo: Tendencias de IA (Público)
> Modalidad: online
> Enlace: sin enlace en la API — probablemente hay que anotarse desde la plataforma del campus.

**Resultado:** ✅ Funciona. Durante la exploración, Alessia dijo "martes 7/10", pero el 7/10/2026 es miércoles; la skill ya aplicada lo muestra bien. Se corrigió el ejemplo del `SKILL.md`.

## Skill 6 — [extendida] ...
## Skill 6 — [extendida] Tiempo de un trabajo (`4geeks-task-timeline`)

**Necesidad que la motivó:** quiero saber qué tiempo me llevó un trabajo concreto.

**Prompt en lenguaje natural (27/09):**
> Skill 5 lista. Skill 6: quiero saber qué tiempo me llevó un trabajo concreto (un proyecto o un ejercicio), por ejemplo "¿cuánto tiempo me llevó el Talent Pipeline Tracker?". Mirá en `docs/4geeks-api-map.md` el endpoint `/v1/activity/me` (acepta `date_start` y `date_end`). Antes de diseñar nada, probalo con datos reales y contame qué devuelve de verdad [...]. Si la API no permite saber el tiempo por trabajo, decímelo claramente y proponeme la alternativa más cercana, sin inventar números ni estimarlos a partir de fechas. [...]

**Exploración con datos reales, antes de diseñar:**
- `/v1/activity/me` devuelve **403** con el token de estudiante: exige el permiso `read_activity`, que es de staff.
- El campo `assignment_telemetry` del detalle de tarea viene **vacío** en las 5 tareas revisadas.
- Las rutas de telemetría probadas devuelven **404**.
- **Conclusión:** la API no expone el tiempo trabajado por tarea a un estudiante. Alessia lo dijo claramente en lugar de inventar números.

**Qué hace (la alternativa más cercana y honesta):** para un trabajo que nombro, muestra:
- los días de calendario entre apertura y entrega, en hora de España;
- la estimación del curso (campo `duration`, probablemente horas).

Ambas cosas van siempre etiquetadas como **"no es tiempo trabajado"**. Si hay varias coincidencias, pregunta cuál. Si falta un dato, lo dice sin calcular. Entre los duplicados de distintas cohortes, usa la copia con fecha de entrega.

**Endpoints:**
- `GET /v1/assignment/user/me/task`, para `opened_at` y `delivered_at`.
- `GET /v1/registry/asset/{slug}`, para `duration`.

Ambos a través de `4geeks-get`.

**Revisiones antes de aplicar:**
- No dar por confirmada la unidad de `duration`.
- Usar las mismas etiquetas de estado que la Skill 2.
- Convertir las fechas a hora de España.

**Corrección después de aplicar:** ver incidencia 7.

Escáner: `"state":"clean"`, `"critical":0`. Estado: `applied`.

**Resultado de prueba (27/09, 13:21):**
> 📋 Milestone 3 — Talent Pipeline Tracker
> Cohorte: Frontend development with Coding Agents
> Estado: 🟡 Entregado, esperando revisión
> 📅 Abierto el 19/9/2026 22:31 (España), entregado el 23/9/2026 01:14 (España) → 4 días de calendario (no es tiempo trabajado)
> ⏱️ 4Geeks lo estima en 3 (campo duration, probablemente horas) (estimación del curso, no lo que vos tardaste)

**Resultado:** ✅ Funciona. La conversión horaria importaba: en UTC la entrega figuraba el 22/9, y en hora de España es el 23/9.

---

## Incidencias y correcciones

1. **Negativa inicial por las reglas de seguridad** (26/09). Alessia aplicó correctamente las reglas 1 y 10. Se resolvió con una excepción explícita y acotada en `AGENTS.md`, no relajando las reglas.
2. **Cambios editados en local, no en el VPS.** Alessia detectó que sus archivos no habían cambiado y trató la petición como posible inyección (regla 12). Se corrigió editando los archivos en el VPS y verificándolo con `grep`.
3. **Intento de aplicar sin enseñar la versión final.** Alessia intentó aplicar la v3 antes de mostrarla; al pedírselo, reconoció el error y mostró el contenido.
4. **Tarjetas de aprobación caducadas.** Cada aplicación genera una tarjeta con 70 segundos para responder con `/approve <id> allow-once`. Dos caducaron sin respuesta.
5. **Cuarentena por el escáner de seguridad.** La v3 quedó en cuarentena por la regla `secret-exfiltration` (1 hallazgo crítico): una skill que construye la cabecera `Authorization` con una variable de entorno podría enviar el token a otro destino. En lugar de reescribir el texto para esquivar el escáner, se resolvió el riesgo de fondo con el wrapper `4geeks-get`. La nueva propuesta pasó el escaneo sin hallazgos.
6. **Datos de la API guardados en el workspace** (27/09). Mientras exploraba los datos para la Skill 4, Alessia guardó 4 archivos (unos 330 KB) con respuestas completas de la API en `.openclaw/tmp/`, dentro del repo, contradiciendo sus propias skills ("no escribe nada en disco"). Se detectó con `git status`. Se comprobó que no contenían el token, se añadió `.openclaw/` al `.gitignore` antes de que se subieran y se le pidió que los borrara y trabajara en memoria. Alessia borró los archivos tras terminar el análisis (verificado con `ls`: carpeta vacía).
7. **Regla escrita y resultado real no coincidían (Skill 6).** El `SKILL.md` decía "días completos (redondeo hacia abajo)", que daría 3, pero la prueba mostró 4. Se detectó revisando el `SKILL.md` con `grep`. Se corrigió mediante una propuesta de actualización: ahora cuenta días de calendario en hora de España (del 19/9 al 23/9 = 4), y la descripción ya no da por confirmada la unidad de `duration`.
**Comprobación de fugas:** se buscó el valor del token en todos los archivos de `~/.openclaw`, excepto el `.env`, sin imprimirlo. Resultado: **sin coincidencias** (27/09/2026). El token solo existe en `~/.openclaw/.env`.
