# Feature Specification: MenuFlow – Plataforma de Gestión de Menús Digitales (MVP)

**Feature Branch**: `[001-menuflow-backend-mvp]`

**Created**: 2026-09-05

**Status**: Draft

**Input**: User description: "MenuFlow - Sistema de Gestión de Menús Digitales: sistema backend para que restaurantes, bares y establecimientos de hostelería gestionen sus menús digitales. Los propietarios administran el contenido (platos, categorías, precios, disponibilidad) desde un panel web, y los clientes acceden al menú escaneando un código QR ubicado en la mesa. Incluye autenticación de administradores, gestión multi-tenant de restaurantes, múltiples menús con activación por horario, categorías y productos con reordenamiento y duplicado, vista pública sin login con filtros, generación de código QR descargable, roles Admin/Editor/Super-admin, y estadísticas básicas de visualizaciones."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un restaurante se da de alta y publica su primer menú (Priority: P1)

Un dueño o gerente de restaurante crea una cuenta, registra los datos básicos de su local (nombre, URL amigable, logo, colores) y queda con una sesión autenticada para empezar a cargar contenido. Sin esto, ninguna otra funcionalidad del sistema es accesible ni segura.

**Why this priority**: Es el punto de entrada obligatorio para todo lo demás: sin cuenta ni local creado no hay dónde cargar un menú ni a quién mostrárselo. Es la unidad más pequeña de valor entregable de forma aislada (un restaurante "existe" en la plataforma).

**Independent Test**: Puede probarse íntegramente registrando una cuenta nueva, iniciando sesión y verificando que el restaurante queda creado con una URL pública única, sin depender de ninguna otra historia.

**Acceptance Scenarios**:

1. **Given** que no existe una cuenta previa, **When** una persona se registra con email, contraseña y los datos de su restaurante, **Then** el sistema crea la cuenta, el restaurante asociado con una URL pública única (slug) y devuelve una sesión autenticada.
2. **Given** una cuenta ya registrada, **When** el usuario inicia sesión con sus credenciales correctas, **Then** el sistema le concede acceso autenticado válido por 24 horas.
3. **Given** un usuario que ya administra un restaurante, **When** registra un segundo restaurante (por ejemplo, para una cadena), **Then** ambos restaurantes quedan asociados a su cuenta y son gestionables de forma independiente entre sí.
4. **Given** un intento de registro con una URL de restaurante (slug) ya utilizada por otro local, **When** el usuario intenta guardarla, **Then** el sistema rechaza el registro e informa que la URL no está disponible.

---

### User Story 2 - Un administrador gestiona el contenido de su menú (Priority: P2)

El administrador del restaurante crea categorías (Entrantes, Principales, Postres, Bebidas), agrega productos con nombre, descripción, precio, foto, alérgenos y etiquetas dietéticas, y mantiene esa información actualizada: cambia precios, marca productos como agotados o disponibles, reordena categorías y productos, y duplica elementos existentes para agilizar la carga.

**Why this priority**: Es el corazón del valor del producto — reemplaza al menú impreso. Sin esta historia, la cuenta del restaurante existe pero no tiene nada que mostrar.

**Independent Test**: Puede probarse creando una categoría, agregando un producto con todos sus campos, editando su precio y disponibilidad, y verificando que los cambios quedan guardados — todo esto usando solamente una cuenta ya autenticada (de la Historia 1), sin depender de la vista pública ni del QR.

**Acceptance Scenarios**:

1. **Given** un restaurante recién creado sin menú, **When** el administrador crea una categoría con un nombre, **Then** la categoría queda disponible para asociarle productos.
2. **Given** una categoría existente, **When** el administrador agrega un producto con nombre, precio, descripción, foto, alérgenos y etiquetas, **Then** el producto queda guardado y asociado a esa categoría.
3. **Given** un producto existente, **When** el administrador cambia su precio o lo marca como "agotado", **Then** el cambio se guarda de inmediato y queda reflejado como el estado vigente del producto.
4. **Given** varias categorías o productos dentro de una categoría, **When** el administrador cambia su orden de presentación, **Then** el nuevo orden se conserva la próxima vez que se consulte el menú.
5. **Given** un producto ya cargado, **When** el administrador lo duplica, **Then** se crea una copia editable con los mismos datos (salvo el nombre, que se distingue del original) sin tener que volver a cargar cada campo.
6. **Given** un restaurante que necesita más de una carta (ej. "Carta principal" y "Brunch"), **When** el administrador crea un menú adicional con un horario de activación, **Then** cada menú se activa y desactiva automáticamente según el horario configurado, o puede activarse/desactivarse manualmente.
7. **Given** que el administrador intenta guardar un producto con un precio negativo o no numérico, **When** intenta guardarlo, **Then** el sistema rechaza el guardado e indica el error.

---

### User Story 3 - Un cliente final consulta el menú público mediante un código QR (Priority: P3)

El comensal escanea el código QR de su mesa, accede a la vista pública del menú del restaurante sin necesidad de crear cuenta ni iniciar sesión, y puede filtrar por categoría o por etiquetas dietéticas para encontrar lo que busca. Los cambios que el administrador guardó en la Historia 2 se ven reflejados sin demora perceptible.

**Why this priority**: Completa el ciclo de valor de negocio (el motivo de ser del producto: reemplazar el menú impreso), pero depende de que ya exista contenido cargado (Historia 2), por lo que se prioriza después.

**Independent Test**: Puede probarse generando la URL pública de un restaurante con contenido ya cargado y accediendo a ella sin sesión iniciada, verificando que se ve el contenido vigente y que los filtros funcionan, sin necesidad de generar o escanear un QR físico.

**Acceptance Scenarios**:

1. **Given** un restaurante con un menú publicado, **When** cualquier persona abre la URL pública del restaurante sin iniciar sesión, **Then** ve las categorías y productos disponibles, organizados según el orden configurado por el administrador.
2. **Given** un producto marcado como "agotado", **When** un cliente ve el menú público, **Then** el producto se muestra visualmente diferenciado como no disponible, sin ocultarse.
3. **Given** un menú con productos etiquetados (ej. vegano, sin gluten), **When** el cliente aplica un filtro por categoría o etiqueta, **Then** solo se muestran los productos que cumplen ese filtro.
4. **Given** que el administrador actualizó un precio o disponibilidad, **When** un cliente abre o recarga la vista pública después de ese cambio, **Then** ve la información actualizada, sin necesidad de que se reimprima o regenere el código QR.
5. **Given** un restaurante suspendido o dado de baja, **When** un cliente intenta acceder a su URL pública, **Then** el sistema muestra un mensaje indicando que el menú no está disponible, sin exponer datos del restaurante.

---

### User Story 4 - Un administrador genera y descarga el código QR de su restaurante (Priority: P4)

El administrador obtiene automáticamente un código QR al crear su restaurante, y puede descargarlo en un formato listo para imprimir y colocar en las mesas. Opcionalmente puede generar códigos QR adicionales identificados por mesa.

**Why this priority**: Es el puente físico entre el contenido cargado (Historia 2) y el cliente final (Historia 3); sin él, el cliente no tiene forma práctica de llegar al menú, pero el valor del sistema ya existe sin esta historia (se puede compartir la URL manualmente).

**Independent Test**: Puede probarse creando un restaurante y verificando que se genera un código QR válido que apunta a su URL pública, descargándolo en un formato imprimible, sin depender de que el menú ya tenga contenido cargado.

**Acceptance Scenarios**:

1. **Given** un restaurante recién creado, **When** el administrador consulta su código QR, **Then** el sistema le entrega un código QR ya generado que enlaza a la URL pública de ese restaurante.
2. **Given** un código QR generado, **When** el administrador lo descarga, **Then** puede obtenerlo en un formato apto para impresión.
3. **Given** un restaurante con varias mesas, **When** el administrador genera un código QR adicional identificando una mesa específica, **Then** ese código conserva el identificador de mesa al abrir la vista pública (preparado para uso futuro, sin afectar el contenido mostrado hoy).

---

### Edge Cases

- Si un administrador intenta eliminar una categoría que todavía tiene productos asociados, el sistema bloquea la eliminación e indica que debe mover o eliminar antes esos productos.
- ¿Qué ocurre si dos administradores del mismo restaurante editan el mismo producto al mismo tiempo? El sistema debe conservar el último cambio guardado sin corromper los datos ni duplicar el producto.
- ¿Qué ocurre si un producto se crea sin foto? Debe quedar visible en el menú público igualmente, sin foto o con una imagen de reemplazo.
- ¿Qué ocurre si un cliente escanea un QR de un restaurante que fue suspendido o eliminado? Debe ver un aviso claro, sin exponer datos ni error técnico.
- ¿Qué ocurre si se intenta registrar un restaurante con una URL (slug) ya usada por otro? El sistema debe rechazarlo y sugerir que se elija otra.
- ¿Qué ocurre si un administrador intenta acceder o modificar el menú de un restaurante que no le pertenece? El sistema debe impedirlo por completo, en todos los casos, sin excepción.
- ¿Qué ocurre si se agota el tiempo de una sesión (24 horas) mientras el administrador está editando? Debe pedir reautenticación sin perder los datos ya guardados previamente.
- ¿Qué ocurre si un menú con horario de activación configurado (ej. "Brunch", sábados y domingos) es consultado fuera de su horario? No debe aparecer en la vista pública en ese momento.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir que una persona registre una cuenta de administrador con email y contraseña.
- **FR-002**: El sistema DEBE permitir iniciar sesión con esas credenciales y mantener la sesión autenticada activa por 24 horas.
- **FR-003**: El sistema DEBE permitir que una cuenta de administrador gestione más de un restaurante de forma independiente (soporte para cadenas).
- **FR-004**: El sistema DEBE permitir registrar un restaurante con nombre, URL pública única (slug), logo y colores de marca.
- **FR-005**: El sistema DEBE rechazar el registro de un restaurante cuya URL pública (slug) ya esté en uso.
- **FR-006**: El sistema DEBE aislar completamente los datos de cada restaurante, de forma que ningún administrador pueda ver ni modificar el contenido de un restaurante que no le pertenece, sin ninguna excepción.
- **FR-007**: El sistema DEBE permitir crear, editar y eliminar categorías dentro de un menú.
- **FR-008**: El sistema DEBE permitir crear, editar y eliminar productos dentro de una categoría, incluyendo nombre, descripción, precio, foto, alérgenos y etiquetas dietéticas.
- **FR-009**: El sistema DEBE validar que el precio de un producto sea un valor numérico positivo antes de guardarlo.
- **FR-010**: El sistema DEBE permitir marcar un producto como "disponible" o "agotado" en cualquier momento, reflejando el cambio de inmediato.
- **FR-011**: El sistema DEBE permitir reordenar categorías y productos, conservando el orden definido por el administrador en cualquier consulta posterior.
- **FR-012**: El sistema DEBE permitir duplicar una categoría o un producto existente para agilizar la carga de contenido similar.
- **FR-013**: El sistema DEBE permitir que un restaurante tenga más de un menú (ej. "Carta principal", "Brunch"), cada uno con activación por horario/días de la semana y/o activación manual.
- **FR-014**: El sistema DEBE exponer una vista pública del menú vigente de cada restaurante, accesible mediante su URL única sin necesidad de autenticarse.
- **FR-015**: La vista pública DEBE mostrar únicamente el contenido del/los menú(s) activo(s) en ese momento, respetando los horarios de activación configurados.
- **FR-016**: La vista pública DEBE mostrar los productos marcados como "agotado" de forma visualmente diferenciada, sin ocultarlos.
- **FR-017**: La vista pública DEBE permitir filtrar productos por categoría y por etiqueta dietética.
- **FR-018**: Cualquier cambio guardado por un administrador (precio, disponibilidad, contenido, orden) DEBE reflejarse en la vista pública sin que el cliente final necesite volver a escanear o regenerar el código QR.
- **FR-019**: El sistema DEBE generar automáticamente un código QR único al crear un restaurante, enlazando a su URL pública.
- **FR-020**: El sistema DEBE permitir descargar el código QR en un formato apto para impresión.
- **FR-021**: El sistema DEBE permitir generar códigos QR adicionales identificados por mesa, preparando el enlace para uso futuro sin alterar el contenido mostrado.
- **FR-022**: El sistema DEBE soportar al menos dos roles asociables a una cuenta dentro de un restaurante: Admin (control total sobre categorías, productos, precios, disponibilidad y estructura) y Editor. El rol Editor queda modelado como dato asociable a una cuenta en este alcance, pero la aplicación de restricciones específicas de permisos para el Editor (limitarlo a cambiar solo disponibilidad) queda fuera de este MVP y se implementará en una fase posterior.
- **FR-023**: El sistema DEBE impedir el acceso a la vista pública de un restaurante suspendido o dado de baja, mostrando un aviso genérico sin exponer datos internos.
- **FR-024**: El sistema DEBE eliminar categorías y productos de forma lógica (no destructiva), preservando el registro para fines de auditoría y recuperación.
- **FR-025**: El sistema DEBE registrar la fecha de creación y de última modificación de cada restaurante, menú, categoría y producto.
- **FR-026**: El sistema DEBE registrar estadísticas básicas de visualizaciones del menú público, contabilizando cada carga de la vista pública como una visualización y presentando al administrador un conteo total acumulado por restaurante (sin desglose por fecha ni por producto en este alcance).
- **FR-027**: El sistema DEBE impedir la eliminación de una categoría mientras contenga productos activos asociados, informando al administrador que primero debe mover o eliminar esos productos.

### Key Entities

- **Restaurante (Tenant)**: representa un local afiliado a la plataforma. Atributos clave: nombre, URL pública única (slug), logo, colores de marca, estado (activo/suspendido/eliminado).
- **Cuenta de administrador**: persona que gestiona uno o más restaurantes. Se autentica con email/contraseña y puede tener distinto nivel de permisos (Admin/Editor) por restaurante.
- **Menú**: agrupación de categorías que pertenece a un restaurante; tiene nombre, estado de activación y, opcionalmente, un horario/días de vigencia.
- **Categoría**: agrupación de productos dentro de un menú (ej. "Entrantes"); tiene nombre y posición de orden.
- **Producto**: un plato o bebida dentro de una categoría; tiene nombre, descripción, precio, foto, alérgenos, etiquetas dietéticas, estado de disponibilidad y posición de orden.
- **Código QR**: enlace visual generado para un restaurante (y opcionalmente para una mesa específica dentro de él) que dirige a la vista pública del menú.
- **Registro de visualización**: evento que representa una consulta al menú público, usado para las estadísticas básicas de uso.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un administrador puede registrar su cuenta, crear su restaurante y publicar una primera categoría con un producto en menos de 15 minutos sin asistencia externa.
- **SC-002**: Un cambio de precio o disponibilidad realizado por un administrador es visible en la vista pública del menú en menos de 5 segundos.
- **SC-003**: La vista pública del menú carga en menos de 2 segundos bajo condiciones de red móvil estándar, para el 95% de las consultas.
- **SC-004**: El sistema mantiene el aislamiento de datos entre restaurantes en el 100% de los casos verificados: ningún administrador puede ver o modificar contenido de un restaurante ajeno.
- **SC-005**: Un administrador puede obtener y descargar el código QR de su restaurante en menos de 1 minuto desde que su cuenta queda creada.
- **SC-006**: El 95% de las operaciones administrativas de guardado (crear, editar, reordenar o eliminar categorías/productos) se perciben como instantáneas por el usuario (menos de 500ms).
- **SC-007**: El sistema sostiene picos de consulta pública equivalentes a 1000 solicitudes por minuto por restaurante sin degradar el tiempo de carga de la vista pública.
- **SC-008**: Cero pérdidas de datos de menú reportadas por eliminaciones accidentales, dado que toda eliminación es reversible mediante el historial conservado.

## Assumptions

- El panel de administración y la vista pública se consumen desde una aplicación cliente separada (web/móvil); esta especificación cubre la capacidad de negocio que dicho cliente necesita, no su interfaz visual.
- No se incluyen pedidos, pagos ni reservas de mesa en este alcance: quedan fuera del MVP.
- No se incluyen planes de suscripción, límites por plan (Free/Pro) ni facturación en este alcance; se asume que todo restaurante registrado tiene acceso completo a las funcionalidades descritas.
- La gestión global de la plataforma (suspender cuentas por falta de pago, administración de todos los tenants) corresponde a un rol de Super-admin que queda fuera de esta especificación y se abordará en una fase posterior.
- El contenido del menú se gestiona en un único idioma al momento de esta especificación; el diseño de datos deberá poder extenderse a múltiples idiomas más adelante sin necesitar esta capacidad ahora.
- Todo restaurante nuevo queda en estado "activo" inmediatamente después del registro, sin un proceso de aprobación manual previo.
- Las credenciales de acceso se gestionan siguiendo buenas prácticas estándar de protección de contraseñas de la industria.
