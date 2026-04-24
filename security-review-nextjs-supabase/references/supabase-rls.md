# Supabase RLS — Referencia Detallada

## Por qué RLS es crítico en este stack

En arquitecturas donde el frontend consume Supabase directamente (sin backend intermedio),
el `anon key` está expuesto en el cliente. TODA la seguridad de datos depende de que RLS
esté correctamente configurado. Un RLS ausente o mal escrito = acceso irrestricto a datos.

---

## Checks de RLS por tipo de política

### SELECT policies

**Patrón correcto:**
```sql
CREATE POLICY "users can only see their own data"
ON public.profiles
FOR SELECT
USING (auth.uid() = user_id);
```

**Patrones peligrosos:**
```sql
-- ❌ CRITICAL: permite ver todos los registros
USING (true)

-- ❌ HIGH: user_id viene del cliente, no de auth
USING (user_id = current_setting('app.user_id'))

-- ❌ MEDIUM: sin política = acceso denegado a todos (pero verificar que no sea error)
-- (tabla sin ninguna política con RLS habilitado)
```

### INSERT policies

```sql
-- ✅ Correcto: fuerza que user_id sea el del usuario autenticado
WITH CHECK (auth.uid() = user_id)

-- ❌ Permite insertar con cualquier user_id
WITH CHECK (true)
```

### UPDATE policies

```sql
-- ✅ Correcto: solo puede modificar sus propios registros
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id)

-- ❌ Permite modificar registros de otros
USING (auth.uid() = user_id)  -- sin WITH CHECK
```

---

## Storage Bucket Policies

### Patrón seguro para archivos de usuario:
```sql
-- SELECT: solo el dueño puede ver
((bucket_id = 'avatars') AND (auth.uid() = owner))

-- INSERT: solo puede subir a su propia carpeta
((bucket_id = 'documents') AND ((storage.foldername(name))[1] = auth.uid()::text))
```

### Patrón peligroso:
```sql
-- ❌ Bucket público sin restricción
(bucket_id = 'uploads')
```

---

## Roles en Supabase

| Rol | Cuándo se activa | Riesgo |
|-----|-----------------|--------|
| `anon` | Usuario NO autenticado | Alto si RLS no filtra bien |
| `authenticated` | Usuario autenticado | Medio — depende de policies |
| `service_role` | Backend con service key | CRÍTICO si se expone al cliente |

---

## Verificación manual en Dashboard Supabase

1. **Table Editor → [tabla] → RLS**: verificar que el toggle está ON
2. **Authentication → Policies**: revisar cada política de cada tabla
3. **Storage → [bucket] → Policies**: verificar restricciones de acceso
4. **SQL Editor**: ejecutar queries como anon para testear
   ```sql
   SET ROLE anon;
   SELECT * FROM profiles; -- debe devolver 0 filas o error
   ```

---

## Señales de alerta en el código

```typescript
// ❌ Usar service role en cliente browser
const supabase = createClient(url, SERVICE_ROLE_KEY) // CRITICAL

// ❌ Query sin filtro por usuario en componente cliente
const { data } = await supabase.from('orders').select('*') // depende de RLS

// ✅ Correcto — RLS filtra automáticamente por auth.uid()
const { data } = await supabase.from('orders').select('*')
// (solo seguro si hay política USING (auth.uid() = user_id))

// ❌ Pasar userId del cliente como filtro adicional (no sustituye a RLS)
const { data } = await supabase
  .from('orders')
  .select('*')
  .eq('user_id', userId) // solo confiar si RLS también está activo
```