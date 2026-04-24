---
name: security-review-nextjs-supabase
description: "Ejecuta un security review completo de Next.js + Supabase. Usar cuando se mencione: security review, OWASP, RLS, vulnerabilidades, pentest, impersonation, path traversal, secrets expuestos, middleware bypass, revisar si la app es segura, chequear permisos de Supabase, validar autenticacion, revisar variables de entorno expuestas."
---

# Security Review — Next.js + Supabase

## Objetivo
Realizar un security review exhaustivo de aplicaciones Next.js que consumen Supabase directamente
desde el cliente, con Supabase Auth. Producir: (1) reporte de hallazgos con severidades y 
remediation steps, y (2) checklist ejecutable paso a paso.

---

## Fase 0: Orientación inicial

Antes de comenzar, ejecutar:

```bash
# Mapear estructura del proyecto
find . -type f -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" | \
  grep -v node_modules | grep -v .next | grep -v .git | head -100

# Detectar archivos críticos de configuración
ls -la .env* 2>/dev/null
find . -name "middleware.ts" -o -name "middleware.js" | grep -v node_modules
find . -path "*/app/api/**" -o -path "*/pages/api/**" | grep -v node_modules
find . -name "supabase.ts" -o -name "supabase.js" -o -name "client.ts" | grep -v node_modules
```

Identificar y anotar:
- ¿Usa `app/` router o `pages/` router?
- ¿Existe `middleware.ts`?
- ¿Hay Server Actions (`"use server"`)?
- ¿Dónde se inicializa el cliente Supabase?

---

## Fase 1: Análisis de Secrets y Variables de Entorno

> Referencia detallada: `references/secrets-env.md`

### Checks automáticos:

```bash
# 1. Buscar claves Supabase hardcodeadas
grep -rn "supabase.co" . --include="*.ts" --include="*.tsx" --include="*.js" \
  --exclude-dir={node_modules,.next,.git}

# 2. Detectar service_role key expuesta en cliente
grep -rn "service_role\|SERVICE_ROLE" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# 3. Variables NEXT_PUBLIC_ que no deberían ser públicas
grep -rn "NEXT_PUBLIC_SUPABASE" . --include="*.env*" --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# 4. .env* en .gitignore
cat .gitignore 2>/dev/null | grep -E "\.env"

# 5. Secrets en código fuente (JWT secrets, API keys genéricas)
grep -rn "anon\|anonKey\|apiKey\|secret\|password\|SUPABASE_KEY" . \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git} | grep -v "//.*comment"
```

### Qué verificar manualmente:
- [ ] `NEXT_PUBLIC_SUPABASE_URL` → OK (es público por diseño)
- [ ] `NEXT_PUBLIC_SUPABASE_ANON_KEY` → OK solo si RLS está bien configurado
- [ ] `SUPABASE_SERVICE_ROLE_KEY` → NUNCA debe aparecer en cliente ni en NEXT_PUBLIC_*
- [ ] `.env.local` en `.gitignore`
- [ ] No hay secrets en `next.config.js` bajo `env:` expuestos al cliente

---

## Fase 2: Supabase RLS (Row Level Security)

> Referencia detallada: `references/supabase-rls.md`

### Checks automáticos:

```bash
# Buscar queries directas sin filtro de usuario
grep -rn "\.from(" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git} | grep -v "auth\."

# Buscar uso de supabase client en componentes de cliente
grep -rn "createClient\|createBrowserClient" . --include="*.tsx" --include="*.ts" \
  --exclude-dir={node_modules,.next,.git}

# Buscar bypasses: service role usado en cliente
grep -rn "service_role\|createServerClient" . --include="*.tsx" --include="*.ts" \
  --exclude-dir={node_modules,.next,.git}

# Detectar queries a tablas sin .select() filtrado
grep -rn "supabase\.from\|sb\.from" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```

### Qué verificar manualmente:
- [ ] Cada tabla con datos de usuario tiene RLS habilitado
- [ ] Las políticas usan `auth.uid()` y no `user_id` sin verificar
- [ ] No hay políticas `USING (true)` en tablas sensibles
- [ ] El rol `anon` solo accede a datos verdaderamente públicos
- [ ] Storage buckets con policies correctas (no `(bucket_id = 'x')` sin user check)
- [ ] Las policies de INSERT también verifican `auth.uid()` (no solo SELECT)

---

## Fase 3: Autenticación e Impersonation

> Referencia detallada: `references/auth-impersonation.md`

### Checks automáticos:

```bash
# Buscar uso de getUser vs getSession
grep -rn "getSession\|getUser\|auth\.user" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# Buscar lectura de user desde localStorage o cookies sin validación
grep -rn "localStorage\|sessionStorage" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# Detectar roles manejados solo en cliente
grep -rn "role\|isAdmin\|isSuperuser\|user\.role" . --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# Buscar JWT decode manual (peligroso)
grep -rn "jwt\.decode\|atob.*payload\|base64" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```

### Qué verificar manualmente:
- [ ] Se usa `supabase.auth.getUser()` (valida contra servidor) y NO solo `getSession()` (puede ser stale)
- [ ] Los roles de usuario NO se guardan solo en metadata sin validar en RLS
- [ ] No hay lógica de autorización basada solo en claims del JWT sin verificación server-side
- [ ] No es posible pasar `user_id` arbitrario en request body/params para acceder a datos de otro usuario
- [ ] El logout invalida la sesión correctamente (no solo borra cookie local)

---

## Fase 4: Next.js Middleware

> Referencia detallada: `references/middleware-nextjs.md`

### Checks automáticos:

```bash
# Leer el middleware completo
cat middleware.ts 2>/dev/null || cat middleware.js 2>/dev/null

# Ver el matcher config
grep -A 20 "matcher\|config" middleware.ts 2>/dev/null

# Buscar rutas que podrían saltear el middleware
grep -rn "matcher" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```

### Qué verificar manualmente:
- [ ] El `matcher` en `middleware.ts` cubre TODAS las rutas protegidas
- [ ] No hay rutas protegidas accesibles via `/_next/` o `/api/` que el matcher omita
- [ ] El middleware revalida la sesión contra Supabase (no solo lee cookie)
- [ ] Hay redirect correcto a login cuando no hay sesión válida
- [ ] Las rutas públicas están explícitamente excluidas (no por defecto)
- [ ] No es posible bypassear el middleware con variaciones de path (`/dashboard` vs `/Dashboard` vs `/dashboard/`)

---

## Fase 5: API Routes y Server Actions

### Checks automáticos:

```bash
# Listar todos los API routes
find . -path "*/app/api/**/*.ts" -o -path "*/pages/api/**/*.ts" | grep -v node_modules

# Listar Server Actions
grep -rn '"use server"' . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# Buscar API routes sin auth check
grep -rn "export.*GET\|export.*POST\|export.*PUT\|export.*DELETE" . \
  --include="*.ts" --exclude-dir={node_modules,.next,.git} | \
  grep -v "layout\|page\|component"

# Buscar validación de input
grep -rn "zod\|yup\|joi\|validate\|schema" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```

### Qué verificar manualmente:
- [ ] Cada API route verifica sesión válida antes de procesar
- [ ] Las Server Actions validan que el usuario tiene permisos sobre el recurso solicitado
- [ ] Hay validación de input (schema validation con Zod o similar)
- [ ] No se confía en `req.body.userId` para autorizar — se usa el user de la sesión autenticada
- [ ] CSRF: Server Actions de Next.js tienen protección built-in, pero validar si hay API routes que no
- [ ] Rate limiting en rutas sensibles (login, signup, password reset)

---

## Fase 6: OWASP Top 10 — Checks Específicos Next.js/Supabase

> Referencia detallada con ejemplos de código: `references/owasp-top10.md`

### A01 - Broken Access Control

```bash
grep -rn "params\.\|searchParams\.\|req\.query" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```
- [ ] Los IDs en URL params no permiten acceder a recursos de otros usuarios (IDOR)
- [ ] No hay endpoints que devuelvan listados completos sin filtro por usuario

### A02 - Cryptographic Failures

- [ ] HTTPS enforced (no mixed content)
- [ ] No hay datos sensibles en localStorage sin cifrar
- [ ] Passwords nunca se almacenan (Supabase Auth lo maneja, pero verificar custom auth flows)

### A03 - Injection

```bash
grep -rn "\.rpc(\|\.query(\|raw\|textSearch" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}
```
- [ ] No hay construcción dinámica de queries con string interpolation
- [ ] Los `textSearch` y `filter` sanitizan input del usuario

### A05 - Security Misconfiguration

```bash
cat next.config.js 2>/dev/null || cat next.config.ts 2>/dev/null
grep -rn "headers\|cors\|csp\|x-frame" . --include="*.ts" --include="*.js" \
  --exclude-dir={node_modules,.next,.git}
```
- [ ] Security headers configurados en `next.config.js` (CSP, X-Frame-Options, HSTS)
- [ ] CORS configurado correctamente en API routes
- [ ] `dangerouslyAllowBrowser: true` en Supabase client solo cuando corresponde

### A07 - Identification and Authentication Failures

- [ ] No hay "remember me" persistente sin refresh token rotation
- [ ] Email confirmation habilitado en Supabase Auth
- [ ] No hay magic links con expiración muy larga

### A09 - Security Logging and Monitoring

```bash
grep -rn "console\.log\|console\.error" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git} | head -30
```
- [ ] No se loguea PII o tokens en `console.log`
- [ ] Los errores de auth no revelan si el usuario existe o no (user enumeration)

---

## Fase 7: Path Traversal y Open Redirects

```bash
# Path traversal en file serving
grep -rn "readFile\|createReadStream\|path\.join\|path\.resolve" . \
  --include="*.ts" --include="*.tsx" --exclude-dir={node_modules,.next,.git}

# Open redirects
grep -rn "redirect\|router\.push\|window\.location" . \
  --include="*.ts" --include="*.tsx" --exclude-dir={node_modules,.next,.git}

# Query params usados en redirects
grep -rn "returnUrl\|redirectTo\|callbackUrl\|next=" . \
  --include="*.ts" --include="*.tsx" --exclude-dir={node_modules,.next,.git}
```

- [ ] Los redirects post-login validan que la URL destino es del mismo dominio
- [ ] No hay file serving dinámico con path del usuario sin sanitizar
- [ ] Los `callbackUrl` / `redirectTo` de auth están allowlisted

---

## Fase 8: Generación del Reporte

Al finalizar todos los checks, generar el reporte usando esta estructura:

```markdown
# Security Review Report — [Nombre del Proyecto]
**Fecha:** [fecha]  
**Stack:** Next.js [versión] + Supabase  
**Revisor:** Claude Code (Security Review Skill)

## Executive Summary
[2-3 líneas con el estado general de seguridad]

## Hallazgos

### 🔴 CRITICAL
| ID | Descripción | Archivo | Remediación |
|----|-------------|---------|-------------|
| C-01 | ... | ... | ... |

### 🟠 HIGH
| ID | Descripción | Archivo | Remediación |
|----|-------------|---------|-------------|

### 🟡 MEDIUM
| ID | Descripción | Archivo | Remediación |
|----|-------------|---------|-------------|

### 🟢 LOW / INFO
| ID | Descripción | Archivo | Remediación |
|----|-------------|---------|-------------|

## Detalle de Hallazgos
[Para cada hallazgo: descripción, PoC si aplica, código vulnerable, código corregido]

## Checklist de Remediación
[Ver sección siguiente]

## Métricas
- Total hallazgos: X
- Critical: X | High: X | Medium: X | Low: X
- Cobertura: X de Y áreas revisadas
```

---

## Fase 9: Checklist de Remediación (Ejecutable)

Generar un checklist ordenado por prioridad con pasos específicos:

```markdown
# Checklist de Remediación — [Proyecto]

## Prioridad 1 — CRITICAL (resolver antes de producción)

### [ ] C-01: [Nombre del hallazgo]
**Archivo:** `src/lib/supabase.ts`  
**Tiempo estimado:** 30 min

**Paso 1:** [acción concreta]
**Paso 2:** [acción concreta]  
**Verificación:** [cómo confirmar que está resuelto]

---
[repetir por cada hallazgo]

## Prioridad 2 — HIGH
...

## Prioridad 3 — MEDIUM / LOW
...
```

---

## Fase 10 (Opcional): Crear GitHub Issues

Si el usuario lo confirma, crear un issue por cada hallazgo CRITICAL y HIGH:

```bash
# Verificar gh CLI disponible
gh auth status

# Crear issue por hallazgo
gh issue create \
  --title "[SECURITY][CRITICAL] C-01: Descripción" \
  --body "$(cat issue-c01.md)" \
  --label "security,critical" \
  --assignee "@me"
```

**IMPORTANTE:** Preguntar al usuario antes de crear issues — pueden contener información sensible.

---

## Fase 11: Persistencia en Engram (si está disponible)

Al finalizar el reporte y el checklist, verificar si Engram está disponible y grabar el contexto
del review para que sea accesible en futuras sesiones.

### Paso 1: Verificar disponibilidad de Engram

```bash
engram --version 2>/dev/null && echo "ENGRAM_AVAILABLE" || echo "ENGRAM_NOT_FOUND"
```

Si el comando no existe, **omitir esta fase silenciosamente** sin avisar al usuario a menos que
lo haya pedido explícitamente.

### Paso 2: Preparar el contexto a grabar

Construir un resumen estructurado con:
- Nombre del proyecto y fecha del review
- Stack exacto (versión Next.js, versión Supabase client)
- Resumen de hallazgos por severidad (conteos)
- Lista de hallazgos CRITICAL y HIGH con su ID y descripción corta
- Estado de áreas clave: RLS ✅/❌, Middleware ✅/❌, Secrets ✅/❌, Auth ✅/❌
- Remediaciones pendientes (IDs sin resolver)
- Próximos pasos recomendados

### Paso 3: Grabar en Engram

```bash
# Grabar resumen ejecutivo del review
engram add "security-review/[nombre-proyecto]/[YYYY-MM-DD]" \
  --content "$(cat <<'EOF'
## Security Review — [Nombre Proyecto]
**Fecha:** [fecha]
**Stack:** Next.js [v] + Supabase

### Estado General: [PASS/FAIL/NEEDS WORK]

### Hallazgos
- Critical: X | High: X | Medium: X | Low: X

### Áreas
- RLS: [✅ OK / ❌ Issues / ⚠️ Revisar]
- Middleware: [✅ OK / ❌ Issues / ⚠️ Revisar]
- Secrets: [✅ OK / ❌ Issues / ⚠️ Revisar]
- Auth/Impersonation: [✅ OK / ❌ Issues / ⚠️ Revisar]
- API Routes: [✅ OK / ❌ Issues / ⚠️ Revisar]

### Hallazgos Críticos y Altos
[listar IDs con descripción corta]

### Remediaciones Pendientes
[IDs del checklist sin resolver aún]

### Próximos Pasos
[acciones concretas recomendadas]
EOF
)"
```

```bash
# Grabar también el checklist completo para referencia futura
engram add "security-review/[nombre-proyecto]/[YYYY-MM-DD]/checklist" \
  --content "$(cat security-review-checklist.md)"
```

### Paso 4: Confirmar al usuario

Informar brevemente:
```
✅ Review guardado en Engram bajo: security-review/[nombre-proyecto]/[fecha]
   Incluye: resumen ejecutivo + checklist completo
   Accesible en próximas sesiones de Claude Code.
```

### Notas sobre Engram

- **No grabar código fuente vulnerable** — solo descripciones y referencias a archivos
- **No grabar valores de secrets encontrados** — solo indicar que existen y en qué archivo
- La key de Engram es jerárquica: permite recuperar por proyecto o listar todos los reviews
- Si el usuario pide "mostrar reviews anteriores", usar `engram get "security-review/"` con wildcard

---

## Severidad — Criterios de Clasificación

| Nivel | Criterio |
|-------|----------|
| 🔴 CRITICAL | Acceso no autorizado a datos, RLS bypass, service_role key expuesta, impersonation posible |
| 🟠 HIGH | Auth bypass en middleware, IDOR, secrets en repo, open redirect explotable |
| 🟡 MEDIUM | Falta de rate limiting, headers de seguridad ausentes, user enumeration |
| 🟢 LOW | console.log con info no crítica, mejoras de hardening, best practices |

---

## Notas de Ejecución

1. Ejecutar las fases **en orden** — la Fase 0 orienta todo lo demás
2. Documentar cada hallazgo **a medida que se encuentra**, no al final
3. Si hay Server Actions, leerlas todas — son un vector frecuentemente ignorado
4. El cliente Supabase en browser tiene el `anon` key — toda la seguridad real descansa en RLS
5. Ante la duda sobre una política RLS: asumir que es insegura hasta demostrar lo contrario