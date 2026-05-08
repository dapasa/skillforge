---
name: tech-debt-sync
description: >
  Consulta, sincroniza y resuelve deuda técnica entre Engram y Notion.
  Busca todos los ítems en ambos backends, detecta desincronías y las corrige.
  Cuando un ítem se resuelve, lo marca como finalizado en los dos lados.
  Trigger: Cuando el usuario pregunta por la deuda técnica ("qué deudas tenemos",
  "mostrame las deudas", "deuda técnica", "tech debt status", "resolver deuda X",
  "marcar como resuelta", "cuántas deudas quedan").
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- El usuario pregunta qué deuda técnica hay en el proyecto
- El usuario quiere ver el estado de las deudas
- El usuario quiere marcar una deuda como resuelta
- El usuario quiere sincronizar Engram y Notion

---

## Critical Rules

1. **SIEMPRE buscar en Engram primero**, luego en Notion
2. **SIEMPRE preguntar el proyecto Notion** — el usuario tiene varios, nunca asumir
3. **Sincronizar automáticamente** — si un ítem está en un lado y no en el otro, agregarlo sin preguntar
4. **Para resolver**: marcar `- [x]` en Notion Y actualizar el estado en Engram en el mismo paso
5. **STOP y esperar respuesta** después de preguntar el proyecto Notion
6. **NUNCA inventar IDs** de páginas — siempre buscarlos via MCP

---

## Workflow: Consultar y Sincronizar

### Paso 1 — Leer deudas de Engram

Buscar todas las deudas con:
```
mem_search(query: "DEUDA", project: "{proyecto-actual}")
```

Recopilar todos los resultados cuyo título empiece con `DEUDA:`.
Para cada resultado, leer el contenido completo con `mem_get_observation(id)` para extraer:
- Título limpio (sin "DEUDA: " prefix)
- Estado (buscar si dice "Resuelta" o "DONE" en el contenido)
- topic_key

### Paso 2 — Preguntar proyecto Notion

Buscar proyectos disponibles:
```
notion-search(query: "proyecto", filters: {})
```

Mostrar lista numerada y preguntar:
> "¿De qué proyecto querés ver las deudas?"

**STOP — esperar respuesta.**

### Paso 3 — Leer deudas de Notion

Una vez elegido el proyecto:
1. `notion-fetch(id: {página-raíz-del-proyecto})` → navegar y encontrar la página "Deuda Técnica"
2. `notion-fetch(id: {página-deuda-técnica})` → leer todos los ítems

Parsear los ítems:
- `- [ ] **Título** — descripción` → pendiente
- `- [x] **Título** — descripción` → resuelta

### Paso 4 — Detectar y corregir desincronías

Comparar listas por título (fuzzy match — ignorar mayúsculas/acentos menores):

| Situación | Acción |
|-----------|--------|
| En Engram, no en Notion | Agregar a Notion como `- [ ]` |
| En Notion, no en Engram | Agregar a Engram con `mem_save` |
| En ambos, sin conflicto | OK |
| Resuelta en un lado, pendiente en el otro | Sincronizar el estado resuelto |

Ejecutar las correcciones automáticamente, sin preguntar.

### Paso 5 — Mostrar resultado

Mostrar al usuario la lista consolidada:

```
📋 Deuda Técnica — {Proyecto} ({fecha})

Pendientes (N):
  1. Auditoría UX/UI completa del sitio con Claude Design
  2. Feature invitación agencia + datos facturación US
  ...

Resueltas (N):
  ✓ {título}
  ...

Sincronización: {N ítems agregados a Notion / N ítems agregados a Engram / Todo en sync}
```

---

## Workflow: Resolver una Deuda

Trigger: usuario dice "resolver deuda X", "marcar como resuelta X", "terminamos con X"

### Paso 1 — Identificar el ítem

Si el usuario mencionó el número o título, usarlo directamente.
Si hay ambigüedad, mostrar la lista y pedir que elija.

### Paso 2 — Marcar en Notion

Usar `notion-update-page` con `update_content`:
```
old_str: "- [ ] **{Título}**"
new_str: "- [x] **{Título}**"
```

### Paso 3 — Marcar en Engram

Usar `mem_save` con el mismo `topic_key` de la deuda para hacer upsert:
- Agregar al inicio del contenido: `**Estado**: RESUELTA ✓ ({fecha})\n\n`
- Mantener el resto del contenido intacto

### Paso 4 — Confirmar al usuario

```
✓ Deuda "{Título}" marcada como resuelta en Engram y Notion.
```

---

## Formato de ítems en Notion

```
- [ ] **{Título}** — {descripción}    ← pendiente
- [x] **{Título}** — {descripción}    ← resuelta
```

---

## Mapeo Engram ↔ Notion

| Campo | Engram | Notion |
|-------|--------|--------|
| Identificador | `topic_key: deuda/{slug}` | Checkbox con título en negrita |
| Estado pendiente | sin "RESUELTA" en contenido | `- [ ]` |
| Estado resuelto | "**Estado**: RESUELTA ✓" en contenido | `- [x]` |
| Descripción | campo `**What**` | texto después del `—` |

---

## Decision Tree

```
¿El usuario quiere VER las deudas?
  → Workflow Consultar y Sincronizar

¿El usuario quiere RESOLVER una deuda?
  → Workflow Resolver una Deuda

¿El usuario dice "deuda técnica" sin más contexto?
  → Workflow Consultar y Sincronizar (default)
```

---

## Notas de Implementación

- Usar `mem_search(query: "DEUDA", project: "{proyecto}")` para buscar en Engram — el prefijo `DEUDA:` fue establecido por el skill `tech-debt`
- La comparación de títulos debe ser tolerante: "Auditoría UX/UI" y "Auditoría UX/UI completa" son probablemente el mismo ítem → mostrar al usuario si hay duda
- Si la página "Deuda Técnica" no existe en el proyecto Notion elegido, avisar al usuario
- La fecha en el reporte de resolución es la fecha actual del sistema
