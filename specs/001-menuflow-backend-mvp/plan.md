# Implementation Plan: MenuFlow – Plataforma de Gestión de Menús Digitales (MVP)

**Branch**: `001-menuflow-backend-mvp` | **Date**: 2026-09-05 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-menuflow-backend-mvp/spec.md`

## Summary

Backend REST (NestJS 10 + Prisma 5 + PostgreSQL 15) para que un restaurante se
registre, gestione su carta (categorías, productos, precios, disponibilidad,
múltiples menús con horario), publique un código QR, y exponga esa carta en
una vista pública sin login que refleja los cambios del panel casi de
inmediato. Se adapta la arquitectura propuesta en
[RFC_Sistema_Menus_Digitales.md](../../RFC_Sistema_Menus_Digitales.md)
—originalmente pensada para Next.js + Supabase + Vercel— al stack no
negociable de la constitución del proyecto: NestJS/Prisma/JWT-Passport en
lugar de Next.js Server Actions/Supabase Client/Supabase Auth, Cloudinary/S3
en lugar de Supabase Storage, y Redis con invalidación on-change en lugar de
ISR + `revalidateTag` (que es una capacidad específica de Next.js/Vercel sin
equivalente directo en un backend REST desplegado en Railway/Render). El
modelo relacional del RFC (tenants, menus, categories, products, qr_codes) se
conserva conceptualmente, adaptado a convenciones de Prisma y recortado a lo
que el spec de esta feature cubre (sin `subscriptions`/planes ni
`product_translations`, ambos fuera de alcance según las Assumptions del
spec).

## Technical Context

**Language/Version**: TypeScript 5 (strict mode), Node.js 20 LTS

**Primary Dependencies**: NestJS 10 (`@nestjs/core`, `@nestjs/platform-express`),
Prisma 5 (`@prisma/client` + CLI), `@nestjs/passport` + `passport-jwt` +
`@nestjs/jwt`, `class-validator` + `class-transformer`, `@nestjs/swagger`,
`@nestjs/throttler`, `bcrypt`, `qrcode` + `pdf-lib` (generación de QR y
export a PDF), cliente Redis (`@nestjs/cache-manager` + `cache-manager-ioredis-yet`)

**Storage**: PostgreSQL 15 vía Prisma 5 (datos relacionales, con Row Level
Security); Redis (caché de la vista pública del menú, invalidación
on-change); almacenamiento de objetos para imágenes de productos/logos —
Cloudinary o Amazon S3 (decisión a fijar en `research.md`)

**Testing**: Jest (unit tests de servicios) + Supertest (integration tests de
endpoints), cobertura mínima 70% (Constitución, Principio I)

**Target Platform**: Servidor Linux containerizado; hosting en Railway o
Render (decisión a fijar en `research.md`)

**Project Type**: Servicio web backend (API REST) — proyecto único NestJS;
el panel de administración y la vista pública son consumidos por un cliente
externo fuera de este repositorio (ver Assumptions del spec)

**Performance Goals**: 95% de peticiones < 500ms (Constitución III, SC-006);
vista pública del menú servible en < 2s en 4G (SC-003); sostener ≥1000
solicitudes públicas/minuto por tenant sin degradación (SC-007, Constitución
IV)

**Constraints**: JWT con expiración fija de 24h sin excepciones; rate
limiting obligatorio 100 req/min en endpoints admin y 1000 req/min en
endpoints públicos (Constitución IV, vía `@nestjs/throttler`); aislamiento
multi-tenant estricto por `tenant_id` + PostgreSQL RLS, sin vía alternativa
de resolución de tenant (Constitución II); sin borrado físico, solo soft
delete vía campo de estado (Constitución VI); IDs vía `cuid()` de Prisma
(Constitución VI)

**Scale/Scope**: MVP orientado a cientos-miles de restaurantes (RNF de
escalabilidad del PRD); 4 historias de usuario del spec; ~7 módulos NestJS
(auth, tenants, menus, categories, products, qr, public-menu) y ~20
endpoints REST

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Estado | Cómo lo cumple este plan |
|---|---|---|
| I. Stack Tecnológico No Negociable | ✅ PASS | NestJS 10, Prisma 5, JWT+Passport, class-validator/transformer, Swagger, Jest+Supertest — exactamente el stack fijado, sin librerías alternativas que dupliquen responsabilidades |
| II. Aislamiento Multi-Tenant Estricto | ✅ PASS | Todo modelo con datos de negocio lleva `tenantId`; RLS en PostgreSQL como segunda capa; resolución de tenant vía slug en la vista pública y vía membership validada contra el JWT en el panel admin (ver `research.md` para el mecanismo con Prisma) |
| III. Rendimiento y Caché | ✅ PASS | Caché Redis de la respuesta pública del menú por tenant, invalidada explícitamente en cada mutación relevante (crear/editar/eliminar/reordenar categoría o producto, cambio de disponibilidad) — nunca solo por TTL |
| IV. Seguridad por Defecto | ✅ PASS | JWT 24h fijo; `@nestjs/throttler` con configuración diferenciada admin (100/min) vs público (1000/min); `class-validator` en todos los DTOs; Prisma parametriza queries (previene SQL injection); sanitización de campos de texto enriquecido (descripción de producto) |
| V. Estándares de Código | ✅ PASS | TypeScript strict; un módulo NestJS autocontenido por dominio (controller+service+DTOs); unit tests de servicios e integration tests (Supertest) de cada endpoint |
| VI. Integridad y Auditoría de Datos | ✅ PASS | IDs vía `cuid()` de Prisma; eliminación siempre lógica (`status`/`deletedAt` según la entidad); `createdAt`/`updatedAt` en todos los modelos |
| Flujo de Trabajo de Desarrollo | ✅ PASS | Conventional Commits; PRs revisados contra este plan antes de mergear |
| Quality Gates y Cumplimiento | ✅ PASS | Cobertura >70% exigida en CI; Swagger sincronizado con DTOs/controladores en cada PR que toque un endpoint |

No hay violaciones que requieran justificación — ver `Complexity Tracking`
(vacío).

### Re-check post-diseño (Fase 1)

Tras completar `research.md` y `data-model.md`, los puntos que estaban
pendientes de mecanismo concreto quedaron resueltos sin introducir
violaciones nuevas:

- **Principio II** (RLS): mecanismo concreto definido (interceptor que fija
  `app.tenant_id` por request + políticas RLS por tabla, incluyendo la
  desnormalización de `tenantId` en `Category`/`Product` para que la política
  no dependa de un join) — ver `research.md` §4 y `data-model.md` §RLS.
- **Principio III** (caché con invalidación on-change): mecanismo concreto
  definido (cache-aside en Redis por `slug`, invalidado en cada mutación) —
  ver `research.md` §3.
- **Principio IV** (rate limiting diferenciado): mecanismo concreto definido
  (throttlers nombrados `admin`/`public` de `@nestjs/throttler`) — ver
  `research.md` §8.

Gate: ✅ **PASS** — sin cambios de alcance ni excepciones nuevas.

## Project Structure

### Documentation (this feature)

```text
specs/001-menuflow-backend-mvp/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md         # Fase 1
├── quickstart.md         # Fase 1
├── contracts/
│   └── api.md            # Fase 1 — contrato REST
└── tasks.md              # Fase 2 (/speckit-tasks, no generado aquí)
```

### Source Code (repository root)

```text
prisma/
├── schema.prisma         # Modelos + políticas RLS documentadas (ver data-model.md)
└── migrations/

src/
├── main.ts                       # Bootstrap NestJS, Swagger, ValidationPipe global, Throttler
├── app.module.ts
├── common/
│   ├── decorators/                # @CurrentUser, @TenantParam, etc.
│   ├── guards/                    # JwtAuthGuard, TenantMembershipGuard
│   ├── interceptors/              # Interceptor que fija el tenant activo en la sesión de Postgres (RLS)
│   ├── filters/                   # Exception filters (errores consistentes)
│   └── pipes/
├── prisma/
│   ├── prisma.module.ts
│   └── prisma.service.ts          # Cliente Prisma + helper para set tenant context (RLS)
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts         # POST /auth/register, POST /auth/login
│   ├── auth.service.ts
│   ├── strategies/jwt.strategy.ts
│   └── dto/
├── tenants/
│   ├── tenants.module.ts
│   ├── tenants.controller.ts      # POST/GET/PATCH /tenants
│   ├── tenants.service.ts
│   └── dto/
├── menus/
│   ├── menus.module.ts
│   ├── menus.controller.ts        # /tenants/:tenantId/menus
│   ├── menus.service.ts
│   └── dto/
├── categories/
│   ├── categories.module.ts
│   ├── categories.controller.ts   # /menus/:menuId/categories, reorder, duplicate
│   ├── categories.service.ts
│   └── dto/
├── products/
│   ├── products.module.ts
│   ├── products.controller.ts     # /categories/:categoryId/products, availability, reorder, duplicate
│   ├── products.service.ts
│   └── dto/
├── qr/
│   ├── qr.module.ts
│   ├── qr.controller.ts           # /tenants/:tenantId/qr(/tables)
│   ├── qr.service.ts              # genera PNG/PDF con `qrcode` + `pdf-lib`
│   └── dto/
├── media/
│   ├── media.module.ts
│   └── media.service.ts           # upload de imágenes a Cloudinary/S3
└── public-menu/
    ├── public-menu.module.ts
    ├── public-menu.controller.ts  # GET /public/menu/:slug (sin auth)
    └── public-menu.service.ts     # lee de caché Redis, filtra, incrementa contador de vistas

test/
├── unit/                          # o *.spec.ts colocados junto a cada service (convención Nest/Jest)
└── e2e/                           # *.e2e-spec.ts con Supertest, uno por módulo/endpoint crítico
```

**Structure Decision**: Proyecto único (Opción 1), sin separación
backend/frontend porque este repositorio es exclusivamente el backend
(el cliente admin/público vive en otro repositorio, según las Assumptions
del spec). Organización modular de NestJS: un módulo por dominio del
negocio, siguiendo el Principio V de la constitución (módulo autocontenido
con controller+service+DTOs propios).

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla no aplica.*
