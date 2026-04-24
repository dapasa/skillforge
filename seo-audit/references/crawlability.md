# Crawlability — Referencia Detallada

## Cómo funciona el rastreo de Google

Google usa Googlebot para rastrear webs. El proceso:
1. **Descubre URLs** → vía sitemaps, links externos, links internos
2. **Rastrea** → descarga el HTML (y ejecuta JS con demora)
3. **Indexa** → procesa el contenido y lo añade al índice
4. **Rankea** → decide posición según relevancia + autoridad

Si Googlebot no puede rastrear una página, **no puede mostrarla**.

---

## robots.txt — Formato correcto

```
User-agent: *
Disallow: /admin/
Disallow: /checkout/
Disallow: /gracias/
Allow: /

Sitemap: https://dominio.com/sitemap.xml
```

**Errores comunes:**
- `Disallow: /` → bloquea todo el sitio (suele ocurrir en entornos de staging que pasan a producción)
- Falta de `Sitemap:` en robots.txt → Google tarda más en descubrir el sitemap
- Bloquear carpetas `/wp-admin/` pero olvidar que están enlazadas desde el frontend

---

## Sitemap XML — Qué incluir

**Incluir:**
- Páginas de producto/servicio
- Landing pages de conversión
- Homepage
- Páginas de categorías (si hay estructura jerárquica)
- Posts del blog (si son parte de la estrategia SEO)

**Excluir:**
- Páginas de "gracias por tu compra"
- Formularios vacíos
- Páginas de login/registro
- URLs con parámetros de sesión o filtros dinámicos (duplicados)
- Páginas con `noindex`

**En Next.js con next-sitemap:**
```js
// next-sitemap.config.js
module.exports = {
  siteUrl: 'https://dominio.com',
  generateRobotsTxt: true,
  exclude: ['/admin/*', '/gracias', '/checkout/*'],
}
```

---

## CSR vs SSR — El problema de indexación en React/Next.js

| Tipo | Qué ve Googlebot | Impacto SEO |
|------|-----------------|-------------|
| SSG (Static Site Generation) | HTML completo pre-renderizado | ✅ Óptimo |
| SSR (Server Side Rendering) | HTML completo en cada request | ✅ Bueno |
| ISR (Incremental Static Regen) | HTML pre-renderizado con revalidación | ✅ Bueno |
| CSR puro (Client Side Rendering) | HTML vacío + JS que carga el contenido | ❌ Problemático |

**En CSR puro**, Googlebot ve algo como:
```html
<div id="root"></div>
<script src="/bundle.js"></script>
```

Google *puede* ejecutar JS, pero con demora y limitaciones. Las páginas CSR puro pueden:
- Tardar semanas en indexarse
- Indexarse con contenido incompleto
- No indexarse en absoluto si el JS falla

**Cómo verificar si una página es CSR:**
1. Abrir la URL en Chrome
2. Deshabilitar JavaScript: DevTools → Settings → Debugger → Disable JavaScript
3. Recargar: si la página está en blanco o vacía → es CSR puro

**Solución en Next.js:** usar `generateStaticParams` para SSG o `dynamic = 'force-dynamic'` para SSR en pages que deben indexarse.
