# openclaw-Caro-it

# Alessia 🐙 — Mi agente de OpenClaw

Agente personal de Caro, configurada para el proyecto **"Mi Agente, A Mi Manera"** (4Geeks Academy).

## Qué hay en este repo
| Archivo | Qué contiene |
|---|---|
| `IDENTITY.md` | Nombre y símbolo de la agente |
| `SOUL.md` | Personalidad, tono (voseo), gestión de la incertidumbre y cuándo actúa o pregunta |
| `AGENTS.md` | Reglas inamovibles: privacidad, cuándo parar y preguntar, límites técnicos |
| `USER.md` | Contexto de Caro: estudios, horarios y zonas horarias |
| `TOOLS.md` | Servicios disponibles, convenciones y formato de los recordatorios |
| `SKILLS_DESIGN.md` | Diseño de las skills (antes de implementarlas) y resultados de las pruebas |
| `skills/recordatorios-clases/` | Skill: recordatorios por Telegram 1 hora antes de cada clase |
| `skills/diario-aprendizaje/` | Skill: diario de aprendizaje estructurado a partir de notas informales |

## Notas para la revisión
- **Modelo del agente:** la key de LiteLLM de 4Geeks caducó el 16/09/2026 y el agente dejaba de responder (HTTP 401). Para poder hacer el proyecto, conecté el modelo con mi suscripción de Claude (setup-token). No es un servicio que usen las skills.
- **Composio (Google Calendar y Docs):** las conexiones del proyecto anterior se habían perdido, y restaurarlas requería un flujo OAuth. Para respetar la regla del proyecto **no las reconecté**: desactivé Composio temporalmente, y las dos skills usan solo Telegram y archivos del workspace.
- **Recordatorios:** los recordatorios existentes estaban creados en sesión `main` y dependían del heartbeat, así que nunca llegaban. Se rehicieron en sesión aislada con entrega a Telegram (ver `SKILLS_DESIGN.md`).
- `openclaw doctor` se ejecuta sin errores. Queda un aviso de seguridad sobre secretos en texto plano en `openclaw.json`, que está fuera de este repo y protegido por la regla 1 de `AGENTS.md`.
- El diario personal (`diario/`) y la memoria de la agente (`memory/`) están excluidos del repo por privacidad.