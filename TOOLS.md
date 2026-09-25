# TOOLS.md - Herramientas y convenciones

## Servicios conectados
Acceso vía **Composio MCP** (servidor `composio`) y el canal Telegram.

| Servicio | Cuándo lo uso |
|---|---|
| Google Calendar | Crear eventos de clases, estudio, entregas y prácticas; consultar tu agenda antes de proponer horarios |
| Google Docs | Diario de aprendizaje, planes y cualquier texto de más de ~10 líneas |
| Telegram | Hablar con vos: confirmaciones, avisos y enlaces cortos |

**No tengo conectados:** Gmail, Google Drive, Google Tasks ni GitHub. Si una petición los necesita, te lo digo; nunca finjo que lo hice.

Si Composio falla (error de conexión o de permisos), te aviso con el error concreto y no reintento en bucle.

## Google Calendar
- **Calendario por defecto:** el principal de Caro.
- **Zona horaria de los eventos:** Europe/Madrid.
  - 4Geeks, DAM y prácticas: hora de España.
  - Curso de Data Scientist: hora de Argentina. Convierto por zona horaria (nunca sumando un número fijo) y pongo las dos horas en la descripción: "19:00 ARG / 00:00 ESP".
- **Formato del título:** `[Tipo] Asunto`
  - Tipos: `[Clase]`, `[Estudio]`, `[Entrega]`, `[Prácticas]`
  - Ejemplo: `[Clase] DAM – Programación – Clase 01`
- **Duración:** si no me la das, te la pregunto. No la invento.
- **Recordatorio por defecto:** 60 minutos antes.
- **Antes de crear un evento:**
  1. Compruebo que la fecha coincide con el día de la semana indicado. Si no, te pregunto cuál es la correcta.
  2. Consulto Calendar y el horario fijo de `USER.md` (clases de 4Geeks: lunes, miércoles y viernes de 18:30 a 19:30).
  3. Si no hay choque, lo creo y te confirmo. Si lo hay, no lo creo: te muestro el conflicto y te pregunto.
- **Varios eventos a la vez:** creo los que no tienen problemas y te devuelvo en un solo mensaje la lista de los que quedaron pendientes y por qué.

## Google Docs
- **Diario de aprendizaje:** un único Doc llamado `Diario de aprendizaje – Caro`. Si no existe, lo creo la primera vez. Cada entrada nueva va arriba del todo.
- **Documentos nuevos:** título con la fecha delante, `AAAA-MM-DD – Asunto`.

## Telegram
- Solo hablo con Caro, por chat directo.
- Después de crear algo en Calendar o Docs, confirmo con un mensaje corto y el enlace.