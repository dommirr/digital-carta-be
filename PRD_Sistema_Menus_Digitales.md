# PRD: Sistema de Gestión de Menús Digitales (SaaS Multi-Tenant)

**Versión:** 1.0
**Fecha:** Septiembre 2026
**Estado:** Borrador para revisión

---

## 1. Resumen ejecutivo

Plataforma SaaS multi-tenant que permite a restaurantes, bares y establecimientos de hostelería crear, actualizar y publicar menús digitales en tiempo real. Los clientes finales acceden al menú escaneando un código QR ubicado en la mesa o entrada del local, sin necesidad de descargar ninguna app. Los propietarios gestionan el contenido desde un panel de administración web sencillo.

## 2. Problema a resolver

- Los menús impresos son costosos de actualizar (precios, disponibilidad de platos, promociones estacionales).
- Los cambios de precio o de carta requieren reimprimir y reemplazar material físico en todas las mesas.
- Los clientes valoran cada vez más experiencias sin contacto (post-pandemia) y acceso rápido a información (alérgenos, fotos, idiomas).
- Los dueños de pequeños/medianos locales no tienen tiempo ni conocimientos técnicos para mantener una web o app propia.

## 3. Objetivos del producto

| Objetivo | Métrica de éxito |
|---|---|
| Reducir el tiempo de actualización de un menú | De días (impresión) a menos de 1 minuto |
| Facilitar la adopción por dueños no técnicos | Onboarding completo en menos de 15 minutos sin soporte |
| Ofrecer una experiencia rápida al comensal | Carga del menú en menos de 2 segundos en 4G |
| Monetizar como SaaS | X locales pagantes activos en los primeros 6 meses (definir con negocio) |

## 4. Usuarios y personas

### 4.1 Propietario/gerente de restaurante ("Admin del local")
- Necesita editar precios, agregar/quitar platos, marcar productos agotados en tiempo real.
- Baja alfabetización técnica; usa principalmente el celular.

### 4.2 Personal de sala/cocina ("Editor")
- Rol con permisos limitados: puede marcar "agotado"/"disponible" pero no cambiar precios o estructura.

### 4.3 Comensal ("Cliente final")
- Escanea el QR, navega el menú, no requiere login ni descarga de app.
- Puede estar en distintos idiomas, tener restricciones alimentarias.

### 4.4 Super-admin (equipo de la plataforma)
- Gestiona cuentas, planes de suscripción, soporte y facturación de todos los tenants.

## 5. Alcance del producto (MVP vs. futuro)

### 5.1 Funcionalidades del MVP
1. **Registro y onboarding** de un nuevo local (datos del negocio, logo, colores de marca).
2. **Panel de administración** (web responsive):
   - CRUD de categorías (ej. Entradas, Principales, Bebidas, Postres).
   - CRUD de productos: nombre, descripción, precio, foto, alérgenos, etiquetas (vegano, picante, sin TACC).
   - Marcar producto como "agotado" / "disponible" en tiempo real.
   - Reordenar categorías y productos (drag and drop).
   - Activar/desactivar un menú completo (ej. menú de brunch solo fines de semana).
3. **Generación de código QR** único por local (y opcionalmente por mesa) que enlaza a la URL pública del menú.
4. **Vista pública del menú** (sin login):
   - Diseño mobile-first, carga rápida.
   - Filtros por categoría y por etiqueta (vegano, sin gluten, etc.).
   - Soporte multi-idioma básico (ES/EN al menos).
5. **Actualización en tiempo real**: un cambio en el panel se refleja de inmediato en la vista pública (sin caché obsoleta).
6. **Gestión multi-tenant**: cada restaurante tiene su propio espacio aislado de datos.
7. **Planes de suscripción básicos** (ej. Free/Pro) con límites de productos o locales.

### 5.2 Fuera de alcance del MVP (roadmap futuro)
- Pedidos y pagos integrados desde el QR (pedir y pagar en mesa).
- Reservas de mesa.
- Programa de fidelización / cupones.
- Analítica avanzada (productos más vistos, horarios pico).
- Integración con POS (punto de venta) existentes.
- App nativa para el personal (más allá del panel web responsive).
- Menús con IA (recomendaciones personalizadas, traducción automática avanzada).

## 6. Requisitos funcionales detallados

### 6.1 Gestión de cuenta y tenant
- RF-01: El sistema debe permitir crear una cuenta de restaurante con email/contraseña o login social.
- RF-02: El sistema debe permitir un onboarding guiado (nombre del local, rubro, logo, colores).
- RF-03: Un mismo usuario puede administrar más de un local (para cadenas).
- RF-04: El super-admin debe poder suspender o eliminar una cuenta por falta de pago.

### 6.2 Gestión de menú
- RF-05: El admin puede crear, editar y eliminar categorías.
- RF-06: El admin puede crear, editar y eliminar productos dentro de una categoría.
- RF-07: Cada producto debe soportar: nombre, descripción, precio, imagen, alérgenos, etiquetas dietéticas, estado (disponible/agotado).
- RF-08: El admin puede reordenar categorías y productos.
- RF-09: El admin puede duplicar un producto o categoría para agilizar la carga.
- RF-10: El sistema debe soportar múltiples menús por local (ej. desayuno, almuerzo, carta de vinos) con horarios de activación.

### 6.3 Código QR y publicación
- RF-11: El sistema debe generar automáticamente un código QR único por local al crear la cuenta.
- RF-12: El sistema debe permitir generar QR adicionales por mesa (con un identificador de mesa embebido en la URL, opcional para analítica futura).
- RF-13: El QR debe poder descargarse en formato imprimible (PNG/PDF) en distintos tamaños.
- RF-14: Los cambios publicados deben reflejarse en la URL pública sin necesidad de regenerar el QR.

### 6.4 Vista pública del menú
- RF-15: La vista debe ser accesible sin autenticación mediante una URL única por local.
- RF-16: La vista debe adaptarse a dispositivos móviles (mobile-first).
- RF-17: El cliente debe poder filtrar por categoría y por etiquetas dietéticas.
- RF-18: El cliente debe poder cambiar el idioma del menú si hay más de uno disponible.
- RF-19: Los productos marcados como "agotados" deben mostrarse visualmente diferenciados (no ocultos, salvo configuración del admin).

### 6.5 Roles y permisos
- RF-20: El sistema debe soportar al menos dos roles dentro de un local: Admin (control total) y Editor (solo disponibilidad).
- RF-21: El super-admin de la plataforma tiene acceso a un panel de gestión de todos los tenants, planes y facturación.

## 7. Requisitos no funcionales

| Categoría | Requisito |
|---|---|
| Rendimiento | La vista pública debe cargar en menos de 2 segundos en conexión 4G promedio |
| Disponibilidad | SLA objetivo de 99.5% de uptime |
| Escalabilidad | Arquitectura multi-tenant capaz de soportar miles de locales sin degradación |
| Seguridad | Aislamiento estricto de datos entre tenants; cifrado en tránsito (HTTPS) y en reposo |
| Accesibilidad | Cumplir pautas básicas WCAG AA en la vista pública |
| Internacionalización | Arquitectura preparada para múltiples idiomas y monedas desde el diseño de datos |
| Compatibilidad | Vista pública funcional en los navegadores móviles más usados (Chrome, Safari) sin requerir instalación |

## 8. Modelo de datos (alto nivel)

- **Tenant (Local/Restaurante):** id, nombre, logo, colores de marca, plan, estado.
- **Usuario:** id, tenant_id (o relación N:N si administra varios), rol, email, credenciales.
- **Menú:** id, tenant_id, nombre (ej. "Carta principal", "Brunch"), horario de activación, estado.
- **Categoría:** id, menú_id, nombre, orden.
- **Producto:** id, categoría_id, nombre, descripción, precio, imagen_url, alérgenos, etiquetas, disponible (bool), orden.
- **QR:** id, tenant_id, mesa_id (opcional), url_destino, fecha de creación.
- **Suscripción/Plan:** id, tenant_id, tipo de plan, estado de pago, límites (nº de productos, nº de locales).

## 9. Flujos principales (user journeys)

### 9.1 Alta de un nuevo restaurante
1. El dueño se registra en la plataforma.
2. Completa el onboarding (datos del local, logo).
3. Carga su primera categoría y productos (o importa desde una plantilla/PDF existente — funcionalidad deseable).
4. El sistema genera el QR del local.
5. El dueño descarga e imprime el QR para colocarlo en las mesas.

### 9.2 Actualización de precio en tiempo real
1. El admin ingresa al panel desde su celular.
2. Edita el precio de un producto.
3. Guarda el cambio.
4. Cualquier cliente que escanee el QR a partir de ese momento ve el precio actualizado, sin necesidad de reimprimir nada.

### 9.3 Cliente consultando el menú
1. El cliente escanea el QR en la mesa.
2. Se abre el navegador con la vista pública del menú (sin instalar nada).
3. Filtra por categoría o etiqueta dietética si lo desea.
4. Consulta detalles del plato (foto, descripción, alérgenos).

## 10. Modelo de negocio sugerido (a validar con el equipo comercial)

| Plan | Público objetivo | Límites orientativos |
|---|---|---|
| Free / Prueba | Locales pequeños, prueba del producto | 1 local, 1 menú, hasta ~20 productos, marca de agua de la plataforma en el menú |
| Pro | Restaurantes independientes | 1 local, menús ilimitados, productos ilimitados, sin marca de agua, QR por mesa |
| Business | Cadenas / múltiples locales | Varios locales bajo una misma cuenta, roles avanzados, soporte prioritario |

*(Los precios y límites exactos requieren validación de negocio; no se definen en este PRD.)*

## 11. Riesgos y consideraciones

- **Conectividad del cliente final:** si el comensal no tiene datos/wifi, no puede ver el menú. Mitigación: ofrecer PDF de respaldo descargable o menú impreso mínimo como fallback.
- **Adopción por usuarios no técnicos:** el panel debe ser extremadamente simple; validar con pruebas de usabilidad reales con dueños de restaurantes.
- **Multi-tenancy y seguridad de datos:** un fallo de aislamiento entre tenants sería crítico (un local viendo datos de otro). Debe auditarse desde el diseño inicial.
- **Dependencia de terceros para QR:** evaluar generar QR internamente vs. servicio externo, por control y disponibilidad.
- **Estacionalidad de la demanda:** la fuerza de ventas debe considerar picos de interés en temporadas altas de gastronomía.

## 12. Roadmap sugerido (fases)

1. **Fase 1 (MVP):** onboarding, panel básico de menú, QR, vista pública, un solo idioma.
2. **Fase 2:** multi-idioma, planes de suscripción y facturación, roles Editor.
3. **Fase 3:** QR por mesa con analítica de escaneos, importación de menú desde PDF/imagen.
4. **Fase 4:** pedidos desde el QR, integraciones con POS, fidelización.

## 13. Preguntas abiertas para el equipo

- ¿Se requiere soporte para pedidos y pagos en el MVP o queda estrictamente para fases futuras?
- ¿Qué pasarela de pagos y qué modelo de facturación (mensual, anual, por local) se usará para las suscripciones?
- ¿Es necesario soporte offline / fallback en la vista pública ante falta de conectividad del cliente?
- ¿Se requiere importación automática de menús existentes (PDF, Excel) para acelerar el onboarding?
- ¿Cuál es el mercado geográfico inicial (define moneda, idioma por defecto y normativa de datos aplicable)?

---

*Documento vivo — sujeto a revisión conforme avancen las validaciones de negocio y las pruebas con usuarios reales.*
