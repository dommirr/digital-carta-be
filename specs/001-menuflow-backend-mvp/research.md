# Phase 0 Research: MenuFlow – Plataforma de Gestión de Menús Digitales (MVP)

Todas las decisiones parten de dos fuentes fijas: la
[constitución del proyecto](../../.specify/memory/constitution.md) (stack no
negociable) y el
[RFC_Sistema_Menus_Digitales.md](../../RFC_Sistema_Menus_Digitales.md)
(arquitectura de referencia, originalmente en Next.js/Supabase/Vercel). Cada
entrada resuelve un punto marcado como `NEEDS CLARIFICATION` en el Technical
Context del plan, o adapta una decisión del RFC al stack NestJS/Prisma.

---

## 1. Almacenamiento de imágenes: Cloudinary vs. S3

- **Decision**: Cloudinary.
- **Rationale**: El RFC pedía transformación automática de imágenes (resize
  a tamaño móvil, conversión a WebP/AVIF) para cumplir el requisito de carga
  <2s en 4G (SC-003). Cloudinary ofrece esto out-of-the-box vía URLs de
  transformación (sin infraestructura propia de resize), con un SDK oficial
  para Node/NestJS y un tier gratuito suficiente para el MVP. S3 requeriría
  una capa adicional (Lambda@Edge, Sharp, o un servicio de imágenes aparte)
  para lograr el mismo resultado.
- **Alternatives considered**: Amazon S3 + CloudFront + Sharp — más control
  y potencialmente más barato a gran escala, pero mayor esfuerzo operativo
  inicial (no justificado para un MVP con equipo chico, mismo criterio que
  usó el RFC original para preferir Supabase Storage sobre infraestructura
  propia).

## 2. Hosting: Railway vs. Render

- **Decision**: Railway.
- **Rationale**: Provisión nativa de PostgreSQL y Redis como addons dentro
  del mismo proyecto, despliegue directo desde Git con detección automática
  de NestJS, y un modelo de precios por uso adecuado para un MVP con tráfico
  variable. Reduce la superficie operativa, coherente con el mismo criterio
  de simplicidad que motivó a Vercel+Supabase en el RFC original.
- **Alternatives considered**: Render — comparable en funcionalidad
  (Postgres/Redis gestionados, despliegue por Git), pero con provisioning de
  Redis gestionado únicamente en planes pagos desde el inicio; Railway
  resultó más directo para arrancar el MVP. Puede reevaluarse sin impacto en
  el código de la aplicación (ambos son "solo" plataformas de despliegue).

## 3. Estrategia de actualización de la vista pública ("tiempo real")

- **Decision**: Cache-aside en Redis por tenant, con invalidación explícita
  on-write. Clave `menu:public:<tenantSlug>` con el payload ya armado
  (menús activos + categorías + productos disponibles). Cualquier mutación
  administrativa que afecte el contenido visible (crear/editar/eliminar
  categoría o producto, reordenar, cambiar disponibilidad, activar/desactivar
  un menú) borra esa clave dentro de la misma transacción lógica de guardado.
  La siguiente lectura pública (`GET /public/menu/:slug`) recalcula y
  repuebla la caché.
- **Rationale**: El RFC resolvía esto con ISR + `revalidateTag` de
  Next.js/Vercel, una capacidad específica de ese runtime sin equivalente en
  un backend NestJS desplegado en Railway/Render. El patrón cache-aside con
  invalidación on-change logra el mismo efecto observable (el cliente que
  recién escanea el QR ve el dato vigente de inmediato) y es exactamente lo
  que exige la Constitución (Principio III: "invalidación on-change...no
  invalidación por tiempo como único mecanismo"), satisfaciendo también
  FR-018 y SC-002 (cambio visible en <5s).
- **Alternatives considered**: Polling periódico desde el cliente (mencionado
  en el RFC como complemento) — se deja como decisión de la aplicación
  cliente (fuera de este repo backend), no del servidor; WebSockets/SSE —
  descartado por el mismo motivo que en el RFC (complejidad de
  infraestructura no justificada para el MVP).

## 4. Aislamiento multi-tenant con Prisma + PostgreSQL RLS

- **Decision**: Cada request autenticado pasa por un interceptor que, al
  inicio de la unidad de trabajo, ejecuta `SET LOCAL app.tenant_id = '<uuid>'`
  contra la conexión de Prisma (dentro de una transacción/`$transaction`),
  usando el `tenantId` de la ruta ya validado contra las membresías del
  usuario autenticado. Las políticas RLS de cada tabla con `tenantId`
  verifican `tenant_id = current_setting('app.tenant_id')::uuid`. La vista
  pública usa una ruta separada que fija el tenant resuelto por slug de la
  misma forma, nunca con un rol de base de datos con bypass de RLS.
- **Rationale**: Cumple la Constitución (Principio II) al pie de la letra:
  RLS como segunda capa de defensa real, no decorativa — un bug en el
  filtrado `WHERE tenantId = ...` de un service no alcanza para filtrar datos
  de otro tenant, porque la base de datos también lo bloquea. Prisma no
  aplica RLS automáticamente (a diferencia del cliente de Supabase, que sí
  lo hacía por diseño en el RFC original), por lo que este mecanismo
  reemplaza esa pieza que el RFC obtenía "gratis" de Supabase.
- **Reconciliación con FR-003 (un usuario administra varios restaurantes)**:
  el JWT lleva el `userId` y, para evitar tokens que caduquen en cuanto el
  usuario se una a un nuevo restaurante, las membresías (`tenantId` + rol)
  se validan contra la base en cada request vía un guard
  (`TenantMembershipGuard`) que exige que el `tenantId` de la ruta esté
  entre las membresías activas del usuario del JWT. Esto sigue resolviendo
  el tenant "vía JWT en admin" (el JWT identifica *quién* es y qué tenants
  puede operar) sin fijar un único tenant por token.
- **Alternatives considered**: Aplicar el filtro `tenantId` solo a nivel de
  Prisma (`where: { tenantId }` en cada query) sin RLS — más simple, pero
  viola explícitamente el Principio II de la constitución (RLS es
  obligatorio como segunda capa, no opcional).

## 5. Generación de código QR y export a PDF

- **Decision**: Librería `qrcode` (Node) para generar el QR en PNG/SVG a
  partir de la URL pública (`https://<dominio>/public/menu/<slug>` o con
  `?mesa=<id>` para QR por mesa), y `pdf-lib` para envolver esa imagen en un
  PDF descargable con los tamaños predefinidos que pide el spec (RF-020).
- **Rationale**: Igual que el RFC original, se genera el QR internamente
  (sin dependencia de un servicio externo), mitigando el riesgo de
  disponibilidad de terceros. `qrcode` y `pdf-lib` son librerías Node puras,
  sin dependencias nativas complejas, fáciles de correr en Railway/Render.
- **Alternatives considered**: Servicios externos de generación de QR (ej.
  goqr.me) — descartados por el mismo riesgo de dependencia de terceros que
  identificó el RFC.

## 6. Caché Redis en NestJS

- **Decision**: `@nestjs/cache-manager` con el store `cache-manager-ioredis-yet`,
  expuesto como un `CacheService` propio (no el `CacheInterceptor` genérico
  de Nest) para controlar explícitamente las claves de invalidación descritas
  en la decisión 3.
- **Rationale**: Es la integración oficial recomendada por NestJS 10 para
  Redis, con soporte activo y tipado. Usar un servicio propio en lugar del
  interceptor automático da control fino sobre cuándo invalidar (requisito
  no negociable de la Constitución III), en vez de depender solo de un TTL.
- **Alternatives considered**: `ioredis` directo sin `cache-manager` — viable,
  pero se pierde la abstracción estándar de Nest para swapear el store en
  tests (se puede usar un store en memoria en el entorno de test).

## 7. Identificadores, hashing de contraseñas y borrado lógico

- **Decision**: Todos los modelos usan `@id @default(cuid())` en Prisma
  (siguiendo la literalidad de la Constitución VI: "IDs siempre UUID
  (`cuid()` de Prisma)"). Las contraseñas se hashean con `bcrypt` (costo 10)
  antes de persistirse. Ninguna operación de borrado ejecuta `DELETE`; las
  entidades de negocio (`Category`, `Product`, `Menu`) usan un campo de
  estado (`status: ACTIVE | DELETED`, o `isActive: boolean` según la
  entidad) en vez de eliminarse físicamente.
- **Rationale**: Cumplimiento directo de la Constitución VI.
- **Alternatives considered**: `uuid()` nativo de Prisma en vez de `cuid()`
  — técnicamente ambos son identificadores no secuenciales válidos, pero se
  sigue la redacción explícita de la constitución para no introducir
  ambigüedad.

## 8. Rate limiting diferenciado admin vs. público

- **Decision**: Dos configuraciones de `@nestjs/throttler` registradas como
  "named throttlers": `admin` (100 req/min) aplicado por defecto a todos los
  controladores bajo autenticación JWT, y `public` (1000 req/min) aplicado
  al controlador de `public-menu`. Se implementa como un `ThrottlerGuard`
  por módulo/controlador en vez de uno global, para poder asignar el límite
  correcto según el tipo de endpoint.
- **Rationale**: Cumple literalmente la Constitución IV (100/1000 req/min) y
  reemplaza el equivalente de límites de Vercel/Supabase Edge que asumía el
  RFC original, que no aplican al no usar esa plataforma.
- **Alternatives considered**: Un único throttler global — más simple, pero
  no permite los dos límites distintos que exige la constitución.

## Alcance recortado respecto al RFC original

Estas piezas del modelo del RFC **no** se implementan en este plan, porque
el spec de esta feature las deja fuera explícitamente (sección Assumptions):

- Tabla `subscriptions` / campo `tenants.plan` y límites por plan — no hay
  planes de suscripción ni facturación en este alcance.
- Tabla `product_translations` — el spec asume un único idioma por ahora; el
  modelo de datos no necesita esa tabla todavía, y agregarla después no
  rompe nada (no se requiere ningún diseño especial hoy para "dejarlo
  preparado").
- Tabla `platform_admins` / rol `super_admin` a nivel de fila — el rol
  Super-admin queda fuera de esta especificación (Assumption del spec).
- Aplicación efectiva de restricciones del rol Editor — el spec (FR-022,
  aclarado con el usuario) solo pide modelar el rol, no exigir el guard de
  permisos todavía.
