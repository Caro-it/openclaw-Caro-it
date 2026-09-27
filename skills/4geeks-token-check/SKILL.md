---
name: "4geeks-token-check"
description: "Verifica que la sesión de estudiante en 4Geeks/BreatheCode está activa, usando el wrapper de solo lectura del servidor."
---

# 4geeks-token-check

Verifica que la sesión de estudiante de Caro en la API de BreatheCode (4Geeks) está activa. Solo lectura. Esta skill no maneja ninguna credencial directamente: usa un wrapper del sistema que ya la gestiona por dentro.

## Cuándo usarla
Cuando Caro pida comprobar si su acceso a 4Geeks sigue funcionando, o antes de usar otras skills de solo-lectura contra la API de BreatheCode, para confirmar que la sesión sigue viva.

## Cómo está construida esta skill
Toda la comunicación con la API pasa por un único comando del sistema: `4geeks-get <ruta>` (instalado en `/usr/local/bin/4geeks-get`, mantenido por Caro, fuera del alcance de esta skill). Esta skill:
- No usa `curl` directamente.
- No lee, referencia ni imprime ninguna variable de entorno de credenciales: el wrapper es el único lugar del sistema que las toca.
- No escribe nada en disco: todo se lee de la salida estándar del comando.

## Reglas que respeta (AGENTS.md 10a)
- Solo peticiones GET (lo impone el wrapper: solo hace GET, host fijo `breathecode.herokuapp.com`).
- Solo datos de la propia cuenta de estudiante de Caro.
- Ninguna credencial se muestra, copia, imprime ni escribe en mensajes, archivos, `memory/` ni logs — porque esta skill nunca la toca.
- Si el wrapper indica que falta la credencial en el servidor, avisar a Caro y parar; no pedirla ni sugerir mostrarla por chat.
- Si la API responde 401 o 403: parar y avisar a Caro, no reintentar.

## Procedimiento
1. Ejecutar:
   ```sh
   4geeks-get /v1/auth/user/me
   ```
   Esto imprime el JSON de respuesta y, en la última línea, `HTTP_STATUS:<código>`.

2. Interpretar la última línea de la salida:
   - **HTTP_STATUS:200** → sesión activa. Del JSON impreso, tomar solo `first_name` + `last_name`, `roles[].academy.name` y `roles[].role`, y reportar eso a Caro en tono normal. **No incluir `email`** en el reporte aunque esté en el JSON.
   - **HTTP_STATUS:401 o HTTP_STATUS:403** → credencial inválida o expirada. No reintentar. Avisar a Caro con el código exacto y sugerir que revise su acceso a 4Geeks; no pedir ni sugerir mostrar ninguna credencial por chat.
   - **HTTP_STATUS:no_token** → falta la credencial en el servidor. Avisar a Caro que hay que configurarla por SSH (ella ya sabe cómo); no preguntar cuál es ni pedir que la pegue en el chat.
   - **HTTP_STATUS:bad_path** → error interno de esta skill (ruta mal formada). No debería pasar con la ruta fija de arriba; si pasa, reportarlo como bug y no reintentar.
   - **HTTP_STATUS:curl_error** → error de conexión (sin red, DNS, timeout, etc.). Reportarlo como tal, sin reintentar en bucle.
   - Cualquier otro código HTTP → reportar el código y el cuerpo de la respuesta (sin el email, igual que en el caso 200) como error, sin reintentar en bucle (regla 9 de AGENTS.md: si falla dos veces seguidas, parar y contar).

## Notas
- El endpoint `/v1/auth/user/me` es el más liviano para verificar sesión: no expone notas, calificaciones ni nada más allá del perfil básico.
- Esta skill depende de que `/usr/local/bin/4geeks-get` exista en el servidor. No la crea ni la modifica: si no está o cambia de comportamiento, avisar a Caro en vez de intentar recrearla.
- Es la base para futuras skills de solo-lectura sobre la API de BreatheCode (ej. tareas pendientes, asistencia, cohortes): todas deben usar `4geeks-get <ruta>` de la misma forma, nunca `curl` directo ni referencias a credenciales en el texto de la skill.
- No hace falta preguntar a Caro cada vez que se usa esta skill (es lectura interna, reversible), salvo que la regla 9 de AGENTS.md aplique (dos fallos seguidos).
