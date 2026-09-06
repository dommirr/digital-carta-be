# API Contract (REST): MenuFlow Backend MVP

Formato del contrato: endpoints REST (proyecto = servicio web). El contrato
formal y navegable se genera automáticamente vía Swagger/OpenAPI 3 en
`/api/docs` (Constitución I); este documento es la referencia de diseño que
ese Swagger debe cumplir. Todos los cuerpos de petición se validan con
`class-validator`; todas las respuestas de error siguen el mismo formato:

```json
{ "statusCode": 400, "message": "...", "error": "Bad Request" }
```

Todos los endpoints bajo `/admin` (implícito en las rutas de abajo excepto
`/auth/*` y `/public/*`) requieren `Authorization: Bearer <jwt>` y están
sujetos al throttling "admin" (100 req/min). Las rutas con `:tenantId` (o
anidadas bajo un recurso de un tenant) exigen que el usuario del JWT tenga
una `TenantMembership` activa para ese tenant (`TenantMembershipGuard`,
independientemente del RLS de base de datos — ver research.md, decisión 4).

## Auth

### `POST /auth/register`
Crea la cuenta de administrador **y** su primer restaurante en un solo paso
(US1, AC1). Body: `{ email, password, tenant: { name, slug } }`.
Respuesta 201: `{ accessToken, user: { id, email }, tenant: { id, slug, name } }`.
Error 409 si el email o el slug ya existen (FR-005).

### `POST /auth/login`
Body: `{ email, password }`. Respuesta 200: `{ accessToken }`, válido 24h
(FR-002). Error 401 en credenciales inválidas.

## Tenants

### `POST /tenants`
Crea un restaurante adicional para la cuenta autenticada (FR-003). Body:
`{ name, slug, logoUrl?, brandPrimaryColor?, brandSecondaryColor? }`.
Respuesta 201 con el tenant creado y una `TenantMembership` `ADMIN` para el
usuario autenticado. Error 409 si el slug ya existe.

### `GET /tenants`
Lista los restaurantes que administra el usuario autenticado (según sus
`TenantMembership`).

### `GET /tenants/:tenantId`
Detalle de un restaurante propio.

### `PATCH /tenants/:tenantId`
Actualiza nombre/logo/colores. Solo rol `ADMIN`.

## Menus

### `POST /tenants/:tenantId/menus`
Body: `{ name, isActive?, activeFrom?, activeTo?, activeDays? }` (FR-013).

### `GET /tenants/:tenantId/menus`
Lista los menús (activos y con soft-delete excluido) del restaurante.

### `PATCH /menus/:menuId`
Actualiza nombre, horario o activación manual.

### `DELETE /menus/:menuId`
Soft delete (`status = DELETED`).

## Categories

### `POST /menus/:menuId/categories`
Body: `{ name, sortOrder? }` (FR-007).

### `PATCH /categories/:categoryId`
Actualiza nombre.

### `DELETE /categories/:categoryId`
Soft delete. Responde 409 si existen productos `ACTIVE` asociados (FR-027),
con mensaje indicando mover/eliminar los productos primero.

### `POST /categories/:categoryId/duplicate`
Crea una copia de la categoría (y opcionalmente de sus productos, ver
`duplicateProducts` en el body) (FR-012).

### `PATCH /menus/:menuId/categories/reorder`
Body: `{ orderedCategoryIds: string[] }` (FR-011). Reescribe `sortOrder` de
forma atómica según la posición en el array.

## Products

### `POST /categories/:categoryId/products`
Body: `{ name, description?, price, imageUrl?, allergens?, dietaryTags? }`.
`price` se valida como número positivo (FR-009); error 400 en caso
contrario.

### `PATCH /products/:productId`
Actualiza cualquier campo editable (nombre, descripción, precio, foto,
alérgenos, etiquetas).

### `PATCH /products/:productId/availability`
Body: `{ isAvailable: boolean }` (FR-010). Endpoint dedicado y liviano para
cumplir el guardado percibido como instantáneo (SC-006) y para que, cuando
se implemente la restricción del rol Editor en una fase futura (FR-022), sea
el único endpoint de escritura de producto al que ese rol tenga acceso.

### `DELETE /products/:productId`
Soft delete (FR-024).

### `POST /products/:productId/duplicate`
Crea una copia editable del producto (FR-012).

### `PATCH /categories/:categoryId/products/reorder`
Body: `{ orderedProductIds: string[] }` (FR-011).

## QR

### `GET /tenants/:tenantId/qr`
Devuelve el QR general del restaurante (creado automáticamente junto con el
tenant, FR-019): `{ id, targetUrl, tableIdentifier: null }`.

### `GET /tenants/:tenantId/qr/download?format=png|pdf&size=mesa|atril|cartel`
Descarga binaria del QR general en el formato/tamaño pedido (FR-020).

### `POST /tenants/:tenantId/qr/tables`
Body: `{ tableIdentifier }`. Crea un QR adicional por mesa (FR-021).

### `GET /tenants/:tenantId/qr/tables/:qrCodeId/download?format=png|pdf`
Descarga el QR de una mesa específica.

## Public (sin autenticación)

### `GET /public/menu/:slug`
Vista pública del menú vigente (US3). Query params opcionales:
`?category=<categoryId>&tag=<dietaryTag>&mesa=<tableIdentifier>` (FR-017,
FR-021). Comportamiento:

- 404/mensaje genérico si el tenant no existe, o si `status != ACTIVE`
  (FR-023) — sin distinguir "no existe" de "suspendido" en el mensaje, para
  no filtrar información interna.
- Solo incluye `Menu` vigentes según la regla de horario (FR-015) y sus
  `Category`/`Product` no eliminados.
- Los `Product` con `isAvailable = false` se incluyen igual, marcados como
  no disponibles (FR-016), nunca ocultos.
- Se sirve desde la caché Redis `menu:public:<slug>` (research.md, decisión
  3); en cache miss, se arma la respuesta desde Postgres y se repuebla la
  caché.
- Cada solicitud exitosa incrementa `Tenant.publicViewCount` en +1 (FR-026),
  de forma asíncrona/no bloqueante para no afectar la latencia de respuesta.

Respuesta 200 (forma aproximada):

```json
{
  "tenant": { "name": "...", "logoUrl": "...", "brandPrimaryColor": "..." },
  "menus": [
    {
      "name": "Carta principal",
      "categories": [
        {
          "name": "Entrantes",
          "products": [
            {
              "name": "...", "description": "...", "price": 4500,
              "currency": "ARS", "imageUrl": "...",
              "allergens": [], "dietaryTags": ["vegano"],
              "isAvailable": true
            }
          ]
        }
      ]
    }
  ]
}
```
