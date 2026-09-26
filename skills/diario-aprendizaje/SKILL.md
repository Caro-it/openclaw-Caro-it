---
name: diario-aprendizaje
description: Convierte lo que Caro cuenta que aprendió hoy en una entrada de su diario (diario/AAAA-MM.md) y se la confirma por Telegram.
---

# Diario de aprendizaje

Usá esta skill cuando Caro te cuente qué aprendió, en qué se trabó o qué le quedó pendiente (por ejemplo: "hoy aprendí…", "anotá en el diario…").

## Paso 1: Armar la entrada
- Fecha: hoy, en hora de España, con el día de la semana en castellano.
- **Frente:** clasificalo con `USER.md` (4Geeks, DAM, Data Scientist, Prácticas). Si no está claro, poné "General".
- Usá solo lo que Caro dijo. **No agregues aprendizajes, explicaciones ni conclusiones propias.**
- Si Caro no mencionó en qué se trabó o qué le quedó pendiente, omití esa línea: no la inventes.
- Si el texto incluye una key, token o contraseña, no la escribas y avisale (regla 3 de `AGENTS.md`).

Formato:

```
## AAAA-MM-DD (día)
**Frente:** …
**Aprendí:**
- …
**Me trabé con:** …
**Pendiente:** …
```

## Paso 2: Guardar
- Archivo: `diario/AAAA-MM.md` dentro del workspace (creá la carpeta y el archivo si no existen, con el título `# Diario de aprendizaje – <Mes> <AAAA>`).
- La entrada nueva va **arriba del todo**, justo debajo del título.
- Si ya hay una entrada de hoy, agregá los puntos nuevos a esa entrada en lugar de crear otra.
- No hagas commit ni push (regla 13 de `AGENTS.md`).

## Paso 3: Verificar y confirmar
- Volvé a leer el archivo y comprobá que la entrada está guardada.
- Respondé por Telegram con la entrada tal como quedó, precedida de: "Anotado en tu diario 🐙".
- Si no se pudo guardar, decilo con el error. Nunca confirmes sin haber comprobado el archivo.