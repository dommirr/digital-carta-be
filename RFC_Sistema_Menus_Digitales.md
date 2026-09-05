# RFC: Arquitectura Técnica — Sistema de Gestión de Menús Digitales (SaaS Multi-Tenant)

**Estado:** Borrador para revisión
**Autor:** (completar)
**Fecha:** Septiembre 2026
**Documento de referencia:** PRD_Sistema_Menus_Digitales.md v1.0

---

## 1. Resumen

Este RFC propone la arquitectura técnica para implementar el MVP descripto en el PRD: una plataforma SaaS multi-tenant para gestión de menús digitales, con panel de administración, vista pública sin login vía QR, y actualización de contenido reflejada rápidamente en dicha vista pública.

**Stack propuesto:**
- **Frontend + Backend:** Next.js (TypeScript), App Router, desplegado en Vercel.
- **Base de datos:** PostgreSQL gestionado por Supabase.
- **Autenticación:** Supabase Auth.
- **Storage de imágenes:** Supabase Storage.
- **Actualización de vista pública:** revalidación bajo demanda (on-demand ISR) + refresco manual/polling liviano en el cliente. Se deja abierta una ruta de evolución hacia Supabase Realtime/SSE (ver sección 8).
- **Hosting:** Vercel (app) + Supabase (DB, Auth, Storage, Realtime futuro).

Este documento no repite el "qué" ni el "por qué" del producto (ya cubierto en el PRD); se enfoca en el "cómo" técnico.

---

## 2. Objetivos y no-objetivos de esta RFC

### Objetivos
- Definir el modelo de datos concreto (tablas, relaciones, índices) sobre Postgres.
- Definir la estrategia de aislamiento multi-tenant.
- Definir cómo se estructura la app Next.js (rutas, capas, autenticación, roles).
- Definir la estrategia de actualización de la vista pública y su evolución a futuro.
- Definir el manejo de imágenes, QR, y almacenamiento de archivos.
- Mapear los requisitos no funcionales del PRD a decisiones técnicas concretas.

### No-objetivos (fuera de esta RFC)
- Diseño de UI/UX detallado (wireframes, design system).
- Modelo de precios y facturación (se define en RFC de billing, fase 2).
- Analítica de escaneos de QR por mesa (fase 3).
- Pedidos y pagos desde el QR (fase 4).

---

## 3. Arquitectura general

```
┌─────────────────────────────────────────────────────────────┐
│                         Vercel (Edge/Node)                   │
│                                                               │
│   ┌─────────────────────┐      ┌───────────────────────┐    │
│   │  App Admin (/admin)  │      │  App Pública (/m/:slug)│   │
│   │  - CRUD menú          │      │  - Solo lectura        │   │
│   │  - Auth requerida     │      │  - Sin login           │   │
│   │  - Server Actions     │      │  - ISR + revalidateTag │   │
│   └──────────┬───────────┘      └───────────┬────────────┘   │
│              │                              │                │
└──────────────┼──────────────────────────────┼────────────────┘
               │                              │
               ▼                              ▼
        ┌─────────────────────────────────────────┐
        │              Supabase                     │
        │  - Postgres (con RLS por tenant)          │
        │  - Auth (email/pass + OAuth)              │
        │  - Storage (logos, fotos de productos)    │
        │  - Realtime (channels) — fase futura      │
        └─────────────────────────────────────────┘
```

**Justificación de la elección Next.js + Supabase + Vercel:**
- Permite arrancar rápido con un solo proveedor para DB/Auth/Storage, reduciendo superficie operativa para un equipo chico (coherente con el riesgo del PRD de "adopción por usuarios no técnicos" del lado del dueño del local, y con la necesidad de iterar rápido del lado del equipo de producto).
- Row Level Security (RLS) de Postgres resuelve de forma nativa y auditable el requisito crítico de aislamiento multi-tenant (RNF de Seguridad).
- Next.js permite servir la vista pública con estrategias de cacheo/revalidación (ISR) que resuelven el requisito de carga <2s en 4G sin necesidad de un CDN o infraestructura adicional.
- Vercel + Supabase escalan horizontalmente sin gestión manual de servidores, alineado con el RNF de escalabilidad ("miles de locales sin degradación") al menos hasta un umbral razonable para el MVP y fase 2.

---

## 4. Modelo de datos (Postgres/Supabase)

Se traduce el modelo de alto nivel del PRD (sección 8) a un esquema relacional concreto.

```sql
-- Tenants (locales)
create table tenants (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text unique not null,           -- usado en la URL pública /m/:slug
  logo_url text,
  brand_primary_color text,
  brand_secondary_color text,
  default_locale text not null default 'es',
  supported_locales text[] not null default array['es'],
  status text not null default 'active', -- active | suspended | deleted
  plan text not null default 'free',     -- free | pro | business
  created_at timestamptz not null default now()
);

-- Usuarios y su relación N:N con tenants (soporta cadenas, RF-03)
create table tenant_users (
  tenant_id uuid references tenants(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  role text not null check (role in ('admin', 'editor', 'super_admin')),
  primary key (tenant_id, user_id)
);

-- Menús (RF-10: múltiples menús por local, con horario de activación)
create table menus (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references tenants(id) on delete cascade,
  name text not null,               -- "Carta principal", "Brunch"
  is_active boolean not null default true,
  active_from time,                 -- horario de activación (opcional)
  active_to time,
  active_days smallint[],           -- 0=domingo … 6=sábado, null = todos los días
  sort_order int not null default 0,
  created_at timestamptz not null default now()
);

-- Categorías
create table categories (
  id uuid primary key default gen_random_uuid(),
  menu_id uuid not null references menus(id) on delete cascade,
  name text not null,
  sort_order int not null default 0,
  created_at timestamptz not null default now()
);

-- Productos
create table products (
  id uuid primary key default gen_random_uuid(),
  category_id uuid not null references categories(id) on delete cascade,
  name text not null,
  description text,
  price numeric(12,2) not null,
  currency text not null default 'ARS',
  image_url text,
  allergens text[] not null default '{}',
  dietary_tags text[] not null default '{}',  -- vegano, sin_tacc, picante, etc.
  is_available boolean not null default true,
  sort_order int not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- Traducciones de nombre/descripción de producto (soporte multi-idioma, RF-18)
create table product_translations (
  product_id uuid references products(id) on delete cascade,
  locale text not null,
  name text,
  description text,
  primary key (product_id, locale)
);

-- Códigos QR (RF-11, RF-12)
create table qr_codes (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references tenants(id) on delete cascade,
  table_identifier text,           -- null = QR general del local
  target_url text not null,
  created_at timestamptz not null default now()
);

-- Suscripciones/planes (esqueleto; detalle en RFC de billing)
create table subscriptions (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references tenants(id) on delete cascade,
  plan text not null,
  status text not null,            -- active | past_due | canceled
  product_limit int,
  location_limit int,
  created_at timestamptz not null default now()
);
```

**Índices recomendados:**
- `tenants(slug)` único, para resolver la vista pública en O(1).
- `categories(menu_id, sort_order)`, `products(category_id, sort_order)` para lecturas ordenadas sin ordenar en memoria.
- `tenant_users(user_id)` para resolver rápido "a qué tenants tiene acceso este usuario".

---

## 5. Multi-tenancy y aislamiento de datos

El PRD marca esto como un riesgo crítico (sección 11). Se propone:

1. **Aislamiento a nivel de base de datos con Row Level Security (RLS):** cada tabla con `tenant_id` (directo o vía join) tiene políticas RLS que solo permiten acceso a filas donde `tenant_id` esté en la lista de tenants del usuario autenticado (`tenant_users`). Esto es la defensa principal, independiente de bugs en la capa de aplicación.
2. **La vista pública nunca usa credenciales de usuario:** se resuelve con una clave de servicio (`service_role`) del lado del servidor de Next.js, filtrando explícitamente por `slug` del tenant, y sólo exponiendo los campos necesarios para el menú público (nunca datos de facturación, usuarios u otros tenants).
3. **Testing de aislamiento:** se recomienda incluir tests automatizados que verifiquen que un usuario del Tenant A no puede leer/escribir datos del Tenant B, como parte del pipeline de CI antes de cualquier release.

---

## 6. Estructura de la aplicación Next.js

```
/app
  /admin
    /(auth)/login
    /(dashboard)/[tenantSlug]/menu        -> CRUD categorías/productos
    /(dashboard)/[tenantSlug]/qr          -> generación/descarga de QR
    /(dashboard)/[tenantSlug]/settings    -> datos del local, idiomas, marca
  /m/[tenantSlug]                         -> vista pública del menú (RF-15..19)
  /api
    /webhooks/...                         -> (futuro: pagos, integraciones)
/lib
  /supabase (clients: server, browser, admin/service-role)
  /permissions (helpers de rol: admin vs editor)
  /qr (generación de QR, PNG/PDF)
```

- **Server Actions** para las mutaciones del panel admin (crear/editar producto, marcar agotado, reordenar), evitando exponer una API REST/GraphQL propia en el MVP.
- **Roles (RF-20):** un helper central `requireRole(tenantId, ['admin'])` que se invoca al inicio de cada Server Action sensible (precio, estructura) y `requireRole(tenantId, ['admin','editor'])` para las acciones de disponibilidad.
- **Super-admin (RF-21):** panel separado en `/admin/platform`, accesible solo a usuarios con rol `super_admin` en cualquier fila de `tenant_users`, o mejor, en una tabla separada `platform_admins` para no mezclar el modelo de permisos por tenant con el permiso global de plataforma.

---

## 7. Vista pública del menú

- **Ruta:** `/m/[tenantSlug]` (y opcionalmente `?mesa=ID` para QR por mesa, RF-12, sin lógica adicional en el MVP más que registrar el identificador si se quiere dejar preparado para analítica futura).
- **Renderizado:** Server Components de Next.js, con datos leídos directamente de Postgres en el servidor (sin exponer llamadas a la DB desde el cliente).
- **Filtros por categoría/etiqueta (RF-17):** implementados client-side sobre el payload ya cargado (el volumen de productos de un menú de restaurante es chico, no requiere re-fetch al servidor por cada filtro).
- **Idioma (RF-18):** selector que cambia el locale activo; el server component resuelve `product_translations` para ese locale, con fallback al `default_locale` del tenant si falta una traducción.
- **Rendimiento (<2s en 4G):** imágenes servidas vía Supabase Storage con transformación/resize automático a un tamaño máximo adecuado para mobile, formato moderno (WebP/AVIF) y `next/image` para lazy loading y `srcset` automático.

---

## 8. Estrategia de actualización de contenido ("tiempo real")

Se confirmó con el equipo que, para el MVP, **no es necesario un canal de tiempo real verdadero (WebSockets/SSE)**. Se prioriza simplicidad operativa con un mecanismo de refresco/revalidación, dejando el diseño abierto para evolucionar sin reescribir la vista pública.

### 8.1 Enfoque MVP
- La vista pública se sirve con **ISR (Incremental Static Regeneration) + revalidación bajo demanda**:
  - Cuando el admin guarda un cambio (precio, disponibilidad, reordenamiento), el Server Action ejecuta `revalidateTag('menu:<tenantId>')`.
  - Esto invalida el caché de esa página específica en Vercel, y la próxima visita (o un F5 del cliente) sirve el contenido actualizado ya regenerado, sin esperar un ciclo de build.
  - Es funcionalmente equivalente a "tiempo real" desde la perspectiva del comensal que recién escanea el QR, ya que el caché se invalida en el momento del guardado, no en un intervalo fijo.
- Para el caso de un comensal que ya tiene el menú abierto en su navegador y el precio cambia mientras está mirando: se acepta que necesite refrescar la página (F5) o que un pequeño polling liviano (por ejemplo, cada 30–60s, solo mientras la pestaña está en foco) chequee un `updated_at` agregado del menú y dispare un refresh automático si detecta cambios. Esto cubre el caso de uso sin necesidad de infraestructura de sockets.

### 8.2 Ruta de evolución (si en el futuro se requiere tiempo real estricto)
- Supabase ofrece **Realtime** sobre Postgres (basado en `logical replication`), que permitiría suscribir la vista pública a cambios en `products`/`categories` de un tenant específico sin cambiar el modelo de datos.
- La migración sería: reemplazar el polling liviano por una suscripción a un canal `realtime:menu:<tenantId>` desde un Client Component, mantiedo el resto de la arquitectura (RLS, Server Components para la carga inicial) sin cambios.
- Esto se deja documentado pero **no se implementa en el MVP**, evitando complejidad prematura.

---

## 9. Generación y publicación de QR (RF-11 a RF-14)

- Generación **interna** (no depender de un servicio externo, mitigando el riesgo mencionado en el PRD sección 11), usando una librería de generación de QR en el servidor (ej. `qrcode` en Node) a partir de la URL pública (`https://dominio.com/m/<slug>` o con `?mesa=<id>` para QR por mesa).
- El QR se genera on-demand en un endpoint de Next.js y se ofrece para descarga en PNG y PDF, en al menos 3 tamaños predefinidos (mesa, atril, cartel).
- Como el QR apunta siempre a la misma URL y el contenido se resuelve dinámicamente en el servidor (sección 8), **nunca es necesario regenerar el QR ante un cambio de menú** (cumple RF-14 por diseño).

---

## 10. Seguridad

| Requisito (PRD) | Decisión técnica |
|---|---|
| Aislamiento entre tenants | RLS en Postgres (sección 5) + tests de aislamiento en CI |
| Cifrado en tránsito | HTTPS por defecto en Vercel/Supabase |
| Cifrado en reposo | Provisto por Supabase (Postgres gestionado) |
| Autenticación | Supabase Auth (email/password + OAuth social para RF-01) |
| Roles y permisos | Chequeo de rol en cada Server Action sensible (sección 6), nunca solo en el cliente |
| Suspensión de cuentas (RF-04) | Campo `tenants.status`; middleware que bloquea acceso al panel admin si `status != 'active'`, sin afectar la vista pública mientras se define política comercial |

---

## 11. Accesibilidad e internacionalización

- **WCAG AA (RNF):** uso de componentes accesibles (roles ARIA, contraste de color validado incluso cuando el tenant define sus propios colores de marca — se recomienda validar contraste automáticamente y aplicar un color de respaldo si el definido por el admin no cumple el mínimo).
- **i18n:** el esquema de datos ya soporta multi-idioma desde el diseño (`product_translations`, `tenants.supported_locales`), cumpliendo el RNF de internacionalización "desde el diseño de datos" sin necesidad de migración futura.
- **Multi-moneda:** el campo `products.currency` ya está contemplado a nivel de esquema aunque el MVP solo lo exponga para un mercado inicial (a definir, pregunta abierta del PRD sección 13).

---

## 12. Plan de despliegue

- **Entornos:** `preview` (deploys automáticos de Vercel por PR) y `production`, cada uno contra su propio proyecto de Supabase para evitar mezclar datos de prueba con datos reales de tenants.
- **CI:** lint + typecheck + tests (incluyendo tests de aislamiento RLS) en cada PR antes de merge.
- **Migraciones de base de datos:** gestionadas con las migraciones de Supabase CLI, versionadas en el repo.
- **Dominio:** se recomienda subdominio por tenant o path-based (`/m/:slug`) para el MVP — se propone **path-based** por simplicidad de certificados/DNS, dejando subdominios (`local.tuapp.com`) como posible mejora de marca en fase 2.

---

## 13. Alternativas consideradas

| Alternativa | Por qué no se eligió para el MVP |
|---|---|
| Backend propio (Node/Express o NestJS) + Postgres autogestionado | Mayor esfuerzo operativo (hosting de DB, backups, migraciones) sin beneficio claro para el tamaño del MVP |
| Firebase/Firestore | Modelo NoSQL dificulta las relaciones jerárquicas menú→categoría→producto y los reportes futuros (fase 3/4); RLS de Postgres da mejor garantía de aislamiento multi-tenant auditable |
| WebSockets/SSE desde el día uno | Complejidad de infraestructura innecesaria dado que el negocio confirmó que polling/F5 es aceptable para el MVP (sección 8) |

---

## 14. Preguntas abiertas (a resolver antes de implementar)

- Mercado geográfico inicial → define moneda por defecto y normativa de datos aplicable (ver PRD sección 13).
- ¿El dominio será propio de la plataforma (`tuapp.com/m/slug`) o se contempla dominio propio del cliente en el plan Business? Impacta el diseño de multi-tenancy por dominio a futuro.
- Política exacta de suspensión por falta de pago (RF-04): ¿se apaga también la vista pública o solo el panel admin?
- Definición de intervalo de polling liviano (sección 8.1): valor sugerido 30–60s, a validar con UX.

---

*Este RFC depende del PRD_Sistema_Menus_Digitales.md v1.0 y debe revisarse si dicho documento cambia de alcance.*
