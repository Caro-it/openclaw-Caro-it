---
name: recordatorios-clases
description: Crea recordatorios por Telegram 1 hora antes de las clases que Caro pasa en texto libre (tablas del campus, listas).
---

# Recordatorios de clases

Usá esta skill cuando Caro te pase una o varias clases (asignatura, fecha y hora) y quiera que le avises antes.

## Paso 1: Interpretar
- Leé cada clase: asignatura, número de clase, fecha, hora y si se repite.
- Zona horaria según `USER.md`:
  - Data Scientist → `America/Argentina/Buenos_Aires`.
  - 4Geeks, DAM, Taller y prácticas → `Europe/Madrid`.
  - Si no está claro, preguntá antes de seguir.
- Si falta la fecha o la hora de alguna clase, marcala como [PENDIENTE] y no la crees.

## Paso 2: Comprobar
1. **Fecha vs. día de la semana:** si no coinciden, no elijas vos: marcalo para que Caro decida.
2. **Choques:** compará con el horario fijo de `USER.md` y con los recordatorios que ya existen (lista de tareas cron).
3. **Duplicados:** si ya hay un recordatorio para esa misma clase, no lo dupliques.

## Paso 3: Mostrar el plan y esperar
Mandá UN solo mensaje con:
- La lista de recordatorios a crear, con la hora del aviso en hora de España (y en las clases argentinas, las dos: "18:00 ARG / 23:00 ESP").
- Las fechas dudosas y los choques, cada uno con su motivo.
- La pregunta: "¿Creo estos?"

**No crees nada hasta que Caro responda que sí.** Esta pausa es obligatoria (regla 5 de `AGENTS.md`).

## Paso 4: Crear
Para cada clase aprobada, creá una tarea cron con:
- Nombre: `[Clase] <Asunto> – 1h antes`
- Horario: 1 hora antes del inicio, en la zona horaria de la clase.
  - Clase única → tarea de una sola vez que se borra al ejecutarse.
  - Clase semanal → expresión cron con la zona horaria de la clase.
- Sesión: **aislada** (`isolated`). Nunca `main`: depende del heartbeat, que está apagado.
- Entrega: **announce → telegram → `1269748514`**.
- Mensaje de la tarea: `Respondé solo con este texto: ⏰ En 1 hora (HH:MM ESP): [Clase] <Asunto>. ¡Preparate! 🐙`

## Paso 5: Verificar antes de confirmar
- Revisá la lista de tareas cron y comprobá que cada recordatorio nuevo existe, está en sesión aislada y tiene la entrega a Telegram.
- Solo después respondé: "Creé X recordatorios. Verificado: sesión aislada + Telegram." Si alguno falló, decilo con el motivo.
- **Nunca digas que algo está creado sin haberlo comprobado en la lista.**