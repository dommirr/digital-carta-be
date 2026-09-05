<!--
Sync Impact Report
==================
Version change: N/A (unratified placeholder template) → 1.0.0
Rationale: Initial ratification. The prior file on disk was the unfilled
Spec Kit scaffold (all bracket placeholders, no adopted content). This is
the first concrete version of the constitution, so it is seeded at 1.0.0
rather than treated as an amendment bump.

Principles established:
  I.   Stack Tecnológico No Negociable
  II.  Aislamiento Multi-Tenant Estricto
  III. Rendimiento y Caché
  IV.  Seguridad por Defecto
  V.   Estándares de Código
  VI.  Integridad y Auditoría de Datos

Sections added:
  - Flujo de Trabajo de Desarrollo (SECTION_2)
  - Quality Gates y Cumplimiento (SECTION_3)
  - Governance (amendment procedure, versioning policy, compliance review)

Sections removed: none (initial creation)

Templates requiring alignment (checked, not modified per scope guard):
  ✅ .specify/templates/plan-template.md — Constitution Check gate references
     principle set generically; no stale principle names to update.
  ✅ .specify/templates/spec-template.md — no direct constitution references.
  ✅ .specify/templates/tasks-template.md — no direct constitution references.
  ⚠ No command file explicitly enumerates these 6 principles by name; no
    action required now, but future edits to principle names/count should
    re-check dependent templates per the Sync Impact Report contract.

Deferred / follow-up TODOs:
  - TODO(RATIFICATION_DATE): confirmed as 2026-09-05 (date of this session,
    since no earlier ratified version existed). Update if the team can
    identify an earlier, actual adoption date.
-->

# Digital Carta Backend Constitution

## Core Principles

### I. Stack Tecnológico No Negociable
El backend usa siempre NestJS 10 + TypeScript 5, PostgreSQL 15 + Prisma 5 como
capa de datos, Passport para la implementación de JWT, `class-validator` +
`class-transformer` para validación, Swagger/OpenAPI 3 para documentación de
API, y Jest + Supertest para testing con cobertura mínima del 70%. Ninguna
feature puede introducir un framework, ORM o librería alternativa que
duplique estas responsabilidades sin una enmienda explícita de esta
constitución. Rationale: fijar el stack evita fragmentación técnica, reduce
la carga de mantenimiento en un equipo pequeño y garantiza que cualquier
desarrollador pueda incorporarse a cualquier módulo sin curva de aprendizaje
adicional.

### II. Aislamiento Multi-Tenant Estricto
Cada tenant DEBE tener sus datos aislados mediante `tenant_id` en toda tabla
que almacene datos de negocio. PostgreSQL Row Level Security (RLS) DEBE
aplicarse como segunda capa de defensa además del filtrado a nivel de
aplicación — nunca como sustituto de este. La resolución del tenant activo
se hace vía slug en las URLs públicas (menú del comensal) y vía claims del
JWT en el panel de administración; ninguna otra vía de resolución de tenant
es válida. Rationale: en un SaaS multi-tenant, una fuga de datos entre
tenants es el fallo más costoso posible (legal, reputacional, contractual);
la doble capa (aplicación + RLS) protege incluso ante errores humanos en
queries individuales.

### III. Rendimiento y Caché
El 95% de las peticiones DEBEN responder en menos de 500ms. Los menús
públicos DEBEN servirse desde caché Redis, con invalidación on-change
disparada por cualquier mutación que afecte el contenido publicado (no
invalidación por tiempo como único mecanismo). Rationale: el comensal final
accede al menú desde un QR sin fricción; una carga lenta destruye la
experiencia y la propuesta de valor del producto frente al menú impreso.

### IV. Seguridad por Defecto
Los JWT expiran a las 24 horas sin excepción. El rate limiting es
obligatorio: 100 req/min para endpoints de administración y 1000 req/min
para endpoints públicos. Todo DTO de entrada DEBE validarse con
`class-validator`. El código y las queries DEBEN prevenir activamente SQL
injection y XSS (uso de Prisma parametrizado, sanitización de salida en
contenido enriquecido). Ninguna feature se considera completa si omite
validación de entrada o expone un endpoint sin rate limiting. Rationale:
como SaaS que expone endpoints públicos sin autenticación (menú vía QR), la
superficie de ataque es amplia; estos controles son la línea base mínima,
no un extra.

### V. Estándares de Código
TypeScript strict mode se usa siempre, sin excepciones ni `any` implícitos.
Cada módulo de NestJS DEBE ser autocontenido con su propio controlador,
servicio y DTOs. Los servicios DEBEN tener unit tests obligatorios; los
endpoints DEBEN tener tests de integración. Rationale: la modularidad de
NestJS solo aporta valor si se respeta consistentemente; strict mode y
tests son la red de seguridad que permite refactorizar con confianza en un
dominio multi-tenant donde los errores son costosos.

### VI. Integridad y Auditoría de Datos
Los identificadores de entidad son siempre UUID (`cuid()` de Prisma). Está
prohibido el borrado físico (`DELETE`) de registros de negocio; toda
eliminación lógica se implementa vía un campo de estado (`status`/`active`).
Toda tabla DEBE registrar `createdAt` y `updatedAt`. Rationale: el borrado
físico es irreversible y complica la auditoría, el soporte al cliente y la
recuperación ante errores; UUIDs evitan colisiones y filtraciones de
información secuencial entre tenants.

## Flujo de Trabajo de Desarrollo

Los commits DEBEN seguir Conventional Commits (`feat`, `fix`, `chore`,
`docs`, entre otros prefijos estándar) para mantener un historial legible y
habilitar automatización futura (changelogs, versionado semántico). Todo
cambio de código DEBE pasar por revisión antes de integrarse a la rama
principal, verificando cumplimiento de los Principios Fundamentales
anteriores, en particular aislamiento multi-tenant (Principio II) y
seguridad por defecto (Principio IV), que son las áreas de mayor riesgo.

## Quality Gates y Cumplimiento

Ningún cambio se considera listo para producción si no cumple, como mínimo:
cobertura de tests de servicios y endpoints por encima del 70% (Principio
I), validación de todos los DTOs de entrada (Principio IV), y ausencia de
queries que omitan el filtrado por `tenant_id` (Principio II). La
documentación Swagger/OpenAPI DEBE mantenerse sincronizada con los DTOs y
controladores reales en cada PR que modifique un endpoint. El cumplimiento
del umbral de rendimiento (<500ms p95, Principio III) se verifica mediante
monitoreo continuo, no solo en revisión de código.

## Governance

Esta constitución prevalece sobre cualquier otra práctica, guía o
convención informal del equipo. Cualquier excepción a un principio marcado
como no negociable DEBE documentarse explícitamente en el PR correspondiente
junto con su justificación técnica o de negocio, y DEBE tratarse como deuda
técnica a resolver, no como precedente.

**Procedimiento de enmienda**: cambios a esta constitución se proponen vía
`/speckit-constitution`, se documentan en el Sync Impact Report del propio
archivo, y requieren aprobación de al menos un mantenedor del proyecto antes
de fusionarse.

**Política de versionado**: esta constitución usa versionado semántico.
MAJOR ante eliminación o redefinición incompatible de un principio; MINOR
al añadir un principio o sección, o ampliar sustancialmente una guía
existente; PATCH ante aclaraciones o correcciones de redacción sin cambio
de significado.

**Revisión de cumplimiento**: toda spec, plan y tarea generada por Spec Kit
para este proyecto DEBE verificarse contra estos principios en su fase de
"Constitution Check". Cualquier desviación no justificada bloquea el avance
a la siguiente fase del flujo de trabajo.

**Version**: 1.0.0 | **Ratified**: 2026-09-05 | **Last Amended**: 2026-09-05
