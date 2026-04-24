# OWASP Top 10 — Next.js + Supabase

Referencia combinada: base genérica OWASP Top 10 (2021) con vectores específicos del stack.

---

## A01 — Broken Access Control

**Qué es:** Usuarios acceden a recursos o acciones que no les corresponden.

**En Next.js + Supabase:**

```typescript
// ❌ IDOR — el ID viene de la URL, cualquiera puede cambiarlo
// GET /api/invoices/[id]
export async function GET(req: Request, { params }: { params: { id: string } }) {
  const { data } = await supabase
    .from('invoices')
    .select('*')
    .eq('id', params.id) // no verifica que pertenezca al usuario
    .single()
  return Response.json(data)
}

// ✅ Correcto
export async function GET(req: Request, { params }: { params: { id: string } }) {
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return new Response('Unauthorized', { status: 401 })

  const { data } = await supabase
    .from('invoices')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id) // RLS + doble check
    .single()

  if (!data) return new Response('Not found', { status: 404 })
  return Response.json(data)
}
```

**Vectores específicos:**
- Tablas sin RLS habilitado en Supabase
- Políticas RLS con `USING (true)` sin filtro de usuario
- Rutas de admin accesibles sin verificación de rol server-side
- Middleware con matcher incompleto que deja rutas protegidas expuestas
- Server Actions que no verifican ownership del recurso

**Checks:**
- [ ] Toda tabla con datos de usuario tiene RLS habilitado
- [ ] No hay endpoints que acepten `user_id` del body para autorizar
- [ ] Las rutas de admin verifican rol en servidor, no solo en cliente

---

## A02 — Cryptographic Failures

**Qué es:** Datos sensibles expuestos por falta o mal uso de criptografía.

**En Next.js + Supabase:**

```typescript
// ❌ Datos sensibles en localStorage (sin cifrar, accesible por XSS)
localStorage.setItem('user_token', session.access_token)
localStorage.setItem('user_role', 'admin')

// ❌ Datos sensibles en URL
router.push(`/dashboard?token=${session.access_token}`)
// El token queda en logs del servidor, historial del browser y headers Referer

// ✅ Dejar que Supabase maneje las cookies de sesión (httpOnly, secure)
// No manipular tokens manualmente — usar supabase.auth.getSession()
```

**Vectores específicos:**
- `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` expuesta en bundle cliente
- JWT secret de Supabase en variables públicas
- Tokens de sesión en localStorage en vez de cookies httpOnly
- Datos PII sin cifrar en columnas de Supabase (sin column-level encryption)
- HTTPS no enforced (mixed content)

**Checks:**
- [ ] Ninguna clave sensible tiene prefijo `NEXT_PUBLIC_`
- [ ] Los tokens no se almacenan en localStorage ni en URL params
- [ ] HTTPS forzado en producción
- [ ] Columnas con PII consideran encryption at rest

---

## A03 — Injection

**Qué es:** Input no sanitizado que modifica la lógica de una query o comando.

**En Next.js + Supabase:**

```typescript
// ❌ Construcción dinámica de filtros con input del usuario
const column = req.query.sortBy // atacante puede pasar cualquier columna
const { data } = await supabase
  .from('products')
  .select('*')
  .order(column) // SQL injection via order clause

// ❌ textSearch con input sin sanitizar
const search = req.query.q
const { data } = await supabase
  .from('articles')
  .select('*')
  .textSearch('content', search) // puede ser explotado con operadores de tsquery

// ✅ Validar y allowlist el input antes de usarlo
const ALLOWED_SORT_COLUMNS = ['name', 'created_at', 'price']
const column = ALLOWED_SORT_COLUMNS.includes(req.query.sortBy)
  ? req.query.sortBy
  : 'created_at'

// ✅ Sanitizar input de búsqueda
const search = req.query.q?.replace(/[^a-zA-Z0-9 ]/g, '') // strip special chars
```

**Vectores específicos:**
- `.order()`, `.filter()`, `.textSearch()` con input del usuario sin validar
- RPC de Supabase con parámetros construidos dinámicamente
- XSS via contenido guardado en Supabase y renderizado sin sanitizar con `dangerouslySetInnerHTML`

```typescript
// ❌ XSS — contenido de DB renderizado directo
<div dangerouslySetInnerHTML={{ __html: post.content }} />

// ✅ Sanitizar antes de renderizar
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(post.content) }} />
```

**Checks:**
- [ ] No hay columnas de Supabase usadas en `.order()` o `.filter()` sin allowlist
- [ ] El input de búsqueda está sanitizado antes de pasarlo a `textSearch`
- [ ] No hay `dangerouslySetInnerHTML` con contenido de la DB sin sanitizar
- [ ] Las RPCs de Supabase usan parámetros tipados, no strings concatenados

---

## A04 — Insecure Design

**Qué es:** Fallas en el diseño de la arquitectura de seguridad, no en la implementación.

**En Next.js + Supabase:**

- Toda la lógica de autorización delegada solo al cliente (sin RLS como backstop)
- Roles de usuario almacenados en `user_metadata` (modificable por el usuario)
- Ausencia de rate limiting en flujos de auth (brute force en login/signup)
- Magic links o reset tokens sin expiración corta
- Flujos de negocio que permiten saltear pasos (ej: checkout sin validar stock)

```typescript
// ❌ Diseño inseguro: rol en metadata que el usuario puede modificar
const { data: { user } } = await supabase.auth.getUser()
if (user.user_metadata.role === 'admin') { ... } // user_metadata es editable

// ✅ Diseño seguro: rol en tabla separada con RLS, solo modificable por service_role
const { data: role } = await supabase
  .from('user_roles')
  .select('role')
  .eq('user_id', user.id)
  .single()
```

**Checks:**
- [ ] Los roles no dependen de `user_metadata`
- [ ] Hay rate limiting en `/api/auth/*` y formularios de login
- [ ] Los magic links expiran en menos de 1 hora
- [ ] Los flujos críticos de negocio tienen validación server-side en cada paso

---

## A05 — Security Misconfiguration

**Qué es:** Configuraciones por defecto inseguras, permisos excesivos, headers faltantes.

**En Next.js + Supabase:**

```javascript
// next.config.js
// ✅ Security headers
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline'", // unsafe-inline necesario para Next.js
      `connect-src 'self' ${process.env.NEXT_PUBLIC_SUPABASE_URL}`,
      "img-src 'self' data: blob:",
    ].join('; ')
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
]

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  }
}
```

```bash
# Verificar headers en producción
curl -I https://tuapp.com | grep -E "x-frame|x-content|strict-transport|content-security"
```

**Vectores específicos:**
- Supabase project con email confirmation desactivado
- Bucket de Storage público sin restricción de tipo de archivo
- `dangerouslyAllowBrowser: true` en cliente Supabase server-side
- Source maps expuestos en producción (revelan código fuente)
- Error messages detallados en producción (stack traces al cliente)

**Checks:**
- [ ] Security headers configurados en `next.config.js`
- [ ] Email confirmation habilitado en Supabase Auth
- [ ] Source maps deshabilitados en producción (`productionBrowserSourceMaps: false`)
- [ ] Los errores de producción no exponen stack traces al cliente
- [ ] Los buckets de Storage tienen restricciones de MIME type

---

## A06 — Vulnerable and Outdated Components

**Qué es:** Dependencias con vulnerabilidades conocidas.

**En Next.js + Supabase:**

```bash
# Auditar dependencias
npm audit

# Ver versiones desactualizadas
npm outdated

# Verificar versiones críticas
npm list next @supabase/supabase-js @supabase/ssr
```

**Vectores específicos:**
- Versiones antiguas de `@supabase/supabase-js` con vulnerabilidades en el cliente auth
- Next.js versions con CVEs conocidos (verificar https://nextjs.org/blog/security)
- Librerías de sanitización desactualizadas (DOMPurify, etc.)

**Checks:**
- [ ] `npm audit` no reporta vulnerabilidades high/critical
- [ ] `@supabase/supabase-js` y `@supabase/ssr` en versiones actuales
- [ ] Next.js en versión LTS activa sin CVEs críticos abiertos
- [ ] Dependabot o Renovate configurado en el repo

---

## A07 — Identification and Authentication Failures

**Qué es:** Fallos en la identificación de usuarios y gestión de sesiones.

**En Next.js + Supabase:**

```typescript
// ❌ Usar getSession() para decisiones de seguridad server-side
// getSession() lee de cookie sin revalidar contra el servidor
const { data: { session } } = await supabase.auth.getSession()
if (!session) return redirect('/login')
// Un token revocado podría pasar este check

// ✅ Siempre getUser() en server-side — valida contra Supabase Auth server
const { data: { user }, error } = await supabase.auth.getUser()
if (error || !user) return redirect('/login')
```

**Vectores específicos:**
- `getSession()` usado en middleware o API routes (no revalida el token)
- Ausencia de refresh token rotation
- Sesiones sin expiración configurada
- Falta de protección contra user enumeration en login/signup

```typescript
// ❌ User enumeration — revela si el email existe
// Supabase por defecto devuelve error diferente para email no registrado
// Configurar en Supabase Dashboard: Auth > Settings > "Enable email enumeration protection"

// ❌ No hay límite de intentos de login
// Configurar rate limiting en el endpoint de auth o usar Supabase's built-in protection
```

**Checks:**
- [ ] `getUser()` en todos los puntos de verificación server-side
- [ ] Email enumeration protection habilitado en Supabase Dashboard
- [ ] Refresh token rotation configurado
- [ ] MFA disponible para usuarios (Supabase soporta TOTP)

---

## A08 — Software and Data Integrity Failures

**Qué es:** Código o datos modificados sin verificación de integridad.

**En Next.js + Supabase:**

- Webhooks de Supabase sin verificación de firma
- Edge Functions sin validar el origen del request

```typescript
// ❌ Webhook sin verificar firma
export async function POST(req: Request) {
  const payload = await req.json()
  // procesar sin verificar que vino de Supabase
  await handleWebhook(payload)
}

// ✅ Verificar firma del webhook
import { createHmac } from 'crypto'

export async function POST(req: Request) {
  const signature = req.headers.get('x-supabase-signature')
  const body = await req.text()
  
  const expected = createHmac('sha256', process.env.WEBHOOK_SECRET!)
    .update(body)
    .digest('hex')
  
  if (signature !== expected) {
    return new Response('Invalid signature', { status: 401 })
  }
  
  await handleWebhook(JSON.parse(body))
}
```

**Checks:**
- [ ] Los webhooks de Supabase verifican firma HMAC
- [ ] Las dependencias de npm tienen lockfile commiteado (`package-lock.json`)
- [ ] No hay scripts de CI/CD que bajen dependencias sin verificar checksums

---

## A09 — Security Logging and Monitoring Failures

**Qué es:** Falta de logging adecuado para detectar y responder a incidentes.

**En Next.js + Supabase:**

```typescript
// ❌ No loguear eventos de seguridad críticos
export async function POST(req: Request) {
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return new Response('Unauthorized', { status: 401 })
  // nadie sabe que esto falló
}

// ✅ Loguear eventos de seguridad (sin incluir datos sensibles)
export async function POST(req: Request) {
  const { data: { user }, error } = await supabase.auth.getUser()
  if (error || !user) {
    console.warn('[AUTH_FAILURE]', {
      path: req.url,
      ip: req.headers.get('x-forwarded-for'),
      timestamp: new Date().toISOString(),
      // NO loguear tokens, passwords ni PII
    })
    return new Response('Unauthorized', { status: 401 })
  }
}

// ❌ Loguear datos sensibles accidentalmente
console.log('User session:', session) // contiene access_token y refresh_token
console.log('Auth error:', error, user) // puede contener email
```

**Checks:**
- [ ] Los intentos de acceso no autorizado se loguean (sin PII)
- [ ] No hay `console.log` con tokens, passwords o datos de sesión
- [ ] Supabase Auth logs habilitados en el Dashboard
- [ ] Alertas configuradas para patrones anómalos (muchos 401 en poco tiempo)

---

## A10 — Server-Side Request Forgery (SSRF)

**Qué es:** El servidor hace requests a URLs controladas por el atacante.

**En Next.js + Supabase:**

```typescript
// ❌ SSRF — el usuario controla la URL del fetch
// GET /api/preview?url=http://internal-service/admin
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url)
  const url = searchParams.get('url')
  
  const response = await fetch(url!) // atacante puede apuntar a servicios internos
  const data = await response.json()
  return Response.json(data)
}

// ✅ Allowlist de dominios permitidos
const ALLOWED_DOMAINS = ['api.example.com', 'cdn.example.com']

export async function GET(req: Request) {
  const { searchParams } = new URL(req.url)
  const url = searchParams.get('url')
  
  if (!url) return new Response('Bad request', { status: 400 })
  
  const parsed = new URL(url)
  if (!ALLOWED_DOMAINS.includes(parsed.hostname)) {
    return new Response('Domain not allowed', { status: 403 })
  }
  
  const response = await fetch(url)
  return Response.json(await response.json())
}
```

**Vectores específicos:**
- API routes que hacen `fetch()` con URLs del request del usuario
- Supabase Edge Functions que consumen URLs externas sin validar
- Open Graph / link preview endpoints sin restricción de dominio
- Image optimization de Next.js mal configurada en `next.config.js`

```javascript
// ❌ Permite cualquier dominio en image optimization
module.exports = {
  images: { domains: ['*'] }
}

// ✅ Solo dominios conocidos
module.exports = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'your-supabase-project.supabase.co' },
    ]
  }
}
```

**Checks:**
- [ ] No hay `fetch(userInput)` sin validación de dominio
- [ ] `next/image` tiene `remotePatterns` restringidos, no `domains: ['*']`
- [ ] Los endpoints de preview/proxy tienen allowlist de dominios
- [ ] Las Edge Functions de Supabase validan URLs externas