# Phase 1 Data Model: MenuFlow – Plataforma de Gestión de Menús Digitales (MVP)

Adaptación a Prisma 5 / PostgreSQL 15 del modelo relacional del
[RFC](../../RFC_Sistema_Menus_Digitales.md#4-modelo-de-datos-postgressupabase),
recortado según las Assumptions del [spec](./spec.md) (ver `research.md`,
sección "Alcance recortado"). Todas las entidades de negocio cumplen la
Constitución: `id` vía `cuid()`, `createdAt`/`updatedAt`, y borrado lógico
(nunca `DELETE`).

## Entidades

### User (cuenta de administrador)

Representa a la persona que se autentica (FR-001, FR-002).

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| email | String | único, usado para login |
| passwordHash | String | bcrypt, nunca se expone en respuestas |
| createdAt / updatedAt | DateTime | auditoría |

Relación: `User` 1—N `TenantMembership` (un usuario puede administrar varios
restaurantes, FR-003).

### Tenant (Restaurante)

FR-004, FR-005, FR-006, FR-023.

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| name | String | nombre del local |
| slug | String | único; resuelve la URL pública (FR-005) |
| logoUrl | String? | URL en Cloudinary/S3 |
| brandPrimaryColor | String? | |
| brandSecondaryColor | String? | |
| status | Enum: `ACTIVE` \| `SUSPENDED` \| `DELETED` | controla acceso al panel y a la vista pública (FR-023); default `ACTIVE` (Assumption) |
| publicViewCount | Int | contador acumulado de visualizaciones del menú público (FR-026, aclarado como conteo total sin desglose); incrementado atómicamente en cada `GET` público |
| createdAt / updatedAt | DateTime | |

Relación: `Tenant` 1—N `TenantMembership`, 1—N `Menu`, 1—N `QrCode`.

**Validación**: `slug` se normaliza (minúsculas, sin espacios) y se valida
como único antes de crear (FR-005); intento de duplicado → error de
validación, no excepción no controlada.

### TenantMembership (rol de un usuario en un restaurante)

FR-003, FR-022.

| Campo | Tipo | Notas |
|---|---|---|
| userId | String (cuid) | FK → User |
| tenantId | String (cuid) | FK → Tenant |
| role | Enum: `ADMIN` \| `EDITOR` | modelado ahora; la restricción de permisos del rol `EDITOR` se aplica en una fase posterior (FR-022 aclarado) |
| createdAt | DateTime | |

Clave primaria compuesta `(userId, tenantId)`. No incluye `SUPER_ADMIN`
(fuera de alcance, ver research.md).

### Menu

FR-013, FR-015.

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| tenantId | String (cuid) | FK → Tenant |
| name | String | ej. "Carta principal", "Brunch" |
| isActive | Boolean | activación manual; default `true` |
| activeFrom | Time? | horario de activación (opcional) |
| activeTo | Time? | |
| activeDays | Int[]? | 0=domingo…6=sábado; `null`/vacío = todos los días |
| sortOrder | Int | default 0 |
| status | Enum: `ACTIVE` \| `DELETED` | soft delete |
| createdAt / updatedAt | DateTime | |

**Regla de negocio (FR-015)**: un menú se considera "vigente" para la vista
pública si `isActive = true` y, cuando tiene horario configurado, la hora y
día actuales del tenant caen dentro de `activeFrom`–`activeTo` /
`activeDays`.

### Category

FR-007, FR-011, FR-012, FR-027.

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| menuId | String (cuid) | FK → Menu |
| name | String | |
| sortOrder | Int | default 0 (FR-011) |
| status | Enum: `ACTIVE` \| `DELETED` | soft delete |
| createdAt / updatedAt | DateTime | |

**Regla de negocio (FR-027)**: eliminar (soft-delete) una categoría se
rechaza si existen `Product` con `status = ACTIVE` referenciando esta
categoría; el llamador debe mover o eliminar esos productos primero.

### Product

FR-008, FR-009, FR-010, FR-011, FR-012, FR-016.

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| categoryId | String (cuid) | FK → Category |
| name | String | |
| description | String? | |
| price | Decimal(12,2) | validado > 0 antes de guardar (FR-009) |
| currency | String | default configurable por tenant/mercado; no forma parte del alcance funcional de este MVP (ver Assumptions) |
| imageUrl | String? | URL en Cloudinary/S3; ausente = se usa imagen de reemplazo en la vista pública (edge case del spec) |
| allergens | String[] | |
| dietaryTags | String[] | ej. `vegano`, `sin_tacc`, `picante` (FR-017) |
| isAvailable | Boolean | default `true`; toggle inmediato (FR-010) |
| sortOrder | Int | default 0 (FR-011) |
| status | Enum: `ACTIVE` \| `DELETED` | soft delete (FR-024) |
| createdAt / updatedAt | DateTime | |

### QrCode

FR-019, FR-020, FR-021.

| Campo | Tipo | Notas |
|---|---|---|
| id | String (cuid) | PK |
| tenantId | String (cuid) | FK → Tenant |
| tableIdentifier | String? | `null` = QR general del local; valor = QR de mesa (FR-021) |
| targetUrl | String | URL pública del menú, con `?mesa=<id>` si aplica |
| createdAt | DateTime | |

Se genera automáticamente un `QrCode` con `tableIdentifier = null` al crear
un `Tenant` (FR-019); los QR de mesa se crean bajo demanda.

## Row Level Security (RLS)

Aplicable a `Tenant` (por `id`), `Menu`, `Category` (vía `menuId` → `Menu.tenantId`),
`Product` (vía `categoryId` → `Category` → `Menu.tenantId`), `QrCode` y
`TenantMembership` (por `tenantId`). Política estándar por tabla:

```sql
using (tenant_id = current_setting('app.tenant_id', true)::text)
```

(para `Category`/`Product`, la política se define sobre una columna
`tenant_id` desnormalizada — ver nota abajo — para evitar políticas RLS con
joins, que PostgreSQL no soporta de forma eficiente).

**Nota de desnormalización deliberada**: `Category` y `Product` incluyen una
columna adicional `tenantId` (redundante respecto a la cadena
`menuId`→`categoryId`) exclusivamente para permitir una política RLS directa
`tenant_id = current_setting(...)`. Se mantiene sincronizada por el
`PrismaService` al crear/mover una categoría o producto (nunca se expone en
los DTOs de entrada/salida de la API; es un detalle interno de aislamiento).

## Diagrama de relaciones

```
User ──< TenantMembership >── Tenant ──< Menu ──< Category ──< Product
                                  └──< QrCode
```

## Referencia cruzada con Requisitos Funcionales

| Entidad | FRs cubiertos |
|---|---|
| User, TenantMembership | FR-001, FR-002, FR-003, FR-022 |
| Tenant | FR-004, FR-005, FR-006, FR-023, FR-026 |
| Menu | FR-013, FR-015 |
| Category | FR-007, FR-011, FR-012, FR-027 |
| Product | FR-008, FR-009, FR-010, FR-011, FR-012, FR-016, FR-024 |
| QrCode | FR-019, FR-020, FR-021 |
| (todas) | FR-025 (`createdAt`/`updatedAt`) |
