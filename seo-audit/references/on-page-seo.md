# On-Page SEO — Referencia Detallada

## Title Tag — El elemento más importante para el CTR

El `<title>` es lo primero que ve el usuario en los resultados de Google.
Un buen title aumenta el CTR (clicks / impresiones) sin subir de posición.

**Fórmula para páginas comerciales:**
```
[Keyword Principal] — [Beneficio o diferenciador] | [Marca]
```

Ejemplos:
- ✅ `Consultoría SEO en Madrid — Resultados en 90 días | Nombre Empresa`
- ✅ `Software de Facturación para Autónomos | Nombre Producto`
- ❌ `Inicio | Mi Empresa` (genérico, sin keyword)
- ❌ `Bienvenido a nuestra web` (no dice nada al usuario ni a Google)

**Límite:** 55-60 caracteres (Google trunca el resto en los resultados)

---

## Meta Description — El texto que convierte impresiones en clicks

La meta description no es un factor de ranking directo, pero **sí afecta el CTR**.
Más CTR → más tráfico → señal positiva para Google.

**Fórmula:**
```
[Qué ofreces] + [Para quién] + [CTA específico]
```

Ejemplos:
- ✅ `Aumenta tus ventas con SEO técnico y de contenido. Auditoría gratuita para empresas B2B. Solicita la tuya hoy.`
- ❌ `Somos una empresa de marketing digital con años de experiencia en el sector.`

**Límite:** 150-155 caracteres

---

## Estructura de Headings (H1–H6)

| Heading | Regla | Uso correcto |
|---------|-------|-------------|
| H1 | Uno por página | Keyword principal + propuesta de valor |
| H2 | Varios permitidos | Secciones principales, responden preguntas del comprador |
| H3 | Varios permitidos | Sub-secciones, detalles, FAQs |

**El H1 no es el título de la página en el logo ni en el nav — es el heading principal del contenido.**

---

## URLs — Estructura limpia

**Reglas:**
- Lowercase siempre: `/servicios/consultoria-seo` no `/Servicios/ConsultoriaSEO`
- Guiones medios, no guiones bajos: `/sobre-nosotros` no `/sobre_nosotros`
- Sin parámetros en URLs indexables: `/producto/zapatos-rojos` no `/producto?id=123&color=rojo`
- Cortas: máximo 3-4 segmentos de profundidad

---

## Open Graph — CTR desde redes sociales y previews

```tsx
// Next.js App Router
export const metadata = {
  title: 'Título de la página',
  description: 'Descripción',
  openGraph: {
    title: 'Título para redes sociales',
    description: 'Descripción para compartir',
    images: [{ url: '/og-image.png', width: 1200, height: 630 }],
  },
}
```

La imagen OG debe ser 1200×630px. Sin ella, las compartidas en LinkedIn/WhatsApp se ven genéricas.

---

## Checklist On-Page por Página Comercial

```
[ ] Title tag: keyword + beneficio + marca, ≤ 60 chars
[ ] Meta description: CTA específico, ≤ 155 chars  
[ ] H1 único: contiene la keyword principal
[ ] H2s: responden preguntas del comprador (intención comercial)
[ ] URL: corta, con keyword, sin parámetros
[ ] Imágenes: alt text descriptivo con keyword si aplica
[ ] Open Graph: título, descripción e imagen configurados
[ ] Internal links: al menos 2-3 links desde otras páginas del sitio
[ ] CTA visible: above-the-fold en desktop y móvil
[ ] Prueba social: testimonios, logos, métricas visibles
```
