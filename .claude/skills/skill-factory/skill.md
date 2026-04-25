---
description: Diseña skills híbridos que pueden activarse por descripción (automático) o por comando (manual).
allowed-tools: [bash, edit, read]
---

Skill: Skill Factory (Hybrid Edition)
Al iniciar este proceso, guía al usuario para definir la dualidad del skill:

Entrevista de Definición:
Nombre: ID del skill (kebab-case).
Trigger Manual: "¿Qué comando quieres usar para invocarlo manualmente? (ej: /test-all). Si no quieres comando, escribe 'ninguno'."
Trigger Automático: "¿En qué situaciones debe Claude decidir usar este skill por sí solo? (ej: 'al modificar archivos en /src' o 'antes de responder sobre errores')."
Acción: "¿Qué pasos exactos debe seguir el skill?"

Construcción del Contrato (SKILL.md):
Frontmatter: Si hay comando, inclúyelo como command: /nombre. La description debe ser muy rica en palabras clave para que el motor de IA sepa cuándo activarlo automáticamente.
Instrucciones: Define un flujo claro de "Entrada -> Proceso -> Validación".

Ejecución de Carpeta y Archivo:
Crea .claude/skills/[nombre]/SKILL.md.

Sincronización:
Pregunta al usuario: "¿Quieres que añada este comando a tu lista de comandos rápidos en CLAUDE.md?"
