# Secrets y Variables de Entorno — Referencia Detallada

## Jerarquía de Claves en Supabase

| Variable | Exposición | Riesgo si se filtra |
|----------|-----------|---------------------|
| `NEXT_PUBLIC_SUPABASE_URL` | Pública (browser) | Ninguno — es la URL del proyecto |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Pública (browser) | Alto si RLS está mal — Bajo si RLS es correcto |
| `SUPABASE_SERVICE_ROLE_KEY` | PRIVADA (solo server) | CRÍTICO — bypasea toda la RLS |
| `SUPABASE_JWT_SECRET` | PRIVADA (solo server) | CRÍTICO — permite firmar JWTs arbitrarios |

---

## Patrones de Exposición Accidental

### 1. Service role en `NEXT_PUBLIC_*`
```bash
# .env.local — MAL
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=eyJ... # ❌ CRITICAL: expuesto al browser
```

### 2. Service role en código cliente
```typescript
// ❌ CRITICAL: cualquiera que inspeccione el bundle puede ver esta key
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // esto se incluye en el bundle cliente
)
```

### 3. Secrets en next.config.js expuestos al cliente
```javascript
// next.config.js
module.exports = {
  env: {
    // ❌ Estas variables van al bundle del cliente
    SUPABASE_SERVICE_ROLE_KEY: process.env.SUPABASE_SERVICE_ROLE_KEY,
  }
}
// ✅ Solo usar 'env' para variables verdaderamente públicas
// Para server-only, acceder directo con process.env.VARIABLE en server components/routes
```

### 4. Logging accidental de secrets
```typescript
// ❌
console.log('Supabase client:', supabase) // puede loguear la key
console.log('Config:', { url, key }) // ❌
console.error('Auth error:', error, { session }) // session contiene tokens
```

---

## Verificación de .gitignore

```bash
# Verificar que .env.local está ignorado
cat .gitignore | grep -E "\.env"

# Buscar si hay .env* en el historial de git (aunque esté en .gitignore ahora)
git log --all --full-history -- "**/.env*"
git log --all --full-history -- ".env*"

# Buscar secrets en el historial completo
git grep -l "service_role\|supabase.*key" $(git rev-list --all)
```

---

## Variables de Entorno Recomendadas

```bash
# .env.local (NUNCA commitear)
NEXT_PUBLIC_SUPABASE_URL=https://[project].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...          # OK en cliente con RLS correcto
SUPABASE_SERVICE_ROLE_KEY=eyJ...              # SOLO server-side
SUPABASE_JWT_SECRET=...                        # SOLO server-side (si se validan JWTs custom)

# Otras variables sensibles comunes
STRIPE_SECRET_KEY=sk_...                       # NUNCA NEXT_PUBLIC_
SENDGRID_API_KEY=SG....                        # NUNCA NEXT_PUBLIC_
DATABASE_URL=postgresql://...                  # NUNCA NEXT_PUBLIC_
```

---

## Checklist de Secrets

- [ ] Ninguna variable con service_role o JWT_SECRET tiene prefijo `NEXT_PUBLIC_`
- [ ] `.env.local` y `.env*.local` están en `.gitignore`
- [ ] No hay secrets en el historial de git
- [ ] `next.config.js` no expone variables sensibles bajo `env:`
- [ ] Los logs de producción no contienen tokens ni keys
- [ ] Las variables de CI/CD (GitHub Actions/Secrets) están marcadas como secretas
- [ ] El Supabase service_role key solo se usa en API routes o Server Actions (nunca en componentes cliente)