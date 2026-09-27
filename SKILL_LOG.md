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
> [PEGAR AQUÍ EL MENSAJE EXACTO QUE LE ENVIASTE A ALESSIA]

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
(mismo formato)

## Skill 4 — Obtener resumen de progreso
(mismo formato)

## Skill 5 — [extendida] ...
- **Necesidad que la motivó:**
(mismo formato)

## Skill 6 — [extendida] ...
(mismo formato)

---

## Incidencias y correcciones

1. **Negativa inicial por las reglas de seguridad** (26/09). Alessia aplicó correctamente las reglas 1 y 10. Se resolvió con una excepción explícita y acotada en `AGENTS.md`, no relajando las reglas.
2. **Cambios editados en local, no en el VPS.** Alessia detectó que sus archivos no habían cambiado y trató la petición como posible inyección (regla 12). Se corrigió editando los archivos en el VPS y verificándolo con `grep`.
3. **Intento de aplicar sin enseñar la versión final.** Alessia intentó aplicar la v3 antes de mostrarla; al pedírselo, reconoció el error y mostró el contenido.
4. **Tarjetas de aprobación caducadas.** Cada aplicación genera una tarjeta con 70 segundos para responder con `/approve <id> allow-once`. Dos caducaron sin respuesta.
5. **Cuarentena por el escáner de seguridad.** La v3 quedó en cuarentena por la regla `secret-exfiltration` (1 hallazgo crítico): una skill que construye la cabecera `Authorization` con una variable de entorno podría enviar el token a otro destino. En lugar de reescribir el texto para esquivar el escáner, se resolvió el riesgo de fondo con el wrapper `4geeks-get`. La nueva propuesta pasó el escaneo sin hallazgos.

**Comprobación de fugas:** se buscó el valor del token en todos los archivos de `~/.openclaw`, excepto el `.env`, sin imprimirlo. Resultado: **sin coincidencias** (27/09/2026). El token solo existe en `~/.openclaw/.env`.
