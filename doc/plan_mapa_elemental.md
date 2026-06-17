# Plan — Mapa Elemental (lead magnet gratuito)

_Documento de planeación. Fecha: 2026-06-16._

---

## Qué vamos a construir

Un mini-informe gratuito ("Mapa Elemental") que muestra cómo se reparten
los cuatro elementos —fuego, tierra, aire, agua— en la carta natal de una
persona. Sirve de puerta de entrada: la persona lo pide gratis, y de ahí se
le invita al análisis completo de pago.

Hoy existe solo como una **maqueta visual** (el archivo `mapa.html`): se ve
terminada y bonita, pero el formulario es "de mentiras" —muestra un mensaje
de "Recibido" pero no guarda ni envía nada a ningún lado—. El trabajo es
volverla real y publicarla.

---

## Decisiones ya tomadas

1. **Dónde vivirá:** dentro de la aplicación grande (la de Vercel), como una
   página más: `tucartografia.com/mapa-elemental`. No como sitio aparte.
   _Razón: escala mejor para futuros productos y se conecta solo con la base
   de datos._

2. **A dónde llegan los datos:** se guardan en **Supabase** (la base de datos
   que ya usa el proyecto). A Sergio le llega **solo una notificación** de que
   entró un nuevo Mapa —sin los datos en el correo—.

3. **Sitio en vivo:** `www.tucartografia.com` lo sirve la **aplicación grande**
   (Vercel), no la página sencilla de GitHub. Esto importa para el SEO (ver
   Fase 4).

4. **Modelo para ejecutar:** Opus 4.8 para construir (Fases 1–2); Sonnet 4.6
   alcanza para las fases mecánicas (3–5).

---

## El plan, por fases

### Fase 1 — Reconstruir el Mapa Elemental dentro de la aplicación
- Crear la página `tucartografia.com/mapa-elemental`.
- Pasar el diseño de la maqueta a la forma que usa la aplicación,
  conservando el aspecto **idéntico**.
- Quitar lo que era solo de prueba: el panel de "tweaks" (herramienta de
  diseño) y las librerías en "modo desarrollo" (que cargan lento).

### Fase 2 — Conectar el formulario a Supabase (de verdad)
- Crear el "buzón" en la base de datos donde caen los datos del Mapa.
- Conectar el formulario para que, al enviar:
  - **guarde** los datos en Supabase, y
  - **mande a Sergio una notificación** (sin los datos en el correo).
- Esto reutiliza el mecanismo que ya existe para las compras, en versión
  más simple (sin cobro, porque es gratis).

### Fase 3 — Ligar el sitio principal ↔ el Mapa Elemental
- **Del Mapa → al sitio principal:**
  - Botón "IR AL ANÁLISIS COMPLETO" → `https://www.tucartografia.com/checkout`,
    con una "marca invisible" en la dirección (`?desde=mapa`) para que el
    botón "Regresar" del checkout devuelva a la persona al Mapa Elemental.
  - Enlaces del pie **TYC** y **Privacidad** → los PDFs que ya usa el sitio
    (`/TuCartografia_Terminos_Condiciones.pdf` y
    `/TuCartografia_Aviso_de_Privacidad.pdf`).
- **Ajuste en el checkout:** hoy el botón "Regresar" siempre va a la portada.
  Hacerlo condicional: si la persona viene del Mapa (marca `?desde=mapa`),
  regresa al Mapa; si no, se queda como está (va a la portada). _No romper
  el flujo de quien llega desde el sitio principal._
- **Del sitio principal → al Mapa:** agregar una invitación al Mapa
  Elemental gratuito que lleve a la página nueva.
- Resultado: ida y vuelta. El gratuito alimenta al de pago; el de pago
  ofrece el gratuito como entrada.

### Fase 4 — Que los buscadores lo encuentren
- **Verificar primero** cuál sitemap está leyendo Google: el de la página
  sencilla (GitHub, el que tocamos esta mañana) o el de la aplicación
  grande (Vercel). Como el sitio en vivo es la aplicación, lo más probable
  es que el válido sea el de la aplicación. Hay que confirmarlo y dejar uno
  solo, correcto.
- Agregar título y descripción a la página del Mapa.
- Sumar `/mapa-elemental` al sitemap de la aplicación (es agregar una línea).
- Pedir indexación en Search Console (Google y Bing).

### Fase 5 — Publicar
- La aplicación ya vive en Vercel; al subir los cambios, la página queda en
  vivo automáticamente.

---

## Notas de ejecución (para arrancar sin redescubrir)

**Archivos clave:**
- Maqueta original (fuente de verdad del diseño): `web/landing 2/mapa.html`
- App donde se construye: carpeta `web/` (Next.js en Vercel)
- Portada actual de la app: `web/app/page.tsx`
- Patrón a copiar para guardar en Supabase + notificar: `web/app/api/intake/route.ts`
  (el Mapa es una versión más simple: sin Stripe, sin verificación de pago)
- Conexión a base de datos: `web/lib/supabase.ts` · Correos: `web/lib/resend.ts`
- Checkout (ajustar botón "Regresar"): `web/app/checkout/CheckoutUI.tsx` (línea ~22)
- Sitemap de la app: `web/app/sitemap.ts` · Robots: `web/app/robots.ts`

**Diseño final ya elegido (fijar al quitar el panel de tweaks):**
- Paleta (temperamento): `técnico` (la base, no arcaico ni severo)
- Color de acento: `#490E0E` (borgona)
- Densidad: `holgada`
- Microcopy de la hora: `asterisco`

**Ojo técnico al portar:** la maqueta usa nombres de variables de su propio
design system (`--bg-1`, `--tc-borgona`, `--space-*`, etc.). La app ya tiene
los suyos en `web/app/globals.css` (`--crema`, `--navy`, `--borgona`,
`--reticula-5mm`…). Hay que **reconciliar** ambos: revisar qué nombres ya
existen en la app antes de copiar el CSS, para no duplicar ni romper estilos.

**Recordatorio (de `web/AGENTS.md`):** "este no es el Next.js que conoces" —
leer la guía en `node_modules/next/dist/docs/` antes de escribir código.

## Orden de ejecución sugerido
1 → 2 → 3 → 4 → 5. Las Fases 1 y 2 son el corazón (sin ellas no hay producto
real). Las 3, 4 y 5 lo conectan, lo hacen encontrable y lo publican.
