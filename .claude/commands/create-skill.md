---
description: Diseña skills híbridos que pueden activarse por descripción (automático) o por comando (manual). Úsalo cuando quieras crear una nueva capacidad automatizada, un flujo repetitivo, o un comando personalizado para Claude Code.
allowed-tools: [bash, edit, read, write]
---

# Skill Factory (Hybrid Edition)

Al iniciar este proceso, guía al usuario para definir la dualidad del skill siguiendo estos pasos:

## Entrevista de Definición

Haz estas preguntas una por una y espera la respuesta antes de continuar:

1. **Nombre**: "¿Cuál será el ID del skill? (usa kebab-case, ej: `test-runner`, `module-generator`)"
2. **Trigger Manual**: "¿Qué comando quieres usar para invocarlo manualmente? (ej: `/test-all`). Si no quieres comando manual, escribe 'ninguno'."
3. **Trigger Automático**: "¿En qué situaciones debe Claude activar este skill por sí solo? (ej: 'al modificar archivos en /src', 'antes de responder sobre errores', 'nunca')."
4. **Acción**: "¿Qué pasos exactos debe seguir el skill? Descríbelos en orden."

## Construcción del Contrato

Con las respuestas del usuario, construye el archivo `SKILL.md`:

- **Frontmatter**: Si hay comando manual, incluye `command: /nombre`. La `description` debe ser rica en palabras clave para activación automática por contexto.
- **Instrucciones**: Define un flujo claro con secciones **Entrada → Proceso → Validación**.

## Ejecución

1. Crea el archivo en `.claude/commands/[nombre].md` (para comandos invocables con `/nombre`).
2. Si el usuario también quiere que el skill esté en `.claude/skills/[nombre]/skill.md`, créalo como copia de referencia.

## Sincronización

Pregunta al usuario: "¿Quieres que añada este comando a tu lista en `CLAUDE.md`?"

Si acepta, edita el `CLAUDE.md` del proyecto y agrega el nuevo comando bajo la sección **Custom Commands** en **Claude Agent Configuration**.
