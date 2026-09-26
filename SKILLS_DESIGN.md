# SKILLS_DESIGN.md

Diseño de las skills personalizadas de Alessia 🐙, escrito antes de implementarlas.

## Contexto y restricciones
- **Servicios disponibles:** Telegram (entrega comprobada) y los archivos del workspace. Composio (Google Calendar y Docs) está desconectado durante esta entrega: sus conexiones se perdieron y restaurarlas implica un flujo OAuth, que el enunciado no permite.
- **Lección previa que condiciona el diseño:** los recordatorios creados en `sesión main` nunca llegaban, porque dependen del heartbeat (apagado). Solo funcionan en **sesión aislada con entrega a Telegram**. Además, Alessia llegó a decir que los había arreglado sin comprobarlo. Por eso las dos skills terminan con una verificación real, no con un "debería funcionar".

---

## Skill 1: `recordatorios-clases`

### 1. ¿Qué hace?
Convierte una lista de clases escrita en lenguaje natural en recordatorios por Telegram que llegan 1 hora antes de cada clase.

### 2. ¿Qué input necesita?
- **Lo que le da Caro:** texto libre por Telegram con asignatura, fecha y hora. Por ejemplo, copiado de la tabla del campus de Linkia: *"DAM - Inglés - Clase 04: martes 09/11/2026, 18:00"*.
- **Lo que ya sabe por la configuración:**
  - `USER.md`: zona de Caro (Europe/Madrid), qué clases van en hora argentina (Data Scientist) y horario fijo de 4Geeks (lunes, miércoles y viernes, 18:30–19:30).
  - `TOOLS.md`: formato de título `[Clase] Asunto`, recordatorios siempre en sesión aislada con entrega a Telegram, horas mostradas en hora de España.
  - `AGENTS.md`: parar y preguntar ante choques, datos que faltan y antes de borrar (reglas 5, 7 y 8).
  - `SOUL.md`: voseo, mensajes cortos, avisar si una fuente se contradice.

**Por qué este input:** es exactamente como le llegan los horarios a Caro (tablas del campus, que ya traían errores). La skill tiene que aceptar el formato real, no uno inventado.

### 3. ¿Cómo es un buen output?
- **Antes de crear nada**, un único mensaje de Telegram con:
  - La lista de recordatorios que va a crear, con las horas en hora de España (y las dos horas si es una clase argentina).
  - Las fechas que no coinciden con su día de la semana, para que Caro elija.
  - Los choques con el horario fijo o con recordatorios existentes.
- **Tras el OK de Caro:** crea los recordatorios con `--session isolated --announce --channel telegram`. Las clases en hora argentina se crean con la zona `America/Argentina/Buenos_Aires`, para que no se desfasen con el cambio de hora español.
- **Verificación:** comprueba con `cron list` que todos quedaron en sesión aislada con entrega a Telegram, y confirma cuántos creó.
- **Cómo sé que funcionó:** el recordatorio llega de verdad a Telegram 1 hora antes. Se prueba forzando uno.

**Por qué este output:** Telegram es donde Caro está todo el día; un recordatorio que no llega no sirve de nada. La confirmación previa existe porque en la primera prueba Alessia borró y recreó 16 tareas sin avisar.

---

## Skill 2: `diario-aprendizaje`

### 1. ¿Qué hace?
Convierte unas pocas líneas sobre lo que Caro aprendió hoy en una entrada estructurada de su diario de aprendizaje, y se la confirma por Telegram.

### 2. ¿Qué input necesita?
- **Lo que le da Caro:** 2 a 5 líneas informales por Telegram. Por ejemplo: *"hoy aprendí que los cron en main dependen del heartbeat, que las zonas horarias hay que ponerlas por nombre y no sumando horas, y me trabé con Composio"*.
- **Lo que ya sabe por la configuración:**
  - `USER.md`: los frentes de Caro (4Geeks, DAM, Data Scientist, prácticas), para clasificar cada aprendizaje.
  - `SOUL.md`: voseo, directa, no inventar. Si Caro no dice algo, no se rellena.
  - `AGENTS.md`: no guardar secretos. Si aparece una key en el texto, no la escribe en el diario.

**Por qué este input:** tiene que costarle menos de un minuto al final del día. Si le pidiera un formato, no lo usaría.

### 3. ¿Cómo es un buen output?
- **Destino:** el archivo `diario/AAAA-MM.md` del workspace, con la entrada nueva arriba del todo. Ejemplo:

~~~markdown
## 2026-09-26 (sábado)
**Frente:** 4Geeks – proyecto OpenClaw
**Aprendí:**
- Los cron en sesión main dependen del heartbeat.
- Las zonas horarias se configuran por nombre (America/Argentina/Buenos_Aires), no sumando horas.
**Me trabé con:** Composio (conexiones perdidas).
**Pendiente:** reconectar Composio después de la entrega.
~~~

- **Por Telegram:** un mensaje corto con la entrada tal como quedó guardada.
- **Cómo sé que funcionó:** el archivo existe y tiene la entrada, y el mensaje de Telegram coincide con lo guardado. No aparece nada que Caro no haya dicho.

**Por qué este output:** un archivo Markdown por mes es fácil de leer y de pasar a Google Docs cuando se reconecte Composio. La sección "Me trabé con" existe porque para Caro lo que no le salió es tan útil de recordar como lo que aprendió.

---

## Resultados de las pruebas y ajustes

### recordatorios-clases (input real: horarios del campus de Linkia)
- ✅ Detectó las dos fechas del campus cuyo día de la semana no coincidía (Inglés 04 y 05) y no eligió por su cuenta.
- ✅ Detectó el choque con 4Geeks y uno que no estaba previsto: IPEI II coincide con el horario de prácticas (sale de `USER.md`).
- ✅ Esperó confirmación antes de crear nada.
- ❌ Primer intento de cálculo de hora fallido (usó `date` con una sintaxis inválida). Se resolvió pasando la hora en formato ISO con desfase.
- ⚠️ No marcó el posible choque de Inglés 04 (18:00) con 4Geeks (18:30), porque no conocía la duración de la clase.
- Verificado en `cron list`: 4 recordatorios en sesión aislada con entrega a Telegram.

### diario-aprendizaje
- ✅ La entrada guardada contiene solo lo que dijo Caro, con el formato definido.
- ❌ Confirmó "Anotado" antes de verificar el archivo. **Ajuste:** el Paso 3 ahora obliga a leer el archivo antes de confirmar.
- ⚠️ Clasificó el frente como "General" porque `USER.md` no decía que el proyecto OpenClaw es de 4Geeks. **Ajuste:** añadido a `USER.md`.

### Patrón detectado
En varias ocasiones Alessia afirmó haber terminado algo sin comprobarlo (también al rehacer los 16 recordatorios, que seguían mal configurados). Por eso ambas skills terminan con un paso de verificación explícito y obligatorio.