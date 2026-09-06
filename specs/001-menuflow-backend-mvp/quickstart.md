# Quickstart: MenuFlow Backend MVP

Guía de validación end-to-end de las 4 historias de usuario del
[spec](./spec.md). No repite el diseño de datos ni el contrato de API — ver
[data-model.md](./data-model.md) y [contracts/api.md](./contracts/api.md).

## Prerrequisitos

- Node.js 20 LTS
- PostgreSQL 15 accesible (local o Railway)
- Redis accesible (local o Railway)
- Cuenta de Cloudinary (o S3) para `imageUrl` de productos/logo

## Setup

```bash
npm install
cp .env.example .env
# Completar en .env: DATABASE_URL, REDIS_URL, JWT_SECRET, JWT_EXPIRES_IN=24h,
# CLOUDINARY_URL (o credenciales S3), THROTTLE_ADMIN_LIMIT=100,
# THROTTLE_PUBLIC_LIMIT=1000

npx prisma migrate dev
npm run start:dev   # levanta en http://localhost:3000, Swagger en /api/docs
```

## Validación por historia de usuario

### US1 — Alta de restaurante y autenticación (P1)

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"dueno@latab.com","password":"Secreta123!","tenant":{"name":"La Taberna","slug":"la-taberna"}}'
# Esperado: 201, devuelve accessToken + tenant.slug = "la-taberna"

curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"dueno@latab.com","password":"Secreta123!"}'
# Esperado: 200, accessToken válido

# Repetir el registro con el mismo slug debe fallar (FR-005):
# Esperado: 409 Conflict
```

Guardar el `accessToken` de la respuesta como `$TOKEN` para los siguientes
pasos (`Authorization: Bearer $TOKEN`).

### US2 — Gestión del contenido del menú (P2)

```bash
# Crear categoría (requiere un menú; el registro de US1 no crea uno por
# defecto — crear uno primero)
curl -X POST http://localhost:3000/tenants/$TENANT_ID/menus \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Carta principal"}'

curl -X POST http://localhost:3000/menus/$MENU_ID/categories \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Entrantes"}'

curl -X POST http://localhost:3000/categories/$CATEGORY_ID/products \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Empanadas","price":3500,"dietaryTags":["picante"]}'
# Esperado: 201

curl -X PATCH http://localhost:3000/products/$PRODUCT_ID/availability \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"isAvailable":false}'
# Esperado: 200, cambio inmediato (SC-006)

# Intento de precio inválido (FR-009):
curl -X POST http://localhost:3000/categories/$CATEGORY_ID/products \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Inválido","price":-10}'
# Esperado: 400 Bad Request

# Intento de borrar la categoría con productos activos (FR-027):
curl -X DELETE http://localhost:3000/categories/$CATEGORY_ID \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 409 Conflict
```

### US3 — Vista pública vía QR (P3)

```bash
curl http://localhost:3000/public/menu/la-taberna
# Esperado: 200, incluye "Empanadas" marcado isAvailable=false (FR-016),
# sin necesitar Authorization

# Verificar reflejo de cambios (SC-002): reactivar disponibilidad y volver
# a pedir la vista pública en menos de 5s — debe reflejar isAvailable=true.
curl -X PATCH http://localhost:3000/products/$PRODUCT_ID/availability \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"isAvailable":true}'
curl http://localhost:3000/public/menu/la-taberna

# Filtro por etiqueta (FR-017):
curl "http://localhost:3000/public/menu/la-taberna?tag=picante"
```

### US4 — Código QR (P4)

```bash
curl http://localhost:3000/tenants/$TENANT_ID/qr \
  -H "Authorization: Bearer $TOKEN"
# Esperado: 200, targetUrl apunta a /public/menu/la-taberna

curl "http://localhost:3000/tenants/$TENANT_ID/qr/download?format=png" \
  -H "Authorization: Bearer $TOKEN" -o qr-la-taberna.png
# Esperado: 200, archivo PNG descargado
```

## Criterios de aceptación de esta validación

- Los 4 flujos anteriores se completan sin intervención manual en base de
  datos.
- Ningún paso de US2–US4 requiere volver a autenticar entre pasos dentro de
  la ventana de 24h (FR-002).
- El paso de aislamiento multi-tenant (SC-004) se valida en
  `test/e2e/tenant-isolation.e2e-spec.ts`: crear dos tenants con dos
  usuarios distintos y verificar que el token del usuario A recibe 403/404
  al operar sobre recursos del tenant B.
