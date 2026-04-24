---
name: seo-audit
description: >
  Ejecuta una auditoría SEO completa orientada a conversión y ventas, no solo tráfico.
  Usar SIEMPRE que el usuario mencione: auditoría SEO, SEO review, revisar el SEO,
  optimizar para Google, Core Web Vitals, mejorar posicionamiento, analizar tráfico orgánico,
  problemas de indexación, meta tags, sitemap, robots.txt, velocidad de carga SEO,
  schema markup, rich results, keywords, intención de búsqueda, on-page SEO, SEO técnico,
  "por qué no aparezco en Google", "quiero mejorar mis ventas desde Google", o
  cualquier análisis de visibilidad orgánica o conversión desde buscadores.
  También activar para: Next.js SEO, React SEO, SSR vs CSR para indexación,
  generateMetadata, next-sitemap, o "cómo saben los crawlers qué indexar".
---

# SEO Audit — Orientado a Conversión y Ventas

## Objetivo
Realizar una auditoría SEO con foco en resultados de negocio: no maximizar visitas,
sino maximizar visitas calificadas que conviertan. Producir un reporte Markdown con
hallazgos priorizados por impacto en ventas, con pasos accionables.

**Stack soportado:** Next.js / React, sitios estáticos (HTML/CSS), cualquier web servida por un dominio.

---

## Antes de comenzar — Preguntas de negocio

Antes de correr cualquier check técnico, entender el contexto de negocio:

1. ¿Cuál es el objetivo principal del sitio? (ventas directas / leads / awareness / contenido)
2. ¿Cuál es el producto o servicio principal que debe rankear?
3. ¿Hay Google Search Console conectado? ¿Y Google Analytics?
4. ¿El sitio tiene tráfico orgánico actual o partimos de cero?
5. ¿Existe alguna página que el cliente considere "la más importante para vender"?

Estas respuestas definen qué hallazgos tienen mayor prioridad en el reporte.

---

## Fase 0: Reconocimiento del Stack

```bash
# Detectar framework y estructura
ls -la package.json next.config.* astro.config.* gatsby-config.* 2>/dev/null

# Ver si es Next.js y qué versión
cat package.json 2>/dev/null | grep -E '"next"|"react"|"gatsby"|"astro"'

# Detectar tipo de rendering en Next.js (App Router vs Pages Router)
ls -d app/ src/app/ pages/ src/pages/ 2>/dev/null

# Buscar configuración de robots y sitemap
find . -name "robots.txt" -o -name "sitemap.xml" -o -name "sitemap*.xml" \
  | grep -v node_modules | grep -v .next

# Buscar next-sitemap u otras librerías de sitemap
cat package.json 2>/dev/null | grep -E "sitemap|robots|seo|meta"
```

Identificar y anotar:
- ¿Es SSR, SSG, ISR o CSR puro? (crítico para indexación)
- ¿Usa App Router (`app/`) o Pages Router (`pages/`)?
- ¿Tiene `next-sitemap`, `next-seo` u otra librería SEO?
- ¿Hay un `robots.txt` accesible?
- ¿Hay Google Search Console o Analytics configurado?

> ⚠️ **Si el sitio es CSR puro (React SPA sin SSR/SSG), esto es un hallazgo CRÍTICO.**
> Los crawlers de Google no ejecutan JavaScript de la misma forma — las páginas SPA sin rendering
> server-side pueden quedar sin indexar. Documentar como `T-01: CSR sin SSR` si aplica.

---

## Fase 1: Rastreabilidad e Indexación

> Referencia detallada: `references/crawlability.md`

### Checks automáticos:

```bash
# 1. Ver robots.txt (si existe localmente)
cat public/robots.txt 2>/dev/null || cat robots.txt 2>/dev/null

# 2. Buscar directivas noindex en código fuente
grep -rn "noindex\|nofollow\|X-Robots" . \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
  --include="*.html" --exclude-dir={node_modules,.next,.git,dist,out}

# 3. Buscar generación de metadata en Next.js App Router
grep -rn "generateMetadata\|export.*metadata" . \
  --include="*.ts" --include="*.tsx" --exclude-dir={node_modules,.next,.git}

# 4. Buscar tags meta robots en templates/layouts
grep -rn '<meta.*robots\|<meta.*name="robots"' . \
  --include="*.ts" --include="*.tsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

# 5. Detectar sitemap
find . -name "*.xml" | grep -i sitemap | grep -v node_modules
cat next-sitemap.config.* 2>/dev/null
grep -rn "sitemap" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git} | grep -v "//.*sitemap"
```

### Verificar manualmente:
- [ ] `robots.txt` existe y es accesible en `dominio.com/robots.txt`
- [ ] `robots.txt` NO bloquea las páginas que deberían indexarse
- [ ] Sitemap XML existe en `dominio.com/sitemap.xml` o `sitemap_index.xml`
- [ ] El sitemap incluye las páginas de mayor valor comercial
- [ ] No hay `noindex` en páginas de producto/servicio/landing por error
- [ ] En Next.js: las páginas de conversión usan SSR o SSG (no CSR puro)
- [ ] El `generateMetadata` o `<Head>` está presente en todas las páginas clave

**Clasificación de impacto:** Un `noindex` accidental en la homepage o páginas de producto = `🔴 CRÍTICO`.

---

## Fase 2: Core Web Vitals y Velocidad

> Referencia detallada: `references/core-web-vitals.md`

### Checks automáticos en código:

```bash
# 1. Detección de imágenes sin optimizar (Next.js: usar next/image, no <img>)
grep -rn '<img ' . --include="*.tsx" --include="*.jsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

# 2. Fonts con bloqueo de render
grep -rn '@import.*fonts.google\|<link.*fonts.google\|font-face' . \
  --include="*.css" --include="*.tsx" --include="*.ts" \
  --exclude-dir={node_modules,.next,.git}

# 3. Scripts de terceros sin async/defer (en HTML estático)
grep -rn '<script ' . --include="*.html" \
  --exclude-dir={node_modules,.next,.git} | grep -v "async\|defer\|type=\"module\""

# 4. En Next.js: uso de next/font (correcto) vs @import de Google Fonts (menos óptimo)
grep -rn "next/font\|next/script" . --include="*.ts" --include="*.tsx" \
  --exclude-dir={node_modules,.next,.git}

# 5. Componentes con imágenes en above-the-fold sin priority
grep -rn "priority\|fetchpriority\|preload" . --include="*.tsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}
```

### Herramientas externas a correr (indicar al usuario):
1. **PageSpeed Insights**: `https://pagespeed.web.dev/` → analizar versión **móvil** primero
2. **Google Search Console** → sección "Experiencia de página" y "Core Web Vitals"
3. **WebPageTest**: `https://www.webpagetest.org/` para filmstrip y waterfall

### Umbrales de referencia (Google):
| Métrica | Bueno | Necesita mejora | Malo |
|---------|-------|-----------------|------|
| LCP (carga mayor elemento) | ≤ 2.5s | 2.5–4s | > 4s |
| INP (interactividad) | ≤ 200ms | 200–500ms | > 500ms |
| CLS (estabilidad visual) | ≤ 0.1 | 0.1–0.25 | > 0.25 |

### Verificar manualmente:
- [ ] Imagen hero/banner tiene `priority` o `fetchpriority="high"` (afecta LCP directo)
- [ ] Se usa `next/image` en Next.js (optimización y lazy load automáticos)
- [ ] Fuentes se cargan con `next/font` o con `font-display: swap`
- [ ] No hay layout shifts (elementos que se mueven al cargar: banners de cookies, anuncios, imágenes sin dimensiones)
- [ ] Scripts de terceros (analytics, chat, pixels) se cargan con `next/script strategy="lazyOnload"` o similar

**Clasificación de impacto:** LCP > 4s en móvil en landing de producto = `🟠 ALTO`.

---

## Fase 3: On-Page SEO — Páginas de Mayor Valor Comercial

> Referencia detallada: `references/on-page-seo.md`

Foco: **no revisar todas las páginas**, sino las que tienen intención de compra/contratación.
Preguntar al usuario cuáles son si no está claro del código.

### Checks automáticos:

```bash
# 1. Buscar title tags y meta descriptions
grep -rn "<title\|metadata.*title\|generateMetadata" . \
  --include="*.tsx" --include="*.ts" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

grep -rn "description.*:\|<meta.*description" . \
  --include="*.tsx" --include="*.ts" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

# 2. Detectar H1 en páginas principales
grep -rn "<h1\|<H1" . --include="*.tsx" --include="*.jsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

# 3. Detectar páginas sin metadata explícita en App Router
find . -path "*/app/**/page.tsx" -o -path "*/app/**/page.ts" \
  | grep -v node_modules | xargs grep -L "metadata\|generateMetadata" 2>/dev/null

# 4. Verificar Open Graph (impacta CTR desde redes sociales y preview en Google)
grep -rn "openGraph\|og:title\|og:description\|og:image" . \
  --include="*.tsx" --include="*.ts" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}
```

### Verificar manualmente por página de valor comercial:
- [ ] El `<title>` incluye la keyword principal + nombre de marca (≤ 60 caracteres)
- [ ] La meta description es un CTA real, no un resumen genérico (≤ 155 caracteres)
- [ ] Hay exactamente **un** `<h1>` por página que incluye la keyword comercial principal
- [ ] Los `<h2>` y `<h3>` responden preguntas que el comprador buscaría
- [ ] La URL es corta, descriptiva y sin parámetros innecesarios (ej: `/servicios/consultoria-seo` no `/page?id=42`)
- [ ] Imágenes tienen `alt` descriptivo con keyword si es relevante (no `alt=""` en imágenes de producto)
- [ ] Open Graph configurado: las compartidas en redes muestran imagen + título atractivo
- [ ] No hay contenido duplicado entre páginas (mismo texto en /home y /about, por ejemplo)

**Clasificación de impacto:** Title tag genérico ("Inicio | Mi Empresa") en homepage = `🟠 ALTO`.

---

## Fase 4: Estructura del Sitio e Internal Linking

> Referencia detallada: `references/site-architecture.md`

El objetivo es que tanto usuarios como Google lleguen a las páginas de conversión en **máximo 3 clics**.

### Checks automáticos:

```bash
# 1. Ver estructura de rutas (Next.js App Router)
find . -path "*/app/**/page.tsx" | grep -v node_modules | sort

# 2. Ver estructura de rutas (Pages Router)
find . -path "*/pages/**/*.tsx" | grep -v node_modules | grep -v "_app\|_document\|api" | sort

# 3. Detectar links internos a páginas de conversión
grep -rn "href=" . --include="*.tsx" --include="*.jsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git} | grep -v "http\|mailto\|tel\|#"

# 4. Buscar breadcrumbs
grep -rn "breadcrumb\|Breadcrumb" . --include="*.tsx" --include="*.ts" \
  --exclude-dir={node_modules,.next,.git}

# 5. Detectar huérfanas: páginas sin links entrantes (indicativo)
# (manual: cruzar lista de rutas con href encontrados arriba)
```

### Verificar manualmente:
- [ ] Las páginas de producto/servicio/landing están accesibles desde el menú principal en **≤ 1 clic**
- [ ] Hay internal links desde el blog/contenido hacia las páginas comerciales (con anchor text descriptivo)
- [ ] No hay páginas de conversión "huérfanas" (sin ningún link interno que apunte a ellas)
- [ ] El menú principal tiene arquitectura plana: máximo 2 niveles de profundidad
- [ ] Hay breadcrumbs en páginas internas (mejora UX y habilita rich results en Google)
- [ ] Los CTAs ("Contratar", "Ver precios", "Solicitar demo") enlazan directamente a la página de conversión, no al inicio

**Clasificación de impacto:** Páginas de servicio sin links internos = `🟡 MEDIO-ALTO`.

---

## Fase 5: Schema Markup y Rich Results

Los rich results (reseñas con estrellas, preguntas frecuentes, precios) aumentan el CTR sin subir posiciones.

### Checks automáticos:

```bash
# 1. Buscar schema markup existente
grep -rn "schema.org\|@context\|@type\|application/ld+json" . \
  --include="*.tsx" --include="*.ts" --include="*.html" \
  --exclude-dir={node_modules,.next,.git}

# 2. Detectar páginas de FAQ, pricing, reviews sin schema
grep -rn "FAQ\|Preguntas frecuentes\|precios\|pricing\|reviews\|testimonios" . \
  --include="*.tsx" --include="*.html" \
  --exclude-dir={node_modules,.next,.git} -i
```

### Schemas de mayor impacto en ventas:
| Schema | Impacto | Aplicable cuando |
|--------|---------|-----------------|
| `Product` + `Offer` | 🔴 Alto | Tienda online, producto específico |
| `LocalBusiness` | 🔴 Alto | Negocio local con dirección física |
| `FAQPage` | 🟠 Medio | Página de preguntas frecuentes |
| `Review` / `AggregateRating` | 🟠 Medio | Testimonios o reviews de clientes |
| `BreadcrumbList` | 🟡 Bajo | Cualquier sitio con navegación jerárquica |
| `Organization` | 🟡 Bajo | Cualquier sitio empresarial |

### Verificar manualmente:
- [ ] La homepage tiene schema `Organization` o `LocalBusiness`
- [ ] Páginas de producto tienen `Product` con `Offer` (precio, disponibilidad)
- [ ] Páginas de FAQ/precios usan `FAQPage`
- [ ] Los schemas son válidos: probar en `https://search.google.com/test/rich-results`
- [ ] No hay schemas incorrectos o que describan contenido que no está en la página

---

## Fase 6: Intención de Búsqueda y Alineación con el Comprador

Este es el check más impactante en conversiones. SEO sin intención de búsqueda trae tráfico que no compra.

### Qué revisar (no automatizable, requiere criterio):

**Para cada página comercial clave, preguntarse:**

1. **¿Cuál es la keyword objetivo?** ¿Es transaccional ("contratar X"), informacional ("qué es X") o de comparación ("X vs Y")?
   - Una landing de ventas debe apuntar a keywords transaccionales
   - Un blog post que rankea con keyword transaccional pero ofrece contenido informativo = tráfico que no convierte

2. **¿El contenido de la página satisface lo que el usuario espera ver al buscar esa keyword?**
   - Buscar la keyword en Google (incógnito) y comparar los primeros resultados con el contenido del sitio
   - Si Google muestra comparativas y el sitio muestra solo características: desalienación de intención

3. **¿El CTA está visible sin scroll en desktop y móvil?**
   - El botón de conversión debe estar above-the-fold
   - Verificar en móvil: ¿el CTA sigue siendo visible o queda enterrado?

4. **¿El contenido responde las objeciones del comprador?**
   - Precio, garantía, diferenciadores vs competencia, prueba social

### Señales de alerta a documentar:
- [ ] Páginas de servicio sin keyword transaccional en el título
- [ ] Blog posts con keyword de compra que redirigen a contenido informativo (oportunidad de optimizar o crear landing específica)
- [ ] CTAs genéricos ("Más información") en vez de específicos ("Solicitar presupuesto gratis")
- [ ] Falta de prueba social en páginas comerciales (testimonios, logos de clientes, cifras)

---

## Fase 7: Generación del Reporte

Al finalizar todos los checks, generar el reporte usando esta estructura:

```markdown
# SEO Audit Report — [Nombre del Proyecto / Dominio]

**Fecha:** [fecha]
**URL analizada:** [url]
**Stack:** [Next.js versión / Static / otro]
**Objetivo del sitio:** [ventas / leads / contenido]
**Revisado por:** Claude Code (SEO Audit Skill)

---

## Executive Summary

[3-4 líneas: estado general del SEO, cuántos hallazgos críticos, y el principal
freno actual para las conversiones desde búsqueda orgánica]

**Estado general:** [🔴 Crítico / 🟠 Necesita trabajo / 🟡 Mejorable / 🟢 Sólido]

---

## Hallazgos por Impacto en Ventas

### 🔴 CRÍTICO — Bloquea conversiones o indexación
| ID | Área | Descripción | Página afectada | Esfuerzo |
|----|------|-------------|-----------------|---------|
| C-01 | Indexación | ... | ... | Bajo/Medio/Alto |

### 🟠 ALTO — Reduce tráfico calificado o tasa de conversión significativamente
| ID | Área | Descripción | Página afectada | Esfuerzo |
|----|------|-------------|-----------------|---------|

### 🟡 MEDIO — Mejoras con impacto real pero no urgentes
| ID | Área | Descripción | Página afectada | Esfuerzo |
|----|------|-------------|-----------------|---------|

### 🟢 OPTIMIZACIÓN — Quick wins o mejoras incrementales
| ID | Área | Descripción | Página afectada | Esfuerzo |
|----|------|-------------|-----------------|---------|

---

## Detalle de Hallazgos

### C-01: [Nombre del hallazgo]
**Impacto:** [explicación concreta de cómo afecta las ventas/conversiones]
**Diagnóstico:** [qué se encontró exactamente]
**Solución — Paso a paso:**
1. [paso concreto]
2. [paso concreto]
3. [cómo verificar que está resuelto]

[repetir por cada hallazgo]

---

## Plan de Acción Priorizado

### Semana 1 — Quick wins (máximo impacto, mínimo esfuerzo)
- [ ] [acción concreta con responsable y verificación]

### Semana 2-3 — Correcciones técnicas
- [ ] [acción]

### Mes 2 — Optimización continua
- [ ] [acción]

---

## Métricas de Seguimiento

Una vez implementadas las correcciones, monitorear en Google Search Console:
- Impresiones y clics en páginas corregidas (sección "Rendimiento")
- Cobertura de índice: reducir errores de indexación
- Core Web Vitals: sección "Experiencia de página"
- Rich results: sección "Mejoras" para schemas nuevos

---

## Resumen Ejecutivo para el Cliente

[Versión no técnica, en lenguaje de negocio, para presentar a quien toma decisiones]

**Qué está pasando:** [problema en términos de negocio, no técnicos]
**Por qué importa:** [impacto en ventas/leads estimado]
**Qué proponemos hacer:** [acciones concretas]
**Cuándo ver resultados:** [expectativas realistas: SEO orgánico = 3-6 meses para resultados sostenidos]
```

---

## Clasificación de Hallazgos — Criterios

| Nivel | Criterio de Impacto en Ventas |
|-------|-------------------------------|
| 🔴 CRÍTICO | Bloquea la indexación, CSR sin SSR en páginas clave, `noindex` accidental, sitio no rastreable |
| 🟠 ALTO | Title/H1 genérico en páginas comerciales, LCP > 4s en móvil, sin sitemap, intención de búsqueda desalineada en landing principal |
| 🟡 MEDIO | Meta descriptions sin CTA, falta de schema en páginas con FAQ/precios, internal linking débil, imágenes sin alt |
| 🟢 OPTIMIZACIÓN | Open Graph mejorable, breadcrumbs ausentes, oportunidades de keywords de cola larga, CTAs poco específicos |

---

## Notas de Ejecución

1. **Siempre empezar por Fase 0 y las preguntas de negocio** — sin contexto, el reporte no puede priorizar correctamente
2. **Fase 1 (indexación) y Fase 6 (intención) son las de mayor impacto en ventas** — si hay tiempo limitado, focalizarse ahí
3. **No reportar "falta de contenido"** como hallazgo a menos que el usuario lo haya pedido — el audit es técnico + on-page, no content strategy
4. **El reporte final siempre incluye la sección "Resumen Ejecutivo para el Cliente"** — muchos usuarios de este skill son consultores que presentan a terceros
5. **Expectativas SEO:** ser honesto — resultados orgánicos tardan 3-6 meses; los quick wins de CTR (rich results, title tags) pueden verse en 2-4 semanas tras reindexación
6. **Para Next.js:** el error más común y más impactante es tener páginas comerciales renderizadas client-side sin `generateStaticParams` o SSR — verificar siempre primero
