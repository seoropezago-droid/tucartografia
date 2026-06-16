# Auditoría Tu Cartografía — Junio 2026

Estado: Fase 1 completa · Fases 2-4 pendientes
Regla: solo lectura durante auditoría; correcciones se aprueban una por una.

---

## FASE 1 — SEGURIDAD (completa)

### CRÍTICO

**S1. ~~`/admin` sin protección~~ — FALSO POSITIVO (corregido 2026-06-09)**
- La protección SÍ existe en `web/proxy.ts` (convención de Next.js 16 que reemplaza a `middleware.ts`). La auditoría inicial buscó solo `middleware.ts` y no lo encontró.
- `/admin/*` redirige a `/admin/login` si la cookie no coincide. Las debilidades del esquema de cookie se mantienen como hallazgo S4.

**S2. `/api/intake` acepta inserciones sin validar pago — CORREGIDO (2026-06-09)**
- Archivo: `web/app/api/intake/route.ts`
- Inserta en BD con `payment_status: "confirmed"` hardcodeado, sin verificar que `stripe_session_id` corresponda a una sesión pagada real. Tampoco valida ningún campo.
- Cualquiera puede: crear registros falsos "pagados", disparar emails de Resend a direcciones arbitrarias (abuso/spam que puede quemar la reputación del dominio), y contaminar la BD.
- Fix aplicado: antes de insertar, la ruta verifica la sesión contra Stripe (rechaza con 402 si no está pagada o no existe) y rechaza `session_id` duplicado con 409 (consulta previa; el UNIQUE de BD queda como respaldo). Bypass `session_id=dev` solo en desarrollo local, mismo criterio que `verify-payment`. Pendiente de deploy.
- Nota: al verificar, se detectó que `verify-payment/route.ts:24` compara contra `"no_payment_needed"`, valor que no existe en Stripe (el correcto es `"no_payment_required"`); hoy solo pasa `"paid"`, que es más estricto, no menos. No se tocó por estar fuera del alcance de S2.

### ALTO

**S3. `/api/debug-env` expuesta públicamente — CORREGIDO (2026-06-09)**
- Devolvía existencia y prefijos (7-10 chars) de STRIPE_SECRET_KEY, RESEND_API_KEY, etc. a cualquier visitante.
- Fix aplicado: ruta eliminada (`web/app/api/debug-env/`). Sin referencias restantes en el código. Pendiente de deploy.

**S4. Autenticación de admin débil — CORREGIDO (2026-06-09)**
- Archivos: `web/app/api/admin/login/route.ts`, `web/app/api/deliver/route.ts`, `web/proxy.ts`
- La cookie `admin_auth` contiene la contraseña en texto plano (viaja en cada request).
- Falta el flag `secure: true` en la cookie.
- Comparación `password !== process.env.ADMIN_PASSWORD` no es constante en tiempo (timing attack, menor).
- Sin rate limiting en el login → fuerza bruta ilimitada.
- Fix aplicado: nueva librería `web/lib/admin-auth.ts` centraliza la auth. La cookie ahora lleva un token firmado HMAC-SHA256 con expiración de 8h (`<exp_ms>.<firma>`) — la contraseña ya no viaja. Cookie con `secure: true` en producción. Todas las comparaciones (contraseña y token) en tiempo constante vía `timingSafeEqual`. Rate limit en login: 5 intentos fallidos por IP cada 15 min (en memoria de instancia — básico; no persiste entre instancias serverless, suficiente para frenar fuerza bruta sostenida). `proxy.ts` y `deliver` validan el token firmado. Pendiente de deploy. Nota: la sesión activa actual se invalida; basta volver a entrar en `/admin/login`.

### MEDIO

**S5. Bucket `informes` de Supabase Storage es público**
- Archivo: `web/app/api/deliver/route.ts:51` (usa `getPublicUrl`)
- Los PDFs de informes (contenido personal sensible) son accesibles a cualquiera con la URL. El nombre es `{tc_number}_{timestamp_ms}.pdf` — difícil de adivinar pero no imposible si tc_number es secuencial.
- Fix: bucket privado + signed URLs con expiración, o al mínimo nombres con UUID.

**S6. `/api/checkout` devuelve mensajes de error internos al cliente — CORREGIDO (2026-06-10)**
- Archivo: `web/app/api/checkout/route.ts:25` — `err.message` crudo puede filtrar detalles de configuración de Stripe.
- Fix aplicado: el cliente recibe "No se pudo iniciar el pago. Intenta de nuevo."; el detalle real sigue en `console.error` para los logs de Vercel. Pendiente de deploy.

**S7. `/api/verify-payment` reenvía emails sin control real de duplicados**
- Archivo: `web/app/api/verify-payment/route.ts:7` — el `Set` en memoria no persiste entre instancias serverless. Cualquiera con un session_id válido puede provocar reenvíos del email de intake. (También es hallazgo operativo → Fase 2.)
- Fix: marcar el envío en BD, no en memoria.

### VERIFICADO OK
- Webhook de Stripe verifica firma correctamente.
- `web/.env.local` está ignorado por git; sin secretos en el historial.
- Cliente Supabase con service role key solo se usa server-side (no se filtra al cliente).

---

## FASE 2 — ERRORES OPERATIVOS (completa 2026-06-09)

### ALTO

**O1. Los 8 registros de la BD figuran como NO entregados — RESUELTO (2026-06-09)**
- Sergio confirmó que TC-037 a TC-040 fueron entregados manualmente. Se marcaron `diagnosis_status = 'delivered'` en BD (decisión: `informe_sent_at` se dejó NULL por no conocerse la fecha exacta).

Detalle original:
- Todos tienen `diagnosis_status = 'pending'`, `informe_url = NULL`, `informe_sent_at = NULL` — incluidos TC-037 a TC-040 (17–24 de mayo, hace 2-3 semanas; la promesa al cliente es 24-48h).
- Dos lecturas posibles: (a) las entregas ocurrieron fuera del sistema (email manual) y la BD no lo registra — cero trazabilidad; o (b) hay clientes sin informe. Sergio debe confirmar caso por caso.
- Fix: registrar entregas pasadas en BD (update manual) y entregar siempre vía `/admin/[id]` para que quede trazado.

### MEDIO

**O2. 4 registros sin `tc_number` (NULL) — RESUELTO (2026-06-09)**
- Eran pruebas de Sergio (los 4 con email sergio.oropezag@gmail.com). Eliminados de la BD con su aprobación. La BD queda solo con clientes reales: TC-037 a TC-040.

Detalle original:
- 2 del 2026-03-21 y 2 del 2026-05-09, anteriores a los triggers de folio. Si son clientes reales, no tienen folio para referencias futuras.
- Fix: backfill manual con folios de la secuencia, o marcarlos como pruebas.

**O3. Tres triggers duplicados asignan el folio en cada INSERT**
- `trg_assign_tc_folio` (usa secuencia `tc_folio_seq`), `trg_assign_tc_number` y `trigger_set_tc_number` (ambos usan `tc_number_seq`). Se ejecutan en orden alfabético; el último sobreescribe incondicionalmente lo que asignó el primero.
- Resultado: `tc_folio_seq` quema un número por insert que nunca se usa, y hay 3 piezas de lógica para mantener donde debería haber 1. Riesgo de folios saltados/confusos si alguien borra el trigger equivocado.
- Fix: conservar solo `trg_assign_tc_number` (condicional, secuencia correcta) y eliminar los otros dos + `tc_folio_seq`.

**O4. El webhook de Stripe es un no-op en el flujo normal**
- `webhooks/stripe/route.ts` actualiza `payment_status` buscando por `stripe_session_id`, pero la fila solo se crea cuando el cliente envía el formulario — el webhook llega antes y actualiza 0 filas sin error.
- Hoy no rompe nada porque `/api/intake` hardcodea `payment_status: 'confirmed'` (que es el hallazgo S2). Al corregir S2, rediseñar esta pieza junta.

**O5. Reenvío de email de intake no controlado (= S7)**
- El dedupe vive en memoria de la instancia serverless; cada recarga de `/intake?session_id=...` en una instancia fría reenvía el email.
- Fix: marcar el envío en BD.

### VERIFICADO OK
- Templates de email: todos los links se construyen desde `NEXT_PUBLIC_BASE_URL`; sin localhost hardcodeado.
- Vercel Production tiene las 11 variables de entorno, mismos nombres que `.env.local`.
- UNIQUE en `stripe_session_id` y `tc_number`: el doble envío del formulario se bloquea a nivel BD (aunque el usuario recibe un error 500 poco amigable — mejora menor).

## FASE 3 — LANDING + CONSOLA (completa 2026-06-09)

Método: revisión estática de componentes + navegador headless (Edge vía puppeteer-core) sobre producción (`/`, `/checkout`, `/confirmation`, `/admin/login`), con captura de errores de consola, page errors, requests fallidos y respuestas HTTP ≥ 400.

### BAJO

**L1. Atributo SVG inválido en el lienzo del hero**
- `web/components/landing/LienzoOverlay.tsx:17` y `:551` — `preserveAspectRatio="xMidYTop meet"`. El valor `xMidYTop` no existe (el correcto para alinear arriba es `xMidYMin`). El navegador lo ignora y aplica el centrado por defecto, así que el SVG puede no alinearse como se diseñó. Genera 2 errores de consola en la home.

**L2. Google Analytics aparece bloqueado en el test — verificar en el dashboard real**
- Los beacons a `google-analytics.com/g/collect` fallaron con ERR_ABORTED en las 4 páginas. Lo más probable es que sea la prevención de rastreo de Edge headless (el script de GA sí carga). Acción: confirmar en GA4 (tiempo real) que llegan visitas; si llegan, no hay problema.

**L3. GA rastrea también las páginas `/admin`**
- El tag corre en el layout global, así que las visitas del operador al panel contaminan las métricas de la landing.

**L4. Archivos legacy de GitHub Pages en la raíz del repo**
- `index.html`, `form.html`, `mini-reporte.html`, `tucartografia_landing.html`, `tyc_mx.html`, `privacidad_mx.html`, `CNAME` en la raíz. El dominio se sirve 100% desde Vercel (verificado por headers); estos archivos ya no se publican pero confunden el repo. Candidatos a archivarse/eliminarse con aprobación de Sergio.

### VERIFICADO OK
- Consola limpia en las 4 páginas salvo L1 y los beacons de GA.
- Anclas del navbar (`#como-funciona`, `#producto`, `#preguntas`) existen todas.
- PDFs legales del footer funcionan (307 apex→www, luego 200 application/pdf).
- `/`, `/checkout`, `/confirmation`, `/admin/login` cargan sin errores HTTP.
- `/tyc` y `/privacidad` dan 404, pero nada en la app los enlaza (los links legales van a los PDFs) — sin impacto en usuarios.

## FASE 4 — MEJORAS POSIBLES (completa 2026-06-09)

Solo recomendaciones — nada implementado.

### SEO

**M1. No existen `robots.txt` ni `sitemap.xml`** (ambos devuelven 404 en producción). En App Router se agregan con `app/robots.ts` y `app/sitemap.ts`. Sin ellos, Google indexa a ciegas.

**M2. Todas las páginas comparten el mismo `<title>` y description** (definidos solo en `app/layout.tsx`; ninguna página define los suyos). `/checkout`, `/confirmation` e `/intake` muestran el mismo título que la home. Además conviene `noindex` en `/intake`, `/confirmation` y `/admin/*` — son páginas de proceso, no de captación.

**M3. Sin etiqueta canonical, y el `og:url` apunta al apex** (`https://tucartografia.com`) cuando el dominio canónico real es `www` (el apex redirige). Unificar: canonical + og:url con `www`.

**M4. La home tiene dos `<h1>`** (`Hero.tsx:28` y `Producto.tsx:120`). Lo ideal es un solo h1 por página; el segundo debería ser h2.

### PERFORMANCE / FIABILIDAD

**M5. Iconos Phosphor cargados desde unpkg.com** (`layout.tsx:49-50`): dos hojas de estilo render-blocking servidas por un CDN de terceros, sin integridad (SRI). Si unpkg falla o se compromete, afecta al sitio. Recomendación: self-host (el paquete ya se puede instalar vía npm y servir desde `/public`).

**M6. GA corre también en `/admin`** (= L3): excluir las rutas de admin del tag para no contaminar métricas.

### VERIFICADO OK
- Fuentes locales con `font-display: swap`.
- Open Graph y Twitter Card completos con imagen 1200×630.
- `lang="es"` en el HTML; imágenes con `alt`.

---

## RESUMEN DE PENDIENTES (a 2026-06-09)

| ID | Severidad | Tema |
|----|-----------|------|
| S5 | Medio | Bucket `informes` público |
| S7/O5 | Medio | Reenvío de email de intake sin control persistente |
| O3 | Medio | 3 triggers duplicados de folio en BD |
| O4 | Medio | Webhook Stripe es no-op (S2 ya corregido; decidir si el webhook se rediseña o se elimina) |
| L1 | Bajo | SVG `preserveAspectRatio` inválido (LienzoOverlay ×2) |
| L2 | Verificar | Confirmar en GA4 que llegan visitas |
| L4 | Limpieza | Archivos legacy de GitHub Pages en raíz del repo |
| M1-M6 | Mejora | SEO (robots, sitemap, titles, canonical) y self-host de iconos |

Corregido en esta sesión: S1 (falso positivo — `proxy.ts` ya protegía `/admin`), S2 (`/api/intake` ahora verifica el pago contra Stripe y rechaza duplicados), S3 (debug-env eliminada), S4 (auth admin con token firmado, `secure`, tiempo constante y rate limit), S6 (`/api/checkout` ya no filtra `err.message` al cliente), O1 (TC-037–040 marcados delivered), O2 (4 registros de prueba eliminados).

