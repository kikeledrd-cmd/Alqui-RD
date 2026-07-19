# Alqui-RD — Diseño funcional y técnico del MVP

**Fecha:** 19 de julio de 2026
**Estado:** Aprobado por el usuario para planificación técnica
**Producto:** Plataforma inmobiliaria web progresiva para el Gran Santo Domingo
**Modelo:** SaaS modular + marketplace controlado + generación de oportunidades

## 1. Resumen ejecutivo

Alqui-RD será una plataforma inmobiliaria dominicana enfocada principalmente en alquileres residenciales y, como categoría secundaria, en propiedades en venta. Permitirá que visitantes encuentren inmuebles por ubicación, tipo, precio y características; que agentes verificados administren sus publicaciones y contactos; y que agencias coordinen equipos desde una cuenta corporativa.

El MVP se construirá como una PWA adaptable a celulares y computadoras. La arquitectura será modular para que los módulos de pagos automáticos, inteligencia artificial, análisis de precios y expansión nacional puedan incorporarse posteriormente sin rehacer el núcleo.

## 2. Objetivos del MVP

1. Centralizar inventario inmobiliario residencial del Gran Santo Domingo.
2. Facilitar búsquedas por provincia, municipio, sector, operación y características.
3. Permitir que agentes aprobados publiquen propiedades mediante un panel sencillo.
4. Registrar los leads generados, incluso cuando el contacto continúe por WhatsApp.
5. Mantener el inventario actualizado mediante renovaciones inteligentes.
6. Crear una base comercial monetizable mediante planes, destacados y comisiones opcionales.
7. Generar confianza mediante perfiles profesionales y verificaciones por niveles.

## 3. Alcance territorial inicial

El lanzamiento cubrirá:

- Distrito Nacional.
- Santo Domingo Este.
- Santo Domingo Norte.
- Santo Domingo Oeste.

La estructura territorial deberá admitir posteriormente municipios adicionales, provincias y zonas turísticas sin migraciones destructivas.

## 4. Modelo de usuarios y permisos

### 4.1 Visitante

Puede navegar sin registro, buscar propiedades, aplicar filtros, abrir fichas, ver perfiles de agentes, compartir anuncios, contactar por WhatsApp y solicitar visitas.

### 4.2 Usuario registrado

Además de las funciones del visitante, puede guardar favoritos, comparar propiedades, configurar alertas y consultar su historial de solicitudes.

### 4.3 Agente pendiente de aprobación

Puede registrarse, verificar correo o teléfono, completar su perfil y enviar documentos. No puede publicar propiedades hasta que el administrador apruebe su cuenta y active su plan.

### 4.4 Agente verificado

Puede administrar su perfil, publicar propiedades, gestionar leads, confirmar disponibilidad, coordinar visitas y consultar estadísticas básicas.

### 4.5 Agencia inmobiliaria

La cuenta de agencia admite varios usuarios y los roles administrador de agencia, supervisor y agente. Puede distribuir propiedades y contactos, administrar el inventario del equipo y consultar métricas agregadas.

### 4.6 Administrador Alqui-RD

Controla usuarios, verificaciones, moderación, propiedades, territorios, planes, comprobantes, destacados, leads, visitas, reportes y asignaciones de propiedades captadas a particulares.

## 5. Oferta inmobiliaria del MVP

### 5.1 Operaciones

- Alquiler, como propuesta principal.
- Venta, como categoría secundaria.

### 5.2 Tipos de propiedades

- Apartamento.
- Casa.
- Habitación.
- Estudio.
- Penthouse.
- Villa.

Los campos visibles y obligatorios deberán variar según el tipo de propiedad.

## 6. Portal público

### 6.1 Página de inicio

Contendrá:

- Selector Alquiler/Venta.
- Provincia o municipio.
- Sector.
- Tipo de propiedad.
- Rango de precio en pesos dominicanos.
- Botón principal “Buscar inmuebles”.
- Propiedades destacadas.
- Propiedades recientes.
- Sectores populares.
- Llamado “Publica tus propiedades” para agentes.
- Llamado “Tengo una propiedad” para propietarios particulares.

### 6.2 Resultados de búsqueda

El usuario podrá filtrar por:

- Operación.
- Provincia, municipio y sector.
- Tipo de inmueble.
- Precio mínimo y máximo.
- Habitaciones.
- Baños.
- Parqueos.
- Amueblado.
- Verificación.

Los resultados podrán ordenarse por fecha, menor precio, mayor precio, popularidad y nivel de verificación.

Cada tarjeta mostrará fotografía, precio, operación, tipo, sector, municipio, habitaciones, baños, parqueos, estado de verificación y agente responsable.

### 6.3 Ficha de propiedad

Incluirá:

- Carrusel de fotografías.
- Precio y operación.
- Título y descripción.
- Características y amenidades.
- Condiciones de alquiler o venta.
- Ubicación exacta, aproximada o limitada al sector.
- Sellos de verificación.
- Perfil del agente.
- Propiedades similares.
- Botón fijo de WhatsApp con mensaje y código de propiedad precargados.
- Formulario para solicitar visita.
- Botón de compartir.
- Código único del inmueble.

## 7. Mini oficina digital del agente

Cada agente tendrá una página pública con:

- Foto o logo.
- Nombre profesional.
- Inmobiliaria, cuando aplique.
- Biografía.
- WhatsApp y redes sociales.
- Zonas donde trabaja.
- Propiedades activas.
- Reseñas habilitables en una fase posterior.
- Distintivos de verificación.
- Tiempo de respuesta calculado cuando exista suficiente información.

## 8. Panel del agente

El menú inicial contendrá:

- Resumen.
- Mis propiedades.
- Crear propiedad.
- Contactos.
- Visitas.
- Mi perfil.
- Verificación.
- Plan y pagos.

El resumen mostrará propiedades activas, publicaciones pendientes, inmuebles que requieren confirmación, contactos nuevos, visitas próximas, vistas acumuladas y estado del plan.

## 9. Creación y administración de propiedades

El formulario será un asistente por pasos:

1. **Operación:** alquiler o venta, tipo, precio, moneda, depósito y condiciones.
2. **Ubicación:** provincia, municipio, sector, dirección privada, coordenadas y privacidad pública.
3. **Características:** habitaciones, baños, parqueos, metros, piso, amueblado y amenidades.
4. **Contenido:** título, descripción, requisitos, servicios incluidos y disponibilidad.
5. **Fotografías:** carga múltiple, portada, orden, eliminación y validaciones.
6. **Revisión:** vista previa, declaraciones de responsabilidad y envío.

Estados de la propiedad:

- Borrador.
- En revisión.
- Requiere cambios.
- Publicada.
- Pendiente de confirmación.
- Pausada.
- Reservada.
- Alquilada.
- Vendida.
- Rechazada.
- Archivada.

## 10. Moderación y confianza progresiva

Las primeras publicaciones de cada agente requerirán revisión administrativa. El sistema conservará un nivel interno de confianza basado en calidad, historial, reportes, actualización de disponibilidad y cumplimiento.

Los agentes con buen historial podrán obtener publicación automática. La administración conservará la capacidad de revisar, pausar o devolver cualquier anuncio.

Motivos de corrección o rechazo:

- Información incompleta.
- Imágenes deficientes, falsas o repetidas.
- Precio sospechoso.
- Dirección o ubicación inconsistente.
- Propiedad duplicada.
- Contenido engañoso.
- Denuncias sin resolver.

## 11. Verificación por niveles

Cada propiedad podrá mostrar los sellos que correspondan:

- Identidad verificada.
- Documentación revisada.
- Ubicación confirmada.
- Propiedad visitada.
- Fotografías verificadas.

Los sellos indican el alcance de la validación realizada, pero no constituyen garantía legal de la operación.

## 12. Leads, WhatsApp y trazabilidad

El modelo será híbrido. El usuario podrá escribir directamente al agente, pero el clic se registrará antes de abrir WhatsApp.

Cada lead almacenará:

- Propiedad consultada.
- Agente y agencia responsables.
- Usuario registrado o datos mínimos del visitante.
- Canal de origen.
- Fecha y hora.
- Medio de contacto.
- Estado comercial.
- Historial de cambios.
- Resultado final.
- Comisión aplicable, cuando corresponda.

Estados sugeridos:

- Nuevo.
- Contactado.
- Visita solicitada.
- Visita pautada.
- Visita realizada.
- Negociación.
- Cerrado.
- No interesado.
- No localizado.
- Descartado.

## 13. Agenda híbrida de visitas

El interesado podrá contactar por WhatsApp o solicitar una visita desde la web. La solicitud contendrá fecha preferida, bloque horario y datos de contacto.

El agente podrá confirmar, proponer un cambio, cancelar o marcar la visita como realizada. Se registrará el resultado: asistió, no asistió, interesado, negociación o cierre.

## 14. Renovación inteligente del inventario

- Alquileres: confirmación cada 15 días.
- Ventas: confirmación cada 30 días.

El sistema enviará recordatorios antes del vencimiento. Si el agente no confirma, la propiedad pasará a “pendiente de confirmación” y después se pausará. La reactivación será inmediata al confirmar disponibilidad, salvo que exista una revisión administrativa pendiente.

## 15. Propietarios particulares y asignación de oportunidades

Un propietario no profesional podrá enviar su inmueble mediante un formulario. La propiedad no se publicará directamente.

El administrador validará los datos y recibirá una recomendación de agentes basada en:

- Zona de trabajo.
- Tipo de inmueble.
- Rendimiento.
- Nivel de confianza.
- Velocidad de respuesta.
- Carga de oportunidades.
- Plan contratado, con peso limitado.
- Conflictos o duplicidad.

El administrador confirmará, modificará o reasignará la recomendación. Todas las asignaciones quedarán auditadas.

## 16. Monetización

### 16.1 Estructura escalonada

- Plan de prueba.
- Plan Agente.
- Plan Agente Pro.
- Plan Agencia.
- Extras por destacados y mayor visibilidad.

Los límites exactos y precios quedan fuera del MVP de diseño y se definirán después de validar costos y disposición de pago.

### 16.2 Pagos

El MVP aceptará transferencia bancaria y carga de comprobante. El administrador podrá aprobar o rechazar el pago y activar el plan.

La arquitectura deberá incluir una abstracción de pasarela para integrar pagos automáticos en una fase posterior.

### 16.3 Comisión por cierre

Las propiedades publicadas por agentes con suscripción no tendrán comisión obligatoria. Las propiedades captadas directamente por Alqui-RD podrán generar una participación previamente acordada y registrada.

## 17. Arquitectura modular recomendada

### 17.1 Aplicación pública

Responsable de inicio, búsqueda, resultados, ficha, perfiles, favoritos, alertas y formularios públicos.

### 17.2 Consola profesional

Responsable de los paneles de agente y agencia, propiedades, leads, visitas, perfil y pagos.

### 17.3 Consola administrativa

Responsable de aprobaciones, moderación, territorios, planes, comprobantes, asignaciones, verificaciones y reportes.

### 17.4 Servicios de dominio

- Identidad y permisos.
- Catálogo inmobiliario.
- Búsqueda y filtros.
- Medios e imágenes.
- Leads y visitas.
- Suscripciones y pagos.
- Verificaciones.
- Notificaciones.
- Auditoría.
- Analítica.

Cada servicio deberá exponer interfaces claras y evitar dependencia directa de componentes visuales.

## 18. Entidades principales

- User.
- AgentProfile.
- Agency.
- AgencyMembership.
- VerificationRequest.
- Property.
- PropertyMedia.
- PropertyFeature.
- Location.
- PropertyVerification.
- Favorite.
- SavedSearch.
- Lead.
- LeadActivity.
- VisitRequest.
- SubscriptionPlan.
- Subscription.
- Payment.
- Promotion.
- OwnerSubmission.
- AssignmentRecommendation.
- Notification.
- AuditLog.

## 19. Flujo principal de datos

```text
Visitante ejecuta una búsqueda
        ↓
El motor aplica filtros territoriales y comerciales
        ↓
El visitante abre una ficha
        ↓
Se registra la visualización
        ↓
Contacta por WhatsApp o solicita una visita
        ↓
Se crea el lead y se asigna al agente responsable
        ↓
El agente actualiza el seguimiento
        ↓
Se registra visita, negociación y cierre
        ↓
La propiedad cambia de disponibilidad
        ↓
Alqui-RD actualiza métricas y comisión aplicable
```

## 20. Manejo de errores y reglas críticas

- Las cargas de fotografías deben aceptar reintentos y no perder el formulario.
- Una propiedad no puede publicarse sin precio, ubicación mínima, portada y datos obligatorios.
- Los errores de WhatsApp no deben impedir que el lead quede registrado.
- Un pago manual no activa el plan hasta ser aprobado.
- Una cuenta suspendida no puede publicar ni recibir nuevas asignaciones.
- Las direcciones privadas nunca deben exponerse por errores de interfaz o API.
- Las acciones administrativas sensibles deberán quedar en AuditLog.
- Los cambios de estado deben validarse para impedir transiciones incoherentes.
- Las búsquedas sin resultados deberán sugerir ampliar zonas, precios o características.

## 21. Seguridad y privacidad

- Control de acceso por roles y permisos.
- Verificación de correo o teléfono.
- Protección contra carga de archivos maliciosos.
- URLs firmadas o privadas para documentos sensibles.
- Separación entre dirección pública y privada.
- Registro de consentimiento para comunicaciones.
- Limitación de intentos y protección contra abuso.
- Backups, logs y políticas de retención.
- No exponer documentos de identidad a agentes ni visitantes.

## 22. Analítica mínima

El MVP medirá:

- Propiedades activas por zona y tipo.
- Búsquedas y filtros más utilizados.
- Vistas por propiedad.
- Clics a WhatsApp.
- Solicitudes de visita.
- Leads por agente.
- Tiempo de respuesta.
- Conversión a visita y cierre.
- Propiedades vencidas o pausadas.
- Ingresos por planes, pagos y destacados.

## 23. PWA y experiencia móvil

La plataforma será mobile-first y podrá instalarse desde navegadores compatibles. El MVP priorizará velocidad, formularios fáciles de completar, botones de contacto visibles, imágenes optimizadas y navegación usable con una mano.

Las notificaciones push podrán activarse en una iteración posterior del MVP cuando exista consentimiento y una estrategia clara de alertas.

## 24. Pruebas y criterios de aceptación

### 24.1 Pruebas funcionales

- Registro y aprobación de agentes.
- Restricción de publicación para agentes no aprobados.
- Creación, edición y moderación de propiedades.
- Búsqueda combinada por ubicación, operación, tipo y precio.
- Carrusel y carga de imágenes.
- Registro de clics a WhatsApp.
- Solicitud y gestión de visitas.
- Confirmación y pausa automática de propiedades.
- Carga y aprobación de comprobantes.
- Permisos de agencia.

### 24.2 Pruebas no funcionales

- Diseño adaptable en dispositivos móviles y escritorio.
- Tiempo de respuesta aceptable en búsquedas comunes.
- Imágenes optimizadas y carga progresiva.
- Accesibilidad básica de formularios y navegación.
- Protección de información privada.
- Recuperación ante fallos de red durante formularios.

### 24.3 Criterio de lanzamiento

El MVP estará listo cuando un agente aprobado pueda publicar un inmueble, el administrador pueda moderarlo, un visitante pueda encontrarlo, abrir su carrusel, contactar por WhatsApp o solicitar una visita, y el sistema pueda registrar y seguir esa oportunidad de principio a fin.

## 25. Fuera de alcance en la primera entrega

- Aplicaciones móviles nativas.
- IA avanzada para precios o fraude.
- Pagos automáticos con tarjeta.
- Firma digital de contratos.
- Chat interno.
- Integración bancaria completa.
- Visitas virtuales 3D.
- Gestión legal de contratos.
- Motor avanzado de comisiones.
- Expansión nacional completa.

## 26. Fases posteriores previstas

### Fase 2

Leads avanzados, visitas, agencias, planes, pagos manuales, destacados y reportes operativos.

### Fase 3

Pagos automáticos, verificaciones ampliadas, automatizaciones de WhatsApp, recomendaciones y reglas de confianza.

### Fase 4

Análisis de precios, inteligencia artificial inmobiliaria, prevención de fraude, expansión nacional y aplicaciones móviles cuando la tracción lo justifique.

## 27. Decisiones aprobadas

- Modelo de agentes y agencias aprobadas.
- Monetización híbrida.
- Lanzamiento en Gran Santo Domingo.
- Registro abierto con publicación bloqueada.
- Alquileres como enfoque y ventas como categoría secundaria.
- Leads híbridos con WhatsApp y trazabilidad.
- Catálogo residencial ampliado.
- Registro opcional para visitantes.
- Privacidad de ubicación configurable.
- Pagos manuales inicialmente, con arquitectura para pagos automáticos.
- Confianza progresiva para publicaciones.
- Mini oficina digital para agentes.
- Renovación cada 15 días en alquileres y 30 días en ventas.
- Cuentas de agentes independientes y agencias.
- Propietarios particulares captados y asignados a agentes.
- Asignación inteligente híbrida con confirmación administrativa.
- Verificación por niveles.
- Agenda híbrida.
- PWA primero.
- Comisión opcional en oportunidades captadas por Alqui-RD.
- Planes escalonados.
- Arquitectura SaaS modular.
