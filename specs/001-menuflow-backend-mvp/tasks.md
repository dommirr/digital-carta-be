---

description: "Task list template for feature implementation"
---

# Tasks: MenuFlow – Plataforma de Gestión de Menús Digitales (MVP)

**Input**: Design documents from `/specs/001-menuflow-backend-mvp/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/api.md](./contracts/api.md), [quickstart.md](./quickstart.md)

**Tests**: Incluidos. La [constitución del proyecto](../../.specify/memory/constitution.md)
(Principio V, Quality Gates) exige unit tests obligatorios para servicios e
integration tests para endpoints con cobertura >70%; no son opcionales para
este proyecto.

**Organization**: Las tareas se agrupan por historia de usuario (P1–P4 del
spec) para permitir implementación y prueba independiente de cada una.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias)
- **[Story]**: Historia de usuario a la que pertenece (US1, US2, US3, US4)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Proyecto único NestJS (ver `plan.md` → Project Structure): `src/<módulo>/`,
`prisma/`, `test/e2e/`. Los unit tests se colocan junto al archivo que
prueban como `*.spec.ts` (convención Jest/NestJS).

**Scaffolding**: el proyecto y cada módulo/controller/service/guard/filter/
decorator se generan con los schematics oficiales del Nest CLI
(`nest g ...`, ver https://docs.nestjs.com/cli/overview) — nunca creando los
archivos manualmente uno por uno. Los DTOs (simples clases) se generan con
`nest g class <ruta>/<nombre>.dto --no-spec --flat`. Se usa `--no-spec` en
todos los generadores de servicios/controllers/guards/filters/decorators
porque los tests correspondientes se escriben antes, como tareas explícitas
de "Tests for User Story N" (TDD), y no deben pisarse con el spec de
placeholder que el CLI generaría por defecto. `nest g module <nombre>`
registra automáticamente el módulo en `src/app.module.ts`; las tareas que
antes decían "registrar" ahora dicen "verificar" ese registro automático.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Inicialización del proyecto NestJS (vía Nest CLI) y sus dependencias

- [ ] T001 Generar el proyecto con el Nest CLI (`npx @nestjs/cli new . --package-manager npm --strict`) en lugar de crear la estructura archivo por archivo — produce `src/main.ts`, `src/app.module.ts`, `test/`, `tsconfig.json` (strict), `.eslintrc.js`, `.prettierrc` según `plan.md` → Project Structure y https://docs.nestjs.com/cli/overview
- [ ] T002 Sobre el proyecto generado por el CLI, instalar vía `npm install` las dependencias adicionales del stack: `@prisma/client`+`prisma`, `@nestjs/passport`+`passport-jwt`+`@nestjs/jwt`, `class-validator`+`class-transformer`, `@nestjs/swagger`, `@nestjs/throttler`, `bcrypt`, `qrcode`, `pdf-lib`, `@nestjs/cache-manager`+`cache-manager-ioredis-yet`
- [ ] T003 [P] Ajustar el ESLint/Prettier generados por el Nest CLI a las convenciones del proyecto (TypeScript strict mode) en `.eslintrc.js`/`.prettierrc`
- [ ] T004 [P] Crear `.env.example` con `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN=24h`, `CLOUDINARY_URL`, `THROTTLE_ADMIN_LIMIT=100`, `THROTTLE_PUBLIC_LIMIT=1000` según `quickstart.md`
- [ ] T005 Inicializar Prisma con su CLI oficial (`npx prisma init --datasource-provider postgresql`), que genera `prisma/schema.prisma` y agrega `DATABASE_URL` a `.env`
- [ ] T006 [P] Configurar bootstrap de Swagger/OpenAPI 3 (`SwaggerModule`, `DocumentBuilder`) en `src/main.ts`, expuesto en `/api/docs` (Constitución I)
- [ ] T007 [P] Configurar `ValidationPipe` global (`whitelist`, `transform`, `forbidNonWhitelisted`) en `src/main.ts`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infraestructura núcleo que TODAS las historias de usuario necesitan

**⚠️ CRITICAL**: Ninguna historia de usuario puede empezar hasta completar esta fase

- [ ] T008 Definir el schema completo de Prisma en `prisma/schema.prisma`: modelos `User`, `Tenant`, `TenantMembership`, `Menu`, `Category`, `Product`, `QrCode` y enums `TenantStatus`, `MembershipRole`, `EntityStatus`, todos con `id` vía `cuid()` y `createdAt`/`updatedAt` (Constitución VI, `data-model.md`)
- [ ] T009 Generar y aplicar la migración inicial de Prisma (`prisma/migrations/`) a partir del schema de T008
- [ ] T010 [P] Generar `PrismaModule`/`PrismaService` con el Nest CLI (`nest g module prisma && nest g service prisma --no-spec`) e implementarlos en `src/prisma/prisma.module.ts` y `src/prisma/prisma.service.ts`, incluyendo el helper para fijar `app.tenant_id` por request (`SET LOCAL`) descrito en `research.md` §4
- [ ] T011 [P] Escribir la migración SQL de políticas RLS para `Tenant`, `Menu`, `Category`, `Product`, `QrCode`, `TenantMembership` en `prisma/migrations/` según `data-model.md` → Row Level Security (incluye las columnas `tenantId` desnormalizadas en `Category`/`Product`)
- [ ] T012 [P] Generar `AuthModule` con el Nest CLI (`nest g module auth`) e implementar `JwtStrategy` en `src/auth/strategies/jwt.strategy.ts` (expiración 24h, Constitución IV)
- [ ] T013 [P] Generar los guards con el Nest CLI (`nest g guard common/guards/jwt-auth --no-spec && nest g guard common/guards/tenant-membership --no-spec`) e implementar `JwtAuthGuard` y `TenantMembershipGuard`
- [ ] T014 [P] Generar el decorator con el Nest CLI (`nest g decorator common/decorators/current-user --no-spec`) e implementar `@CurrentUser()`
- [ ] T015 [P] Generar el filter con el Nest CLI (`nest g filter common/filters/http-exception --no-spec`) e implementar el manejo de errores consistente
- [ ] T016 [P] Configurar `@nestjs/throttler` con throttlers nombrados `admin` (100 req/min) y `public` (1000 req/min) en `src/app.module.ts` (Constitución IV, `research.md` §8)
- [ ] T017 [P] Generar `CacheModule`/`CacheService` con el Nest CLI (`nest g module cache && nest g service cache --no-spec`) e implementarlos sobre `@nestjs/cache-manager` + `cache-manager-ioredis-yet`, con métodos `getPublicMenu(slug)`/`invalidatePublicMenu(slug)` (`research.md` §3 y §6)
- [ ] T018 [P] Generar `MediaModule`/`MediaService` con el Nest CLI (`nest g module media && nest g service media --no-spec`) e implementar el upload a Cloudinary (`research.md` §1)

**Checkpoint**: Infraestructura lista — las historias de usuario pueden implementarse (en paralelo si hay equipo)

---

## Phase 3: User Story 1 - Alta de restaurante y autenticación (Priority: P1) 🎯 MVP

**Goal**: Una persona puede registrarse, crear su restaurante con una URL pública única, iniciar sesión, y administrar más de un restaurante desde la misma cuenta.

**Independent Test**: Registrar una cuenta nueva, iniciar sesión y verificar que el restaurante queda creado con un slug único — sin depender de ninguna otra historia (ver `quickstart.md` → US1).

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar antes de implementar.

- [ ] T019 [P] [US1] Integration test `POST /auth/register` (éxito, email duplicado, slug duplicado FR-005) en `test/e2e/auth-register.e2e-spec.ts`
- [ ] T020 [P] [US1] Integration test `POST /auth/login` (éxito, credenciales inválidas) en `test/e2e/auth-login.e2e-spec.ts`
- [ ] T021 [P] [US1] Integration test `POST /tenants` (segundo restaurante para la misma cuenta, FR-003) en `test/e2e/tenants-create.e2e-spec.ts`
- [ ] T022 [P] [US1] Unit test de `AuthService` (hash de contraseña, emisión de JWT 24h) en `src/auth/auth.service.spec.ts`
- [ ] T023 [P] [US1] Unit test de `TenantsService` (validación de unicidad de slug) en `src/tenants/tenants.service.spec.ts`

### Implementation for User Story 1

- [ ] T024 [P] [US1] Generar `RegisterDto`/`LoginDto` con el Nest CLI (`nest g class auth/dto/register.dto --no-spec --flat && nest g class auth/dto/login.dto --no-spec --flat`) y completar sus reglas de `class-validator`
- [ ] T025 [US1] Generar `AuthService` con el Nest CLI (`nest g service auth --no-spec`) e implementar el registro (crea `User`+`Tenant`+`TenantMembership ADMIN` en una transacción, hash bcrypt) y el login (valida credenciales, emite JWT 24h) en `src/auth/auth.service.ts`
- [ ] T026 [US1] Generar `AuthController` con el Nest CLI (`nest g controller auth --no-spec`) e implementar `POST /auth/register` y `POST /auth/login` en `src/auth/auth.controller.ts`
- [ ] T027 [P] [US1] Generar `CreateTenantDto`/`UpdateTenantDto` con el Nest CLI (`nest g class tenants/dto/create-tenant.dto --no-spec --flat && nest g class tenants/dto/update-tenant.dto --no-spec --flat`)
- [ ] T028 [US1] Generar `TenantsModule`/`TenantsService` con el Nest CLI (`nest g module tenants && nest g service tenants --no-spec`) e implementar: crear restaurante adicional + membership, listar restaurantes propios, obtener/actualizar, validar unicidad de slug FR-005, aislamiento FR-006, en `src/tenants/tenants.service.ts`
- [ ] T029 [US1] Generar `TenantsController` con el Nest CLI (`nest g controller tenants --no-spec`) e implementar `POST/GET/PATCH /tenants`, `GET /tenants/:tenantId`, protegido por `JwtAuthGuard`+`TenantMembershipGuard`, en `src/tenants/tenants.controller.ts`
- [ ] T030 [US1] Verificar que `AuthModule` y `TenantsModule` quedaron registrados en `src/app.module.ts` (automático al generarlos con `nest g module` en T012/T028)

**Checkpoint**: User Story 1 funcional y probable de forma independiente

---

## Phase 4: User Story 2 - Gestión del contenido del menú (Priority: P2)

**Goal**: El administrador crea/edita/elimina categorías y productos, cambia precio y disponibilidad, reordena y duplica, y gestiona múltiples menús con horario de activación.

**Independent Test**: Con una cuenta ya autenticada (US1), crear una categoría, agregar un producto completo, editar su precio y disponibilidad, y verificar que persiste — sin depender de la vista pública ni del QR (ver `quickstart.md` → US2).

### Tests for User Story 2 ⚠️

- [ ] T031 [P] [US2] Integration test CRUD de menús (`POST/GET/PATCH/DELETE /menus`) y activación por horario (FR-013, FR-015) en `test/e2e/menus.e2e-spec.ts`
- [ ] T032 [P] [US2] Integration test CRUD de categorías + reorder + duplicate + bloqueo de borrado con productos activos (FR-007, FR-011, FR-012, FR-027) en `test/e2e/categories.e2e-spec.ts`
- [ ] T033 [P] [US2] Integration test CRUD de productos + toggle de disponibilidad + reorder + duplicate + precio inválido (FR-008, FR-009, FR-010, FR-011, FR-012) en `test/e2e/products.e2e-spec.ts`
- [ ] T034 [P] [US2] Unit test de `MenusService` (regla de vigencia por horario/días, FR-015) en `src/menus/menus.service.spec.ts`
- [ ] T035 [P] [US2] Unit test de `CategoriesService` (regla de bloqueo de borrado, FR-027) en `src/categories/categories.service.spec.ts`
- [ ] T036 [P] [US2] Unit test de `ProductsService` (validación de precio positivo, duplicado) en `src/products/products.service.spec.ts`

### Implementation for User Story 2

- [ ] T037 [P] [US2] Generar los DTOs de `Menu` con el Nest CLI (`nest g class menus/dto/create-menu.dto --no-spec --flat && nest g class menus/dto/update-menu.dto --no-spec --flat`)
- [ ] T038 [US2] Generar `MenusModule`/`MenusService` con el Nest CLI (`nest g module menus && nest g service menus --no-spec`) e implementar CRUD, soft delete, lógica de activación por horario/días FR-015, en `src/menus/menus.service.ts`
- [ ] T039 [US2] Generar `MenusController` con el Nest CLI (`nest g controller menus --no-spec`) e implementar `/tenants/:tenantId/menus`, `/menus/:menuId`, en `src/menus/menus.controller.ts`
- [ ] T040 [P] [US2] Generar los DTOs de `Category` con el Nest CLI (`nest g class categories/dto/create-category.dto --no-spec --flat && nest g class categories/dto/update-category.dto --no-spec --flat && nest g class categories/dto/reorder-categories.dto --no-spec --flat`)
- [ ] T041 [US2] Generar `CategoriesModule`/`CategoriesService` con el Nest CLI (`nest g module categories && nest g service categories --no-spec`) e implementar CRUD, duplicar, reordenar, bloqueo de borrado con productos activos FR-027, en `src/categories/categories.service.ts`
- [ ] T042 [US2] Generar `CategoriesController` con el Nest CLI (`nest g controller categories --no-spec`) e implementarlo en `src/categories/categories.controller.ts`
- [ ] T043 [P] [US2] Generar los DTOs de `Product` con el Nest CLI (`nest g class products/dto/create-product.dto --no-spec --flat && nest g class products/dto/update-product.dto --no-spec --flat && nest g class products/dto/update-availability.dto --no-spec --flat && nest g class products/dto/reorder-products.dto --no-spec --flat`)
- [ ] T044 [US2] Generar `ProductsModule`/`ProductsService` con el Nest CLI (`nest g module products && nest g service products --no-spec`) e implementar CRUD, validación de precio positivo FR-009, toggle de disponibilidad FR-010, duplicar, reordenar, soft delete FR-024, en `src/products/products.service.ts`
- [ ] T045 [US2] Generar `ProductsController` con el Nest CLI (`nest g controller products --no-spec`) e implementar (incluye `PATCH /products/:productId/availability`) en `src/products/products.controller.ts`
- [ ] T046 [US2] Verificar que `MenusModule`, `CategoriesModule`, `ProductsModule` quedaron registrados en `src/app.module.ts` (automático al generarlos con `nest g module` en T038/T041/T044)
- [ ] T047 [US2] Invocar `CacheService.invalidatePublicMenu(slug)` al final de cada mutación relevante en `MenusService`, `CategoriesService` y `ProductsService` (Constitución III, `research.md` §3)

**Checkpoint**: User Stories 1 y 2 funcionan de forma independiente

---

## Phase 5: User Story 3 - Vista pública del menú vía QR (Priority: P3)

**Goal**: Cualquier persona puede ver el menú publicado de un restaurante sin login, filtrar por categoría/etiqueta, y ver reflejados los cambios del administrador casi de inmediato.

**Independent Test**: Con un restaurante ya cargado (US2), pedir `GET /public/menu/:slug` sin sesión y verificar contenido vigente y filtros — sin generar ni escanear un QR físico (ver `quickstart.md` → US3).

### Tests for User Story 3 ⚠️

- [ ] T048 [P] [US3] Integration test `GET /public/menu/:slug` (productos disponibles/agotados, filtros por categoría/etiqueta, tenant suspendido FR-023, vigencia por horario FR-015) en `test/e2e/public-menu.e2e-spec.ts`
- [ ] T049 [P] [US3] Integration test de invalidación de caché: cambio de precio/disponibilidad reflejado en `GET /public/menu/:slug` en menos de 5s (SC-002) en `test/e2e/public-menu-cache.e2e-spec.ts`
- [ ] T050 [P] [US3] Unit test de `PublicMenuService` (filtros, vigencia, lectura/repoblado de caché) en `src/public-menu/public-menu.service.spec.ts`

### Implementation for User Story 3

- [ ] T051 [US3] Generar `PublicMenuModule`/`PublicMenuService` con el Nest CLI (`nest g module public-menu && nest g service public-menu --no-spec`) e implementar: resolver por slug, aplicar vigencia de menú FR-015, filtrar por categoría/etiqueta FR-017, cache-aside vía `CacheService`, incrementar `Tenant.publicViewCount` de forma asíncrona FR-026, en `src/public-menu/public-menu.service.ts`
- [ ] T052 [US3] Generar `PublicMenuController` con el Nest CLI (`nest g controller public-menu --no-spec`) e implementar `GET /public/menu/:slug`, sin `JwtAuthGuard`, con throttler `public`, en `src/public-menu/public-menu.controller.ts`
- [ ] T053 [US3] Verificar que `PublicMenuModule` quedó registrado en `src/app.module.ts` (automático al generarlo con `nest g module` en T051)

**Checkpoint**: User Stories 1, 2 y 3 funcionan de forma independiente

---

## Phase 6: User Story 4 - Código QR del restaurante (Priority: P4)

**Goal**: El administrador obtiene y descarga el código QR de su restaurante (general y por mesa) listo para imprimir.

**Independent Test**: Crear un restaurante y verificar que se genera un QR válido apuntando a su URL pública, descargable en PNG/PDF — sin depender de que el menú ya tenga contenido (ver `quickstart.md` → US4).

### Tests for User Story 4 ⚠️

- [ ] T054 [P] [US4] Integration test `GET /tenants/:tenantId/qr`, descarga PNG/PDF, y `POST /tenants/:tenantId/qr/tables` (FR-019, FR-020, FR-021) en `test/e2e/qr.e2e-spec.ts`
- [ ] T055 [P] [US4] Unit test de `QrService` (construcción de URL destino, generación PNG/PDF) en `src/qr/qr.service.spec.ts`

### Implementation for User Story 4

- [ ] T056 [P] [US4] Generar `CreateTableQrDto` con el Nest CLI (`nest g class qr/dto/create-table-qr.dto --no-spec --flat`)
- [ ] T057 [US4] Generar `QrModule`/`QrService` con el Nest CLI (`nest g module qr && nest g service qr --no-spec`) e implementar: generar PNG con `qrcode`, envolver en PDF con `pdf-lib`, crear el QR general automáticamente, en `src/qr/qr.service.ts`
- [ ] T058 [US4] Generar `QrController` con el Nest CLI (`nest g controller qr --no-spec`) e implementar `GET /tenants/:tenantId/qr`, `GET .../qr/download`, `POST .../qr/tables`, `GET .../qr/tables/:qrCodeId/download`, en `src/qr/qr.controller.ts`
- [ ] T059 [US4] Enganchar la creación automática del QR general (FR-019) en `TenantsService.create` (`src/tenants/tenants.service.ts`) llamando a `QrService`
- [ ] T060 [US4] Verificar que `QrModule` quedó registrado en `src/app.module.ts` (automático al generarlo con `nest g module` en T057)

**Checkpoint**: Las 4 historias de usuario funcionan de forma independiente

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Mejoras que afectan a más de una historia de usuario

- [ ] T061 [P] Integration test de aislamiento multi-tenant: dos tenants/dos usuarios, verificar que el token del usuario A recibe 403/404 al operar sobre recursos del tenant B (SC-004) en `test/e2e/tenant-isolation.e2e-spec.ts`
- [ ] T062 [P] Revisar que las anotaciones Swagger (`@ApiTags`, `@ApiProperty`, `@ApiResponse`) en todos los controladores/DTOs coincidan con `contracts/api.md` (Quality Gates, Constitución)
- [ ] T063 Ejecutar el reporte de cobertura de Jest y confirmar el umbral >70% (Constitución I, Quality Gates)
- [ ] T064 [P] Escribir `README.md` con instrucciones de setup, referenciando `quickstart.md`
- [ ] T065 Ejecutar manualmente la validación completa de `quickstart.md` (las 4 historias) contra un entorno local
- [ ] T066 [P] Pase de hardening de seguridad: confirmar `class-validator` en cada DTO, queries de Prisma parametrizadas, sin logging de contraseñas en texto plano (Constitución IV)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Phase 2)**: depende de Setup — bloquea todas las historias de usuario
- **User Stories (Phase 3-6)**: todas dependen de completar Foundational
  - Pueden avanzar en paralelo si hay equipo, o en orden de prioridad (P1→P2→P3→P4)
  - US4 depende funcionalmente de que `TenantsService.create` exista (T028, de US1) para enganchar la creación automática del QR (T059) — el resto de US4 es independiente
- **Polish (Phase 7)**: depende de que las historias que se quieran entregar estén completas

### User Story Dependencies

- **User Story 1 (P1)**: sin dependencias de otras historias
- **User Story 2 (P2)**: usa la autenticación de US1 para probarse, pero su lógica de negocio (menús/categorías/productos) es independiente
- **User Story 3 (P3)**: requiere contenido cargado (US2) para ser útil, pero su código (lectura pública + caché) es independiente
- **User Story 4 (P4)**: requiere un tenant creado (US1); T059 es el único punto de integración explícito con US1

### Within Each User Story

- Tests antes que implementación (deben fallar primero)
- DTOs/modelos antes que servicios
- Servicios antes que controladores
- Implementación core antes que integración cross-story (ej. T047, T059)

### Parallel Opportunities

- Todas las tareas `[P]` de Setup y Foundational pueden correr en paralelo entre sí
- Una vez cerrada Foundational, US1–US4 pueden trabajarse en paralelo por distintos desarrolladores (con la única atadura señalada: T059 depende de T028)
- Todos los tests `[P]` de una misma historia pueden correr en paralelo
- Los DTOs `[P]` dentro de una historia pueden crearse en paralelo

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los tests de la Historia 1:
Task: "Integration test POST /auth/register en test/e2e/auth-register.e2e-spec.ts"
Task: "Integration test POST /auth/login en test/e2e/auth-login.e2e-spec.ts"
Task: "Integration test POST /tenants en test/e2e/tenants-create.e2e-spec.ts"
Task: "Unit test AuthService en src/auth/auth.service.spec.ts"
Task: "Unit test TenantsService en src/tenants/tenants.service.spec.ts"

# Lanzar juntos los DTOs de la Historia 1:
Task: "Crear RegisterDto/LoginDto en src/auth/dto/"
Task: "Crear CreateTenantDto/UpdateTenantDto en src/tenants/dto/"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (crítico — bloquea todo lo demás)
3. Completar Phase 3: User Story 1
4. **Parar y validar**: correr `quickstart.md` → US1 de forma aislada
5. Desplegar/demo si está listo

### Incremental Delivery

1. Setup + Foundational → base lista
2. + User Story 1 → probar de forma aislada → demo (MVP)
3. + User Story 2 → probar de forma aislada → demo
4. + User Story 3 → probar de forma aislada → demo
5. + User Story 4 → probar de forma aislada → demo
6. Cada historia agrega valor sin romper las anteriores

### Parallel Team Strategy

Con más de un desarrollador:

1. El equipo completa junto Setup + Foundational
2. Una vez lista Foundational:
   - Dev A: User Story 1
   - Dev B: User Story 2 (puede empezar en paralelo si mockea la auth de US1)
   - Dev C: User Story 3 (depende de que exista contenido de prueba, puede sembrarse vía seed sin esperar a US2 terminado)
   - Dev D: User Story 4 (coordina con Dev A el punto de integración T059)
3. Las historias se completan e integran de forma independiente

---

## Notes

- `[P]` = archivos distintos, sin dependencias entre sí
- Todo módulo/controller/service/guard/filter/decorator se crea con los generadores del Nest CLI (ver Path Conventions), nunca escribiendo el archivo a mano desde cero
- La etiqueta `[Story]` traza cada tarea a su historia de usuario
- Cada historia de usuario debe ser completable y probable de forma independiente
- Verificar que los tests fallan antes de implementar
- Commitear después de cada tarea o grupo lógico (Conventional Commits, Constitución)
- Detenerse en cada checkpoint para validar la historia de forma aislada
- Evitar: tareas vagas, conflictos de archivo entre tareas paralelas, dependencias cross-story que rompan la independencia
