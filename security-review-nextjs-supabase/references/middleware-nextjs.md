# Next.js Middleware — Referencia de Seguridad

## Rol del Middleware en la Arquitectura

El middleware es la primera línea de defensa para rutas protegidas en Next.js.
Si el matcher está mal configurado, un atacante puede acceder a rutas protegidas
simplemente usando variaciones de URL que el matcher no captura.

---

## Configuración del Matcher — Errores Frecuentes

### Matcher demasiado restrictivo (rutas protegidas sin cubrir):
```typescript
// ❌ Solo protege /dashboard pero no /dashboard/* ni /api/*
export const config = {
  matcher: ['/dashboard'],
}

// ✅ Cubre todas las rutas bajo /dashboard y /api
export const config = {
  matcher: [
    '/dashboard/:path*',
    '/api/:path*',
    '/settings/:path*',
    '/profile/:path*',
  ],
}
```

### Exclusiones peligrosas:
```typescript
// ❌ Excluir /api completo cuando hay rutas protegidas ahí
matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)']
// Cualquier ruta /api/* bypasea el middleware
```

---

## Patrón Seguro de Middleware con Supabase

```typescript
import { createServerClient } from '@supabase/ssr'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // ✅ CRÍTICO: usar getUser() no getSession()
  const { data: { user } } = await supabase.auth.getUser()

  // Rutas que requieren autenticación
  const protectedPaths = ['/dashboard', '/settings', '/profile', '/api/protected']
  const isProtected = protectedPaths.some(path => 
    request.nextUrl.pathname.startsWith(path)
  )

  if (isProtected && !user) {
    const redirectUrl = new URL('/login', request.url)
    // ✅ Guardar returnUrl pero validar que sea relativa
    const returnPath = request.nextUrl.pathname
    if (returnPath.startsWith('/')) { // solo paths relativos
      redirectUrl.searchParams.set('returnUrl', returnPath)
    }
    return NextResponse.redirect(redirectUrl)
  }

  return supabaseResponse
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

---

## Bypass de Middleware — Vectores Conocidos

### 1. Variaciones de case en path
```
/Dashboard  →  puede saltear matcher si es case-sensitive
/DASHBOARD  →  idem
```
Next.js normaliza paths pero verificar en la versión específica.

### 2. Trailing slash
```
/dashboard/  →  ¿el matcher captura esto?
```

### 3. Rutas con parámetros especiales
```
/dashboard%2F  →  URL encoding
/dashboard/.  →  path traversal attempt
```

### 4. Headers que modifican comportamiento
```
X-Forwarded-For, X-Real-IP  →  pueden afectar rate limiting basado en IP
```

---

## Checklist de Middleware

- [ ] El matcher cubre TODAS las rutas protegidas incluyendo sub-rutas (`/dashboard/:path*`)
- [ ] Se usa `getUser()` no `getSession()` para validar la sesión
- [ ] El `returnUrl` post-login valida que la URL es relativa (no externa)
- [ ] El middleware maneja correctamente el refresh de tokens (usando `@supabase/ssr`)
- [ ] Las rutas de assets estáticos están excluidas correctamente del matcher
- [ ] No hay rutas de API protegidas que el matcher omita
- [ ] El middleware no expone información de la sesión en headers de respuesta