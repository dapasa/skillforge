# Autenticación e Impersonation — Referencia Detallada

## getSession() vs getUser() — Diferencia crítica

### El problema con getSession():
```typescript
// ❌ PELIGROSO en server-side — puede estar desactualizado
const { data: { session } } = await supabase.auth.getSession()
// session puede ser válida localmente pero revocada en el servidor
// Un atacante podría manipular el token en cookies

// ✅ CORRECTO — valida contra el servidor de Supabase
const { data: { user } } = await supabase.auth.getUser()
// Hace una llamada al servidor y verifica el JWT
```

**Cuándo usar cada uno:**
- `getUser()` → siempre en server-side (API routes, Server Actions, middleware)
- `getSession()` → solo para verificaciones de UI en cliente (mostrar/ocultar elementos)

---

## Vectores de Impersonation

### 1. IDOR (Insecure Direct Object Reference)
```typescript
// ❌ Vulnerable: el user puede cambiar el ID en la URL
// GET /api/orders/[orderId]
export async function GET(req: Request, { params }: { params: { orderId: string } }) {
  const { data } = await supabase
    .from('orders')
    .select('*')
    .eq('id', params.orderId) // cualquier usuario puede pedir cualquier orden
    .single()
  return Response.json(data)
}

// ✅ Correcto: validar que el recurso pertenece al usuario autenticado
export async function GET(req: Request, { params }: { params: { orderId: string } }) {
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return new Response('Unauthorized', { status: 401 })
  
  const { data } = await supabase
    .from('orders')
    .select('*')
    .eq('id', params.orderId)
    .eq('user_id', user.id) // RLS debería hacer esto, pero doble check
    .single()
  
  if (!data) return new Response('Not found', { status: 404 }) // no revelar si existe
  return Response.json(data)
}
```

### 2. Roles en user_metadata (manipulable)
```typescript
// ❌ CRÍTICO: user_metadata puede ser modificado por el usuario
const user = await supabase.auth.getUser()
if (user.data.user.user_metadata.role === 'admin') {
  // peligroso — el usuario puede cambiar su propia metadata
}

// ✅ Correcto: roles en tabla separada con RLS, o en app_metadata (solo modificable por service_role)
const { data: profile } = await supabase
  .from('user_roles')
  .select('role')
  .eq('user_id', user.data.user.id)
  .single()
```

### 3. JWT Claims manipulados
```typescript
// ❌ Decodificar JWT manualmente sin verificar firma
const payload = JSON.parse(atob(token.split('.')[1]))
// Un atacante puede modificar el payload si no se verifica la firma

// ✅ Siempre usar getUser() que verifica la firma en Supabase
```

---

## Logout Seguro

```typescript
// ❌ Solo borra la sesión local
await supabase.auth.signOut({ scope: 'local' })

// ✅ Invalida el refresh token en el servidor también
await supabase.auth.signOut({ scope: 'global' })
// o simplemente:
await supabase.auth.signOut() // por defecto es 'global' en versiones recientes
```

---

## Checklist de Impersonation

- [ ] No se puede acceder a recursos de otro usuario cambiando el ID en URL/body
- [ ] Los roles NO se leen de `user_metadata` sin validación adicional
- [ ] `getUser()` se usa en todos los puntos de verificación server-side
- [ ] El logout revoca el refresh token globalmente
- [ ] Los errores de "recurso no encontrado" vs "sin permiso" no revelan existencia del recurso
- [ ] No hay endpoint de "admin" accesible por usuarios regulares
- [ ] La impersonación de admin (si existe) requiere service_role, no user JWT