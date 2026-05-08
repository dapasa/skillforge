---
name: tech-debt
description: >
  Registra un ítem de deuda técnica en Engram (persistencia entre sesiones) y en Notion
  (página "Deuda Técnica" del proyecto correspondiente). Pregunta al usuario a qué proyecto
  de Notion asociar el ítem antes de guardarlo.
  Trigger: Cuando el usuario dice "agregar deuda técnica", "registrar deuda", "add tech debt",
  "guardar como deuda técnica", o cualquier variante.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- El usuario quiere registrar algo como deuda técnica
- El usuario menciona algo subóptimo, un workaround, o algo pendiente de refactorizar
- El usuario dice "esto quedó como deuda" o similar

---

## Critical Rules

1. **SIEMPRE guardar en Engram primero**, luego en Notion
2. **SIEMPRE preguntar el proyecto Notion** — el usuario tiene varios, nunca asumir
3. **NUNCA inventar el ID de la página Deuda Técnica** — buscarlo via MCP antes de escribir
4. **El checkbox de Notion** debe estar sin completar `- [ ]` — es una deuda pendiente
5. **Esperar respuesta del usuario** antes de continuar cuando se hace una pregunta

---

## Workflow Paso a Paso

### Paso 1 — Recopilar información

Si el usuario ya proveyó título y descripción, usarlos directamente.
Si no, pedir:
- **Título corto** (ej: "Falta validación en módulo de pagos")
- **Descripción** (qué es, por qué quedó así, dónde está afectado)

### Paso 2 — Guardar en Engram

Llamar `mem_save` con:
```
title: "DEUDA: {título}"
type: "decision"
project: "{proyecto-actual}"
topic_key: "deuda/{slug-del-titulo}"
content:
  **What**: {descripción del problema}
  **Why**: {por qué quedó así / qué lo originó}
  **Where**: {archivos o módulos afectados, si se conocen}
  **Learned**: Pendiente de implementar. {notas adicionales si las hay}
```

El slug se genera desde el título: minúsculas, espacios→guiones, sin caracteres especiales.
Ejemplo: "Falta validación en pagos" → `deuda/falta-validacion-en-pagos`

### Paso 3 — Buscar proyectos en Notion

Llamar `notion-search` con query `""` (o query genérica tipo "project") para listar páginas raíz disponibles.

Mostrar al usuario la lista de proyectos encontrados y preguntar:
> "¿A qué proyecto de Notion asociamos esta deuda?"

**STOP — esperar respuesta antes de continuar.**

### Paso 4 — Encontrar la página "Deuda Técnica"

Una vez que el usuario elige el proyecto, hacer `notion-fetch` de esa página raíz para obtener sus sub-páginas. Buscar la que se llame "Deuda Técnica" (o similar: "Tech Debt", "Deudas").

Si no existe la página "Deuda Técnica" bajo ese proyecto: avisar al usuario y preguntar si quiere crearla.

### Paso 5 — Agregar el ítem en Notion

Llamar `notion-update-page` o `notion-create-pages` para agregar el ítem en la página Deuda Técnica como:

```markdown
- [ ] **{Título}** — {Descripción breve}
```

Si la página tiene secciones o categorías, agregar al final sin alterar la estructura existente.

---

## Formato del ítem en Notion

```
- [ ] **{Título corto}** — {Una oración describiendo qué es y dónde afecta}
```

Ejemplos:
```
- [ ] **Validación faltante en módulo de pagos** — No se valida el monto mínimo en el server action `createPayment`
- [ ] **Auditoría UX/UI completa** — Revisar consistencia visual de todo el sitio con Claude Design
```

---

## Búsqueda de proyectos Notion — Guía

Para listar los proyectos disponibles:
```
notion-search(query: "AgentsPayouts OR smtmlab OR proyecto", filters: {})
```

Mostrar los resultados como lista numerada para que el usuario elija fácilmente:
```
¿A qué proyecto de Notion lo asociamos?
1. AgentsPayouts
2. Otro Proyecto
3. ...
```

---

## Decision Tree

```
¿El usuario dio título y descripción?
  Sí → Paso 2 (Engram)
  No → Pedir título y descripción → Paso 2

¿Existe página "Deuda Técnica" en el proyecto elegido?
  Sí → Agregar ítem
  No → Avisar + preguntar si crear la página
```

---

## Notas de Implementación

- Los proyectos Notion raíz suelen ser páginas top-level — buscar con `notion-search` genérico
- La página "Deuda Técnica" puede estar anidada dentro de una database (como "Projects") — hacer `notion-fetch` del proyecto raíz para navegar su estructura
- Usar `notion-update-page` para append de contenido si la herramienta lo soporta; si no, usar `notion-create-pages` para crear un bloque hijo
- El `topic_key` en Engram usa `deuda/` como prefijo fijo — esto permite buscar todas las deudas con `mem_search(query: "deuda/")`
