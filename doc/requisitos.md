# Requisitos

**Proyecto:** Sistema interno de facturación y cobros de Turbine  
**Asignatura:** Análisis y Diseño de Aplicaciones (curso 2026-27)

## 1. Contexto y alcance

### 1.1. Objetivo del sistema

Turbine es un laboratorio de inteligencia artificial en fase inicial (seed) cuyo producto, Brain, se factura mediante una suscripción mensual que depende del rango de facturación anual de cada cliente, además de servicios puntuales (por ejemplo, instalaciones locales). Hoy la gestión de clientes, facturas y cobros se hace de forma manual y está concentrada en una sola persona.

El objetivo del sistema es disponer de una **aplicación web interna de administración** que permita dar de alta clientes, generar y enviar facturas (recurrentes y puntuales), hacer seguimiento de los cobros y de los impagos, y mantener informada a la persona correcta en cada momento. Debe ser rápida de manejar (pocos clics), utilizable desde cualquier sitio y dispositivo, y capaz de crecer hasta miles de clientes sin rehacerse, de modo que el futuro departamento administrativo sea más eficiente.

### 1.2. Alcance

El sistema es responsable de:

- Gestión de clientes (datos fiscales y de contacto, estado, responsable interno, etiquetas, canales de comunicación, historial) e importación desde CSV.
- Gestión de suscripciones mensuales (rango de cobro, excepciones comerciales, pausas, bajas y reactivaciones) y de facturas puntuales, presupuestos y facturas rectificativas.
- Generación, numeración, emisión, envío (correo electrónico y, como canal complementario, WhatsApp) y generación en PDF de las facturas.
- Seguimiento de cobros y estado de las facturas, conciliación de pagos, recordatorios, flujo de impagos e incidencias de facturación.
- Usuarios internos, roles, permisos, asignación de clientes, aprobaciones, notificaciones y pantalla de trabajo diario.
- Informes, dashboard financiero, exportaciones (incluido el paquete para la gestoría) y cierre de periodos.
- Configuración global, plantillas de comunicación, auditoría, registros de ejecución y monitorización del sistema.
- Integración simulada con Stripe y API documentada para que otros sistemas (por ejemplo, Brain) puedan consultar información en el futuro.

Quedan explícitamente **fuera del alcance**:

- El sistema **no es una pasarela de pago**: solo comprueba, consulta y registra los cobros.
- No es una extensión ni una pantalla de Brain: es una aplicación **independiente** para la administración interna de Turbine. No se integra con Brain en esta fase; solo se deja preparada una API documentada.
- No es un portal para los clientes de Turbine: los clientes no se registran ni acceden a la aplicación (la atención a sus consultas se hace por canales externos, como el soporte).
- No se integran servicios reales de Stripe, correo ni WhatsApp en esta fase: las integraciones son simuladas y encapsuladas para poder sustituirlas.
- No se implementa la integración con Verifactu ni la deducción automática de la fiscalidad de cada país: las reglas fiscales concretas las marca la gestoría.
- No se incluye un chat propio dentro de la aplicación ni un sistema de tickets de soporte para los clientes.

### 1.3. Actores y partes interesadas

| Actor / stakeholder | Descripción | Intereses principales |
| -------------------- | ------------------------------------------------ | -------------------------------- |
| Superadministrador | Usuario interno de Turbine con acceso prácticamente total a la aplicación y con poder de configuración profunda (usuarios, parámetros globales, plantillas, tarifas, series). | Controlar la configuración y los usuarios del sistema y disponer de trazabilidad completa de lo que ocurre. |
| Administrador | Usuario interno que gestiona operaciones, clientes, facturas, asignaciones y aprobaciones según sus permisos; accede al dashboard financiero y a los informes. | Gestionar la operación de forma ágil, aprobar cambios sensibles y conocer la situación financiera. |
| Usuario operativo | Usuario interno que trabaja sobre los clientes que tiene asignados (consultar y modificar información operativa, crear facturas si su permiso lo permite, seguir cobros y descargar documentos). | Realizar sus tareas administrativas con pocos clics y que la información le llegue sin que se le pase nada. |
| Responsable interno del cliente | Usuario interno (administrador o usuario operativo) encargado de la facturación de un cliente concreto y destinatario de las notificaciones de ese cliente. | Recibir a tiempo los eventos relevantes (pagos, vencimientos, fallos de envío, aprobaciones) y resolverlos. |
| Desarrollador / equipo técnico | Persona que mantiene y despliega la aplicación y recibe los errores técnicos con su identificador. | Diagnosticar fallos con facilidad y mantener el sistema de forma sencilla. |
| Cliente de Turbine | Empresa a la que Turbine presta sus servicios y factura; no accede a la aplicación. | Recibir correctamente sus facturas y comunicaciones sobre los pagos y disponer de un contacto de soporte. |
| Persona de contacto del cliente | Persona de la empresa cliente destinataria de las facturas y comunicaciones (contacto de facturación u otros contactos). | Recibir las facturas y avisos en el canal correcto. |
| Sistema de pagos (Stripe, simulado) | Sistema externo en el que se reflejan los cobros; en esta fase es una integración simulada. | Proporcionar los eventos y consultas que permitan saber si una factura ha sido pagada. |
| Proveedores de correo y WhatsApp (simulados) | Servicios externos (simulados) a través de los cuales se envían facturas y avisos. | Entregar los mensajes y devolver el resultado del envío. |
| Sistema externo vía API (por ejemplo, Brain) | Sistema de Turbine u otro que podrá consultar información (por ejemplo, si un cliente tiene facturas pendientes) mediante la API documentada. | Consultar datos de forma controlada sin saltarse las reglas de la aplicación. |
| Gestoría | Asesoría externa que fija las reglas legales y recibe periódicamente el paquete de facturas y cobros; no tiene usuario en la aplicación. | Recibir información completa, ordenada y versionada de cada periodo. |
| Procesos automáticos del sistema | Tareas programadas del propio sistema (generación mensual de facturas, recordatorios, conciliación, informes programados). | Ejecutarse de forma fiable, reintentable y sin duplicar resultados. |

## 2. Requisitos de negocio

Los requisitos de negocio describen los objetivos, necesidades o resultados que la organización pretende conseguir mediante el desarrollo del sistema.

| ID | Requisito | Fuente |
| -------- | ------------------------------------------------------------------------ | -------------------- |
| BR-01 | Reducir el trabajo manual asociado a la generación de facturas y al seguimiento de los cobros, automatizando los procesos generales y frecuentes y dejando la revisión humana para las excepciones. | Cliente / entrevista |
| BR-02 | Hacer que las operaciones administrativas frecuentes (alta de cliente, factura normal o recurrente, comprobación de estado) sean muy rápidas, con el mínimo de clics y en menos de un minuto. | Cliente / entrevista |
| BR-03 | Poder operar el sistema desde cualquier sitio y dispositivo, no solo desde un ordenador de sobremesa. | Cliente / entrevista |
| BR-04 | Garantizar que la información fluya con fluidez y llegue siempre a la persona correcta, sin que ningún dato se pierda por el camino. | Cliente / entrevista |
| BR-05 | Disponer de una automatización fiable, trazable y explicable: saber qué ha pasado, quién lo ha hecho y por qué, y detectar los fallos en lugar de que ocurran en silencio. | Cliente / entrevista; decisión del equipo |
| BR-06 | Mantener el control económico del negocio: tarifas según el rango de facturación anual de cada cliente, excepciones comerciales controladas y cambios sensibles aprobados. | Cliente / entrevista |
| BR-07 | Poder crecer hasta miles de clientes y cientos de usuarios internos sin tener que rehacer el sistema. | Cliente / entrevista |
| BR-08 | Conocer en todo momento la situación económica: qué se ha facturado, cobrado, está pendiente o vencido y qué se espera ingresar. | Cliente / entrevista |
| BR-09 | Cumplir la normativa fiscal española y el RGPD, conservar los datos el tiempo que exige la ley y facilitar el trabajo con la gestoría. | Cliente / entrevista |
| BR-10 | Proteger la confidencialidad e integridad de la información sensible (clientes, facturas, documentos) y limitar el acceso a las personas autorizadas. | Cliente / entrevista |
| BR-11 | Disponer de un sistema simple, mantenible y evolutivo, con integraciones sustituibles y una API documentada, independiente del producto Brain. | Cliente / entrevista |
| BR-12 | Reducir los impagos y asegurar que las deudas se recobran, también cuando un cliente se da de baja. | Cliente / entrevista |
| BR-13 | Transmitir una imagen corporativa coherente en la aplicación y en los documentos y comunicaciones enviados a los clientes. | Cliente / entrevista |

## 3. Requisitos de usuario

Los requisitos de usuario describen las necesidades, objetivos o servicios que los usuarios esperan obtener del sistema.

| ID | Actor | Requisito | Fuente |
| ------- | -------------------- | ----------------------------------------------------- | -------------------- |
| UR-01 | Todos los usuarios internos | Poder iniciar sesión de forma segura con su correo corporativo y recuperar su contraseña si la olvidan. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-02 | Superadministrador | Poder crear, invitar y desactivar usuarios, y asignarles rol y permisos. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-03 | Administrador | Poder asignar y desasignar clientes a los usuarios operativos sin perder el histórico ni las tareas pendientes. | Cliente / entrevista (Día 1, Ampliados) |
| UR-04 | Superadministrador | Poder configurar los parámetros globales del sistema (plazos, cadencia de recordatorios, series, impuestos, monedas, umbrales de aprobación, canales y políticas de automatización) y simular los cambios delicados antes de aplicarlos. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-05 | Usuario operativo / Administrador | Poder dar de alta y gestionar clientes de forma rápida (datos fiscales y de contacto, país y moneda, canales, estado, etiquetas) evitando duplicados y consultando su historial. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-06 | Usuario operativo / Administrador | Poder importar clientes desde un fichero CSV con control previo de los datos antes de confirmar. | Cliente / entrevista (Día 1) |
| UR-07 | Usuario operativo / Administrador | Poder gestionar la suscripción de un cliente: rango de cobro, excepciones comerciales, prorrateo, pausas y reactivación. | Cliente / entrevista (Día 2) |
| UR-08 | Responsable interno / Administrador | Poder revisar cada año el rango de facturación de los clientes y aprobar los cambios de rango. | Cliente / entrevista (Día 2, Ampliados) |
| UR-09 | Usuario operativo / Administrador | Poder crear facturas puntuales (con varias líneas), reutilizar conceptos frecuentes y preparar presupuestos que se conviertan en factura una vez aceptados. | Cliente / entrevista (Día 1, Ampliados) |
| UR-10 | Usuario operativo / Administrador | Poder revisar una vista previa y emitir facturas con numeración única y un PDF fiel a la imagen de Turbine, con aviso de cualquier dato que falte. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-11 | Administrador | Poder corregir una factura antes de enviarla y rectificarla con trazabilidad cuando ya no pueda modificarse. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-12 | Procesos automáticos / Todos los usuarios | Que las facturas recurrentes se generen y envíen automáticamente cada mes (con prorrateo cuando corresponda) sin intervención manual. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-13 | Usuario operativo / Administrador | Que las situaciones excepcionales (descuentos, cambios no habituales, importes elevados) se revisen y aprueben antes de enviar la facturación. | Cliente / entrevista (Día 2, Ampliados) |
| UR-14 | Administrador / Superadministrador | Poder revisar y resolver, desde una bandeja, las solicitudes de aprobación pendientes con su comentario. | Cliente / entrevista (Ampliados) |
| UR-15 | Usuario operativo / Administrador | Poder conocer en cada momento qué facturas están pagadas, pendientes o impagadas, y revisar los pagos cuyo importe no coincide. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-16 | Sistema de pagos (Stripe, simulado) | Poder informar de los pagos recibidos y permitir su consulta para que el sistema determine si una factura ha sido pagada. | Cliente / entrevista (Día 1, Ampliados) |
| UR-17 | Usuario operativo / Administrador | Que los clientes reciban avisos y recordatorios de pago de forma automática y poder suspenderlos temporalmente cuando proceda. | Cliente / entrevista (Día 1, Ampliados) |
| UR-18 | Administrador | Poder cancelar o declarar incobrable una deuda mediante una acción controlada, motivada y aprobada. | Cliente / entrevista (Día 1, Ampliados) |
| UR-19 | Usuario operativo / Administrador | Poder registrar la baja de un cliente y mantener el recobro de sus deudas pendientes, y reactivar su ficha si vuelve. | Cliente / entrevista (Día 1, Ampliados) |
| UR-20 | Responsable interno / Usuario operativo | Poder saber si las facturas y avisos se han enviado correctamente y gestionar los fallos de entrega. | Cliente / entrevista (Ampliados) |
| UR-21 | Responsable interno / Administrador | Recibir notificaciones relevantes (con distinción entre informativas y pendientes de atender) y poder elegir cuáles recibir. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-22 | Todos los usuarios internos | Disponer al entrar de una pantalla de trabajo diario con lo que requiere atención y poder resolverlo con pocos clics. | Cliente / entrevista (Día 1, Ampliados) |
| UR-23 | Todos los usuarios internos | Poder buscar y filtrar clientes y facturas, guardar filtros, organizar con etiquetas y acceder rápido a favoritos o recientes. | Cliente / entrevista (Día 1, Ampliados) |
| UR-24 | Usuario operativo / Administrador | Poder abrir incidencias de facturación y comentar con otros compañeros mencionándolos, sin perder el contexto. | Cliente / entrevista (Ampliados) |
| UR-25 | Usuario operativo / Administrador | Poder adjuntar documentos de apoyo a un cliente o factura de forma segura. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-26 | Administrador | Poder obtener informes de cobros por periodo, filtrarlos y exportarlos respetando mis permisos. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-27 | Administrador | Poder consultar un dashboard financiero con cifras que se puedan explicar, objetivos y previsión basada en las recurrencias. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-28 | Administrador | Poder cerrar periodos administrativos y exportar el paquete mensual o trimestral para la gestoría. | Cliente / entrevista (Día 1, Ampliados) |
| UR-29 | Superadministrador / Administrador | Poder consultar quién hizo cada cambio relevante en la aplicación, sin que se pueda alterar el pasado. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-30 | Administrador | Poder ver el estado de salud del sistema y los registros de ejecución de los procesos automáticos, y recibir resúmenes periódicos. | Cliente / entrevista (Ampliados) |
| UR-31 | Superadministrador / Administrador | Poder editar las plantillas de las comunicaciones a clientes y ver qué versión se usó en cada envío. | Cliente / entrevista (Día 1, Ampliados) |
| UR-32 | Administrador | Poder configurar alertas y disponer de un calendario laboral para organizar los eventos próximos. | Cliente / entrevista (Ampliados) |
| UR-33 | Todos los usuarios internos | Poder realizar desde el móvil las acciones esenciales (consultar clientes, aprobar algo urgente, ver impagos, marcar una incidencia, revisar un pago). | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-34 | Todos los usuarios internos | Disponer de una interfaz consistente, accesible y cómoda, con borradores, valores por defecto, avisos claros y acciones masivas con control. | Cliente / entrevista (Día 1, Día 2, Ampliados) |
| UR-35 | Sistema externo vía API | Poder consultar a través de la API documentada si un cliente tiene facturas pendientes y registrar solicitudes de baja, sin saltarse las reglas de la aplicación. | Cliente / entrevista (Ampliados) |
| UR-36 | Desarrollador / equipo técnico | Poder diagnosticar los errores con su identificador y mantener, probar y desplegar el sistema con facilidad. | Cliente / entrevista (Día 1, Ampliados) |
| UR-37 | Administrador / Todos los usuarios internos | Que la aplicación siga siendo ágil y manejable cuando crezca el número de clientes, de facturas históricas y de usuarios internos. | Cliente / entrevista (Día 1, Ampliados) |

## 4. Requisitos del sistema

### 4.1. Requisitos funcionales

Los requisitos funcionales describen comportamientos, servicios u operaciones que debe proporcionar el sistema.


#### 4.1.1. Acceso, usuarios, roles y permisos

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-001 | El sistema deberá permitir iniciar sesión mediante correo electrónico y contraseña. | UR-01 |
| FR-002 | El sistema deberá exigir que el correo electrónico de acceso pertenezca al dominio corporativo de Turbine y rechazar los demás (el documento original lo expresa como «correo de TurbineH»; véase Q-01). | UR-01, UR-04 |
| FR-003 | El sistema deberá permitir al superadministrador crear usuarios. | UR-02 |
| FR-004 | El sistema deberá permitir al superadministrador invitar usuarios. | UR-02 |
| FR-005 | El sistema deberá permitir al superadministrador desactivar cuentas de usuario. | UR-02 |
| FR-006 | El sistema deberá permitir recuperar la contraseña mediante enlaces temporales. | UR-01 |
| FR-007 | El sistema deberá distinguir tres roles de usuario: superadministrador, con prácticamente acceso total a la aplicación; administrador, que puede gestionar operaciones, clientes, facturas, asignaciones y aprobaciones según sus permisos; y usuario operativo, que trabaja sobre lo que tiene asignado. | UR-02 |
| FR-008 | El sistema deberá permitir asignar roles y permisos a los usuarios. | UR-02 |
| FR-009 | El sistema deberá permitir que un usuario operativo tenga asignados clientes concretos. | UR-03 |
| FR-010 | El sistema deberá permitir al usuario operativo ver y modificar la información operativa de los clientes que tiene asignados. | UR-05 |
| FR-011 | El sistema deberá permitir al usuario operativo crear facturas de sus clientes asignados cuando su permiso lo permita. | UR-09 |
| FR-012 | El sistema deberá permitir al usuario operativo seguir los cobros de sus clientes asignados. | UR-15 |
| FR-013 | El sistema deberá permitir al usuario operativo descargar los documentos de sus clientes asignados. | UR-25 |
| FR-014 | El sistema no deberá permitir al usuario operativo darse a sí mismo acceso a otro cliente, cambiar tarifas globales, crear series de facturación ni ver el dashboard financiero global. | UR-03, UR-02 |
| FR-015 | El sistema deberá permitir al administrador asignar y desasignar clientes a los usuarios. | UR-03 |
| FR-016 | Cuando cambie la asignación de un cliente, el sistema deberá transferir las tareas pendientes al nuevo responsable. | UR-03 |
| FR-017 | Cuando cambie la asignación de un cliente, el histórico deberá seguir indicando quién realizó cada acción anteriormente. | UR-03, UR-29 |

#### 4.1.2. Clientes, datos fiscales e importación

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-018 | El sistema deberá permitir registrar los datos de un cliente necesarios para la facturación (nombre, NIF, correo, teléfono, etc.). | UR-05 |
| FR-019 | Al crear un cliente, el sistema deberá comprobar que su creación es posible para evitar duplicados y confirmar que se rellenan los campos necesarios. | UR-05 |
| FR-020 | El sistema deberá detectar posibles clientes duplicados por identificador fiscal y avisar al usuario. | UR-05 |
| FR-021 | El sistema no deberá fusionar empresas automáticamente. | UR-05 |
| FR-022 | Al registrar el NIF, el sistema deberá permitir especificar el país y la moneda del cliente. | UR-05 |
| FR-023 | El impuesto establecido según el país deberá poder configurarse o seleccionarse. | UR-05, UR-04 |
| FR-024 | El sistema deberá registrar, por cada cliente, los canales de comunicación permitidos. | UR-05, UR-20 |
| FR-025 | El sistema deberá representar el estado de cada cliente en relación con la facturación. | UR-05 |
| FR-026 | El sistema deberá mantener un historial de eventos relevantes asociados y relacionados con cada cliente. | UR-05 |
| FR-027 | El sistema deberá asociar a cada cliente un responsable interno encargado de su facturación (véase A-04). | UR-05, UR-21 |
| FR-028 | El sistema deberá permitir importar clientes desde un fichero CSV. | UR-06 |
| FR-029 | El sistema deberá mostrar una previsualización de la importación antes de que el usuario la confirme. | UR-06 |
| FR-030 | El sistema deberá permitir al usuario mapear las columnas del fichero CSV. | UR-06 |
| FR-031 | El sistema deberá mostrar al usuario los errores detectados en el fichero CSV. | UR-06 |
| FR-032 | El sistema deberá permitir al usuario decidir si corrige el fichero o importa solo las filas válidas, según la política definida. | UR-06 |
| FR-033 | El sistema deberá avisar cuando, durante la importación, detecte un NIF duplicado. | UR-06 |
| FR-034 | El sistema deberá registrar la importación completa como una operación, con un resumen de clientes creados, actualizados, rechazados y los motivos. | UR-06 |

#### 4.1.3. Suscripciones, tarifas y prorrateo

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-035 | El sistema deberá permitir configurar el rango de cobro del cliente (de anual a mensual, de mensual a anual, etc.; véase Q-07). | UR-07 |
| FR-036 | El sistema deberá avisar de la revisión necesaria cuando se produzca un cambio de rango. | UR-07, UR-08 |
| FR-037 | El sistema deberá permitir registrar excepciones al precio, como descuentos, modalidades gratuitas o precio pactado. | UR-07 |
| FR-038 | Cuando el responsable apruebe el cambio de rango, el sistema deberá registrar quién lo aprobó, cuándo y cuál era el precio anterior y el nuevo. | UR-08, UR-29 |
| FR-039 | El sistema deberá poder listar los clientes cuya facturación anual debe revisarse. | UR-08 |
| FR-040 | Ante un cambio de rango, un responsable deberá revisarlo y aprobarlo. | UR-08, UR-14 |
| FR-041 | El sistema deberá permitir pausar temporalmente una suscripción, registrando el motivo, la fecha de inicio y la fecha prevista de reactivación. | UR-07 |
| FR-042 | Durante la pausa, el sistema no deberá generar las mensualidades que correspondan. | UR-07, UR-12 |
| FR-043 | Durante la pausa, el cliente deberá seguir existiendo y podrá tener facturas antiguas pendientes. | UR-07, UR-15 |
| FR-044 | Cuando llegue la fecha prevista de reactivación, el sistema deberá avisar al responsable y no asumir silenciosamente ninguna acción importante si la política configurada exige confirmación. | UR-07, UR-21 |
| FR-045 | El sistema deberá tener en cuenta las mensualidades desde el día 1 de cada mes. | UR-12, UR-07 |
| FR-046 | Si un cliente entra un día distinto del 1, el sistema deberá generar la parte proporcional hasta final de mes y comenzar la mensualidad completa el día 1 del mes siguiente. | UR-07, UR-12 |
| FR-047 | El cálculo del prorrateo deberá ser visible para el usuario. | UR-07 |
| FR-048 | De forma excepcional y debido a acuerdos, el sistema deberá permitir no cobrar el prorrateo, expresando esta situación de forma explícita. | UR-07 |

#### 4.1.4. Facturación: emisión, numeración, PDF, presupuestos y rectificativas

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-049 | El sistema deberá permitir gestionar facturas recurrentes (mensualidades) además de facturas puntuales y generar automáticamente las facturas recurrentes que correspondan a cada suscripción (véase A-05). | UR-12 |
| FR-050 | El sistema deberá permitir registrar facturas puntuales, además de las recurrentes. | UR-09 |
| FR-051 | Cada factura puntual deberá poder incluir líneas con concepto, cantidad, precio unitario, descuento (si aplica) e impuesto. | UR-09 |
| FR-052 | El sistema deberá permitir buscar conceptos frecuentes al contratar un servicio, con un precio orientativo y un impuesto por defecto. | UR-09 |
| FR-053 | Los datos de un concepto frecuente no deberán afectar a las facturas ya creadas. | UR-09 |
| FR-054 | El sistema deberá permitir corregir una factura antes de enviarla. | UR-11 |
| FR-055 | Una vez enviada una factura, el sistema deberá registrar de forma trazable cualquier cambio relevante. | UR-11, UR-29 |
| FR-056 | Cuando una factura no pueda modificarse, el sistema deberá permitir realizar una factura rectificativa que corrija la anterior. | UR-11 |
| FR-057 | El sistema deberá permitir generar documentos (notas de crédito o rectificativas) a partir de una factura, vinculados a ella, que propongan los errores que se han rectificado. | UR-11 |
| FR-058 | La generación de un documento rectificativo requerirá permiso administrativo y aprobación dependiendo del importe. | UR-11, UR-14 |
| FR-059 | El sistema deberá asignar a cada factura un identificador único compuesto por número y serie. | UR-10 |
| FR-060 | El administrador deberá poder configurar la serie de las facturas (véase Q-06). | UR-10, UR-04 |
| FR-061 | El sistema deberá asignar el identificador único de una factura una vez que ésta se ha emitido. | UR-10 |
| FR-062 | El sistema deberá generar una vista previa de la factura antes de emitirla. | UR-10 |
| FR-063 | El sistema deberá bloquear la emisión de una factura cuando falten datos imprescindibles. | UR-10 |
| FR-064 | El sistema deberá bloquear la emisión de una factura cuando se apliquen importes absurdos (véase Q-05). | UR-10 |
| FR-065 | El sistema deberá avisar al usuario de toda situación de bloqueo o incidencia detectada antes de emitir una factura. | UR-10 |
| FR-066 | El sistema deberá poder representar las facturas mediante un documento PDF. | UR-10 |
| FR-067 | El PDF de la factura deberá incluir los datos fiscales, la numeración, las fechas, las líneas, las bases, los impuestos, el total, las condiciones de pago y los datos de contacto de soporte. | UR-10 |
| FR-068 | El sistema deberá permitir crear presupuestos antes de facturas puntuales grandes. | UR-09 |
| FR-069 | El sistema deberá permitir convertir un presupuesto aceptado en factura. | UR-09 |
| FR-070 | El sistema deberá bloquear la modificación de facturas cuando el trimestre esté cerrado (véase Q-08). | UR-28, UR-11 |
| FR-071 | El sistema deberá permitir a los administradores marcar periodos administrativos como cerrados después de revisar un mes o trimestre. | UR-28 |
| FR-072 | En un periodo cerrado, el sistema no deberá permitir cambios ordinarios en el histórico sin un procedimiento excepcional y auditado. | UR-28, UR-29 |

#### 4.1.5. Envío de facturas y comunicaciones

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-073 | El sistema deberá enviar automáticamente las facturas, por defecto por correo electrónico. | UR-12, UR-20 |
| FR-074 | El sistema deberá poder enviar las facturas por WhatsApp como canal complementario, cuando sea posible y se tenga consentimiento. | UR-20 |
| FR-075 | Cuando surjan problemas al enviar la factura por correo, el sistema deberá notificar la incidencia. | UR-20, UR-21 |
| FR-076 | Cuando surjan problemas al enviar la factura por correo, el sistema deberá intentar el reenvío. | UR-20 |
| FR-077 | El sistema deberá registrar los reintentos de envío. | UR-20, UR-29 |
| FR-078 | El sistema deberá mostrar el estado de los mensajes de correo y WhatsApp: en cola, enviados, entregados (si el proveedor lo permite), fallidos y reintentados. | UR-20 |

#### 4.1.6. Cobros, conciliación e integración de pagos

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-079 | El sistema deberá registrar los cobros y el estado de cada factura (pendiente, pagada, impagada). | UR-15 |
| FR-080 | El sistema deberá ofrecer una integración simulada con Stripe que se comporte como si recibiera eventos de pago. | UR-16 |
| FR-081 | El sistema deberá poder consultar pagos a través de la integración simulada con Stripe. | UR-16, UR-15 |
| FR-082 | El sistema deberá poder recibir una notificación simulada de pago procedente de Stripe. | UR-16 |
| FR-083 | Al recibir el importe de una factura, el sistema deberá comprobar que se ha recibido el total de ese importe. | UR-15 |
| FR-084 | Si el importe recibido no coincide con el total, el sistema deberá notificar el dinero faltante o sobrante para su revisión posterior. | UR-15, UR-21 |
| FR-085 | El sistema deberá registrar la conclusión de la revisión de una diferencia de importe. | UR-15, UR-29 |
| FR-086 | El sistema deberá permitir una decisión administrativa de cancelar o considerar incobrable una deuda, como acción muy controlada, con motivo y aprobación. | UR-18, UR-14 |

#### 4.1.7. Recordatorios, impagos e incidencias

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-087 | El sistema deberá enviar correos informando del incumplimiento del pago, con un primer aviso a los 3 días y un segundo aviso a los 5 días (véase A-06). | UR-17 |
| FR-088 | El superadministrador deberá poder configurar los tiempos de los avisos de pago. | UR-17, UR-04 |
| FR-089 | Si un cliente tiene condiciones pactadas a treinta días, su vencimiento deberá prevalecer sobre el valor general al calcular los avisos. | UR-17 |
| FR-090 | Una vez pasado el vencimiento, el sistema deberá continuar el flujo de impago enviando recordatorios periódicos (por defecto cada dos o tres días) hasta que se detecte el pago o una persona intervenga (véase A-07). | UR-17, UR-18 |
| FR-091 | El sistema deberá permitir suspender temporalmente los recordatorios de una factura. | UR-17 |
| FR-092 | El sistema deberá permitir abrir incidencias de facturación asociadas a un cliente y, opcionalmente, a facturas. | UR-24 |
| FR-093 | Cada incidencia deberá tener contexto, responsable, estado, prioridad, comentarios internos y adjuntos. | UR-24 |
| FR-094 | Mientras una incidencia afecte al cobro, el sistema deberá permitir que pause los recordatorios si así se indica. | UR-24, UR-17 |
| FR-095 | Al cerrar una incidencia, el sistema deberá dejar registrada la resolución. | UR-24 |

#### 4.1.8. Aprobaciones y excepciones

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-096 | El sistema deberá exigir que las excepciones (como cambios no habituales o descuentos) sean revisadas antes de enviar la facturación. | UR-13 |
| FR-097 | El sistema deberá ofrecer una bandeja de aprobaciones ante cambios, cada una con su estado (pendiente, aprobada, rechazada) y el comentario del aprobador. | UR-14, UR-13 |

#### 4.1.9. Notificaciones, alertas y pantalla de trabajo diario

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-098 | El sistema deberá disponer de una bandeja de notificaciones. | UR-21 |
| FR-099 | El sistema deberá hacer llegar al responsable encargado de la facturación de un cliente los eventos importantes relacionados con ella. | UR-21 |
| FR-100 | El sistema deberá ofrecer al administrador una bandeja global y permitirle elegir qué notificaciones recibe según su preferencia. | UR-21 |
| FR-101 | El sistema deberá permitir marcar como leídas las notificaciones que sean únicamente informativas. | UR-21 |
| FR-102 | Las notificaciones que requieran atención deberán permanecer pendientes hasta que se atienda lo notificado. | UR-21 |
| FR-103 | Al entrar en la aplicación, el sistema deberá mostrar una pantalla de trabajo diario con las facturas que vencen pronto, las vencidas, los pagos que necesitan conciliación, los clientes con problemas de entrega, las aprobaciones pendientes, las revisiones anuales que tocan y cualquier tarea asignada al usuario. | UR-22 |
| FR-104 | Desde la pantalla de trabajo diario deberá poderse saltar al elemento y resolverlo con pocos clics. | UR-22 |
| FR-105 | El sistema deberá permitir configurar alertas según eventos concretos o preferencias, siempre respetando los permisos de usuario. | UR-32 |
| FR-106 | El sistema deberá incluir un calendario laboral para la correcta organización de los diferentes eventos próximos. | UR-32 |

#### 4.1.10. Búsqueda, filtros y organización

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-107 | El sistema deberá permitir buscar y filtrar facturas a partir del nombre comercial, la razón social, el NIF, la persona de contacto, el número de factura o el correo. | UR-23 |
| FR-108 | El sistema deberá permitir guardar filtros frecuentes. | UR-23 |
| FR-109 | El sistema deberá permitir asignar etiquetas relacionadas con el sector, la ubicación o el cliente para conseguir una organización más flexible. | UR-23, UR-05 |
| FR-110 | El sistema deberá listar los clientes favoritos o recientes para un acceso más cómodo. | UR-23 |

#### 4.1.11. Informes, dashboard y exportaciones

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-111 | El sistema deberá generar informes semanales, mensuales y trimestrales de los cobros. | UR-26 |
| FR-112 | El sistema deberá permitir filtrar los informes por periodo, cliente, responsable interno, país, sector, estado de factura, tipo (recurrente o puntual) y, si tiene sentido, moneda. | UR-26 |
| FR-113 | El sistema deberá permitir exportar los resultados de los informes, al menos, a CSV. | UR-26 |
| FR-114 | El sistema deberá permitir obtener un informe legible en PDF. | UR-26 |
| FR-115 | La exportación deberá respetar los permisos del usuario. | UR-26 |
| FR-116 | El sistema deberá ofrecer a los administradores un dashboard financiero que muestre lo facturado en la semana, el mes, el trimestre y el año. | UR-27 |
| FR-117 | El dashboard financiero deberá mostrar lo cobrado, lo pendiente y lo vencido. | UR-27 |
| FR-118 | El dashboard financiero deberá mostrar el tiempo medio de cobro y la evolución. | UR-27 |
| FR-119 | El dashboard financiero deberá mostrar los clientes con mayor deuda. | UR-27 |
| FR-120 | El dashboard financiero deberá mostrar los ingresos recurrentes esperados del mes, la facturación puntual y una previsión sencilla basada en las recurrencias activas. | UR-27 |
| FR-121 | El sistema deberá mostrar en el dashboard de los administradores los objetivos de facturación mensuales o anuales y su progreso, a nivel informativo. | UR-27 |
| FR-122 | El sistema deberá permitir exportar un paquete mensual o trimestral para la gestoría con el listado de facturas emitidas, rectificativas, cobros y PDFs, o un ZIP organizado. | UR-28 |
| FR-123 | El sistema deberá registrar cuándo se generó el paquete para la gestoría y qué periodo cubre. | UR-28, UR-29 |
| FR-124 | Los archivos exportados deberán llevar fecha y hora, filtros usados y usuario que los generó, en metadatos o en el registro de auditoría. | UR-26, UR-29 |
| FR-125 | Las exportaciones masivas de datos sensibles deberán quedar auditadas. | UR-29 |
| FR-126 | Los archivos exportados podrán conservarse durante un tiempo configurable y regenerarse si los datos y los permisos lo permiten. | UR-26, UR-04 |

#### 4.1.12. Configuración y plantillas de comunicación

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-127 | El sistema deberá permitir configurar el plazo de pago por defecto, la cadencia de recordatorios, el dominio de correo permitido, los datos fiscales de Turbine, el logo y la plantilla, las series, las monedas habilitadas, los impuestos disponibles, los umbrales de aprobación, los canales de notificación y las políticas de automatización. | UR-04 |
| FR-128 | El sistema deberá validar y auditar los cambios de configuración. | UR-04, UR-29 |
| FR-129 | Los cambios de configuración no deberán afectar a las facturas anteriores. | UR-04 |
| FR-130 | Ante cambios de configuración delicados, el sistema deberá permitir simularlos antes de aplicarlos definitivamente. | UR-04 |
| FR-131 | El sistema deberá disponer de plantillas para los correos de factura, recordatorio previo, vencimiento, impago, disputa resuelta y revisión de datos, editables por quien tenga permiso. | UR-31 |
| FR-132 | Las plantillas deberán admitir variables como nombre de empresa, número de factura, importe, vencimiento, responsable y datos de soporte. | UR-31 |
| FR-133 | Si falta una variable, la previsualización de la plantilla deberá avisar. | UR-31 |
| FR-134 | El sistema deberá mantener un historial de versiones de las plantillas a medida que se actualicen. | UR-31 |

#### 4.1.13. Auditoría, registros y monitorización

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-135 | El sistema deberá mantener, a modo de auditoría, un registro de todos los cambios realizados en la aplicación junto con la persona que los realizó. | UR-29 |
| FR-136 | El sistema deberá mantener un registro de ejecución de los procesos automáticos, recordatorios, conciliaciones e informes programados, indicando, por ejemplo, que el generador mensual inició, cuántos clientes procesó, cuántas facturas creó y cuáles fallaron y por qué. | UR-30 |
| FR-137 | El administrador deberá poder configurar un resumen semanal o mensual que llegue a determinadas personas. | UR-30 |
| FR-138 | El sistema deberá ofrecer una pantalla de salud que muestre la última ejecución de generación recurrente, la cola de correos pendientes, los errores de integración recientes y el último backup correcto. | UR-30 |

#### 4.1.14. Bajas, reactivación y API

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-139 | El sistema deberá permitir registrar solicitudes de baja de clientes. | UR-19, UR-35 |
| FR-140 | El sistema deberá permitir consultar las deudas pendientes de un cliente. | UR-19, UR-35 |
| FR-141 | A través de la API, el sistema deberá permitir comprobar si un cliente tiene facturas pendientes. | UR-35 |
| FR-142 | Si el cliente tiene facturas pendientes, el sistema deberá dejar de generar nuevas mensualidades desde la fecha de baja, manteniendo vivo el flujo de recobro de las anteriores. | UR-19 |
| FR-143 | Si el cliente no tiene facturas pendientes, la parte de facturación deberá poder cerrar la recurrencia en la fecha efectiva de baja. | UR-19 |
| FR-144 | El sistema deberá permitir la baja de clientes solo cuando no tengan facturas pendientes, conservando sus datos según la ley (véase Q-04). | UR-19 |
| FR-145 | Si un cliente vuelve a contratar la suscripción, el sistema deberá permitir reactivar su ficha histórica siempre que sea posible. | UR-19, UR-05 |

#### 4.1.15. Experiencia de usuario y acciones masivas

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-146 | El sistema deberá permitir guardar un borrador antes de completar un formulario para su revisión. | UR-34 |
| FR-147 | El sistema deberá notificar los campos erróneos de un formulario. | UR-34 |
| FR-148 | El sistema deberá mostrar valores por defecto en cada campo (país habitual, moneda habitual, vencimiento por defecto y tarifa sugerida según el rango). | UR-34, UR-05 |
| FR-149 | El sistema deberá permitir realizar acciones masivas (por ejemplo, reasignar veinte clientes de un empleado que se marcha, exportar una selección, cambiar el sector de varias empresas o generar un informe). | UR-34, UR-03 |
| FR-150 | Antes de ejecutar una acción masiva, el sistema deberá mostrar a cuántos elementos afecta. | UR-34 |
| FR-151 | El sistema deberá pedir confirmación cuando una acción masiva sea destructiva o sensible. | UR-34 |
| FR-152 | Tras una acción masiva, el sistema deberá notificar el resultado con los éxitos y los errores. | UR-34 |

#### 4.1.16. Adjuntos y comentarios

| ID | Requisito | Relacionado con |
| --------- | ----------------------------------------------------------------------- | -------------------- |
| FR-153 | El sistema deberá permitir adjuntar documentos (por ejemplo, el pedido del cliente, el presupuesto aceptado, el justificante enviado por correo, una captura o un documento de la gestoría) además del PDF de la factura. | UR-25 |
| FR-154 | Todo adjunto deberá tener un tipo y una descripción. | UR-25 |
| FR-155 | El acceso a un adjunto deberá seguir los permisos del cliente o de la factura a los que esté asociado. | UR-25, UR-02 |
| FR-156 | El sistema deberá permitir comentar en una factura o en una incidencia, mencionar a otro usuario y generarle una notificación, siempre respetando los permisos de usuario. | UR-24, UR-21 |

### 4.2. Requisitos no funcionales

Los requisitos no funcionales describen propiedades de calidad, restricciones o condiciones que debe satisfacer el sistema.

| ID | Categoría | Requisito | Criterio verificable |
| -------- | -------------- | ------------------------------------ | ------------------------------------------ |
| NFR-01 | Identidad corporativa | El diseño de la aplicación web deberá ser coherente con el estilo de la empresa. | Las pantallas y los documentos emitidos utilizan el logo, la tipografía y la estética corporativa de Turbine; se comprueba mediante revisión visual con el cliente. |
| NFR-02 | Identidad corporativa | El PDF de las facturas deberá mantener la estética de la empresa (logo, tipografía razonable). | El PDF generado muestra el logo y la tipografía corporativos; un documento ya emitido se puede reproducir con la imagen con la que se emitió (se conserva el PDF original aunque cambie la plantilla). |
| NFR-03 | Portabilidad / Usabilidad | La aplicación web deberá ser compatible con todo tipo de dispositivos. | Las funcionalidades principales son utilizables desde ordenador, tableta y teléfono móvil (diseño responsive) en navegadores modernos actuales de escritorio, Android e iOS (véase A-08). |
| NFR-04 | Seguridad | La aplicación solo deberá ser accesible internamente desde la empresa. | Solo acceden usuarios internos autenticados con correo del dominio corporativo; no existe autorregistro ni acceso de clientes. La restricción adicional de red está pendiente de aclarar (véase Q-03). |
| NFR-05 | Usabilidad | La facturación deberá poder realizarse en menos de un minuto y con el mínimo de clics. | Un usuario que conoce el proceso administrativo pero no la aplicación completa en menos de un minuto una operación habitual (alta de un cliente estándar, factura normal o recurrente, comprobación de estado); el número de clics se mide en pruebas de usabilidad y se documenta (véase A-08). |
| NFR-06 | Rendimiento | La aplicación deberá ser ligera y rápida. | El tiempo de carga concreto está por definir (véase Q-02). Con carácter provisional, las pantallas y búsquedas habituales deben responder de forma prácticamente inmediata en una conexión normal, sin llegar a esperas del orden de diez segundos con miles de clientes. |
| NFR-07 | Escalabilidad | La aplicación deberá ser escalable para más de 1.000 clientes sin perder rendimiento. | Con al menos 1.000 clientes los tiempos de respuesta de las operaciones habituales se mantienen respecto a los de pocos clientes; el diseño no asume un número fijo de clientes (el objetivo del cliente son miles de clientes y cientos de miles de facturas históricas) y los listados se paginan en lugar de cargar todo el histórico. |
| NFR-08 | Cumplimiento normativo | La aplicación deberá cumplir la normativa fiscal española y el RGPD. | La gestoría valida que las facturas tienen el contenido legalmente exigible; es posible localizar los datos personales de un contacto y anonimizarlos o eliminarlos cuando proceda; los datos se conservan el tiempo que exige la ley. |
| NFR-09 | Mantenibilidad | La aplicación deberá ser simple, funcional y fácil de mantener. | Existe documentación técnica y de despliegue; la configuración se concentra en un único punto (por ejemplo, cambiar la cadencia de recordatorios no exige tocar varios lugares); las reglas de negocio están cubiertas por pruebas automáticas. |
| NFR-10 | Automatización | Todo proceso que sea general y frecuente deberá ser automático. | La generación de las facturas recurrentes y el envío de avisos y recordatorios se ejecutan sin intervención manual cuando no hay excepciones; las excepciones quedan pendientes de revisión (véase FR-096). |
| NFR-11 | Fiabilidad / Integridad | La interfaz deberá prevenir peticiones duplicadas mediante botones deshabilitados o claves de idempotencia, especialmente en operaciones financieras. | Un doble clic, una recarga o un reintento sobre una emisión, un envío o una conciliación no genera facturas, envíos ni cobros duplicados; el usuario recibe un mensaje de que la operación ya se está procesando. |
| NFR-12 | Rendimiento | Las operaciones masivas y las tareas pesadas deberán ejecutarse en segundo plano sin congelar la interfaz ni ralentizar el sistema. | Mientras una tarea pesada se ejecuta (por ejemplo, generar un informe anual o un gran número de PDFs) la interfaz permite seguir trabajando y muestra su estado (pendiente, procesando, completado con errores) hasta que esté lista. |
| NFR-13 | Fiabilidad / Integridad | El sistema deberá utilizar precisión decimal estricta (no tipos float simples) para la moneda. | Los totales cuadran al céntimo; el PDF muestra exactamente lo que se almacena y las sumas del dashboard se derivan de los mismos datos; la regla de redondeo está documentada. |
| NFR-14 | Usabilidad | Las acciones masivas deberán indicar cuántos elementos serán afectados y requerir confirmación explícita si son destructivas. | Toda acción masiva muestra el número de elementos afectados antes de ejecutarse y las destructivas no se ejecutan sin confirmación explícita. |
| NFR-15 | Accesibilidad | El sistema deberá ser accesible, permitiendo el uso completo mediante teclado, ofreciendo un contraste adecuado y no dependiendo exclusivamente del puntero del ratón. | Todas las funciones se pueden operar solo con teclado; ningún control depende de pasar el ratón por encima; el contraste cumple el nivel AA de las pautas WCAG (véase A-08); las etiquetas y mensajes de error son comprensibles. |
| NFR-16 | Seguridad | El sistema deberá expirar automáticamente las sesiones tras un periodo de inactividad e invalidar activamente las sesiones si un usuario es desactivado por el administrador. | Tras el periodo de inactividad configurado (valor por definir, véase Q-09) la sesión se cierra; al desactivar un usuario, su siguiente petición se rechaza. |
| NFR-17 | Seguridad | Los archivos adjuntos y los documentos legales (PDFs) no deberán ser accesibles mediante URLs públicas adivinables, requiriendo enlaces temporales o autenticación. | Sin sesión válida o sin permiso sobre el cliente o la factura, el acceso a un documento por su URL es rechazado; los enlaces temporales caducan. |
| NFR-18 | Fiabilidad / Integridad | El sistema deberá garantizar concurrencia y coherencia al modificar facturas de forma simultánea. | Si dos usuarios modifican a la vez la misma factura, el sistema detecta el conflicto o bloquea la edición, sin sobrescribir silenciosamente los cambios; la numeración y la conciliación son consistentes aunque dos personas actúen casi a la vez. |
| NFR-19 | Recuperabilidad | El sistema deberá realizar y permitir restaurar copias de seguridad automáticas de datos y documentos según la política definida. | Se generan copias automáticas periódicas de datos y documentos; la restauración se ha probado y se puede demostrar; el último backup correcto es visible en la pantalla de salud. |
| NFR-20 | Seguridad | Las comunicaciones y las contraseñas deberán estar cifradas para mantener la seguridad de los datos sensibles. | Todo el tráfico se realiza por un canal cifrado; las contraseñas no se almacenan nunca en texto plano; no hay secretos en el repositorio. |
| NFR-21 | Rendimiento | En situaciones generales, las tareas normales deberán poder realizarse de forma fluida en menos de un minuto y con pocos clics. | Las tareas habituales se completan en menos de un minuto y con pocos clics; la búsqueda o la apertura de un cliente o factura es prácticamente inmediata; las tareas más pesadas se ejecutan en segundo plano (véase NFR-12). |
| NFR-22 | Escalabilidad | El uso de la aplicación web deberá estar preparado para el acceso cómodo de cientos de personas. | El modelo y la arquitectura permiten pasar de pocos usuarios internos a cientos sin rehacerse (la cifra de usuarios concurrentes está pendiente de concretar, véase Q-09). |
| NFR-23 | Disponibilidad | Las tareas automáticas mensuales deberán ser reintentables; ante fallos, el sistema deberá detectar qué facturas no se generaron y recuperarse sin generarlas dos veces. | Si el sistema está caído a las 00:00 del día 1, al volver detecta las facturas no generadas y las crea sin duplicar las ya existentes; el fallo de una tarea concreta no impide procesar las demás. |
| NFR-24 | Internacionalización | El sistema deberá estar disponible en español e inglés (preferiblemente en las plantillas; para la interfaz el inglés puede ser opcional). | Las plantillas principales de comunicación existen en español e inglés y se puede elegir el idioma por cliente o contacto; la interfaz está en español y su diseño permite añadir otros idiomas. |
| NFR-25 | Internacionalización / Fiabilidad | El sistema deberá mantener la consistencia de fechas y zonas horarias ante eventos tanto dentro como fuera de España. | Los instantes se almacenan de forma consistente y se muestran en la zona horaria adecuada; para la fecha legal de una factura se usa la fecha de negocio correspondiente y no la hora del servidor. |
| NFR-26 | Usabilidad / Portabilidad | La interfaz móvil deberá ser cómoda, accesible y contar con las funciones esenciales. | Desde un móvil se pueden consultar clientes, aprobar algo urgente, ver impagos, marcar una incidencia y revisar un pago, con controles utilizables sin depender del ratón. |
| NFR-27 | Seguridad | Los adjuntos deberán limitarse a formatos y tamaños razonables y no deberá ejecutarse ningún archivo peligroso. | Solo se aceptan los formatos y tamaños permitidos por la configuración; los archivos con formato no permitido son rechazados y ningún adjunto se ejecuta en el servidor. |
| NFR-28 | Mantenibilidad / Documentación | El contrato de la API deberá estar documentado para el correcto uso de sus funciones. | Existe una documentación estructurada de la API (recursos, parámetros, respuestas, permisos y errores) suficiente para que otro sistema la use sin consultar el código. |
| NFR-29 | Mantenibilidad | Las integraciones (como CSV o Stripe) deberán estar encapsuladas para poder reemplazarlas de forma cómoda y sin afectar al resto del sistema. | Sustituir el proveedor de una integración solo exige modificar el módulo de integración, sin cambiar el modelo de facturas ni el resto de funcionalidades. |
| NFR-30 | Observabilidad / Mantenibilidad | El tratamiento de errores deberá ser distinto para el desarrollador y para el administrador: el desarrollador recibe el error con identificador y stack trace en el servidor; el administrador lo visualiza a través de un aviso. | Cada error técnico genera un identificador y un registro en el servidor con su traza; el usuario final solo ve un aviso comprensible con ese código y nunca información técnica sensible. |
| NFR-31 | Seguridad | El sistema deberá ofrecer protección básica ante intentos repetidos de inicio de sesión fallidos (ataque de fuerza bruta). | Tras el número de intentos fallidos definido (véase Q-09) el acceso se limita de forma temporal, sin convertir la aplicación interna en un sistema de bloqueos permanentes. |
| NFR-32 | Calidad / Mantenibilidad | Las reglas económicas importantes, como el prorrateo o el cálculo de impuestos, deberán contar con pruebas automáticas. | Existen pruebas automáticas ejecutables que cubren, al menos, el prorrateo, el cálculo de impuestos, los vencimientos, la generación mensual sin duplicados y los permisos. |
| NFR-33 | Mantenibilidad | El sistema deberá permitir el uso de datos de prueba para testeo sin mezclarlos con los datos de producción. | Se puede cargar un conjunto de datos ficticios para pruebas y demostraciones sin que se mezcle con producción. |
| NFR-34 | Usabilidad | La interfaz deberá ser consistente y cómoda de navegar. | Las acciones equivalentes se ubican en el mismo lugar en todas las pantallas y los estados se muestran con etiquetas y textos coherentes. |
| NFR-35 | Observabilidad | El sistema deberá registrar métricas de rendimiento para comprobar su correcto funcionamiento. | Se registran métricas agregadas (tiempos de generación, fallos) consultables por el equipo técnico, sin utilizarlas para vigilar a las personas (véase A-08). |

## 5. Reglas de negocio

Reglas del dominio o de la organización que condicionan el comportamiento del sistema pero que no representan por sí mismas una funcionalidad.

| ID | Regla |
| -------- | -------------------------------------------------------------------------------------------- |
| RB-01 | El plazo de pago por defecto de una factura es de cinco días; el superadministrador puede cambiarlo. Un cliente puede tener un plazo pactado distinto (por ejemplo, treinta días) que prevalece sobre el valor general. *(Fuente: Cliente / entrevista (1.1.4, 3.1.16))* |
| RB-02 | Las mensualidades se facturan a partir del día 1 de cada mes. Si el alta es otro día, se factura la parte proporcional hasta final de mes y desde el día 1 siguiente la mensualidad completa; el no cobro del prorrateo es una excepción que debe quedar explícita. *(Fuente: Cliente / entrevista (2.1.6))* |
| RB-03 | Las facturas recurrentes son mensuales y de importe constante durante el año; no existe pago fraccionado como modalidad comercial. *(Fuente: Cliente / entrevista (aclaración))* |
| RB-04 | El precio de la suscripción depende del rango de facturación anual del cliente; el rango se revisa una vez al año y un cambio de rango debe ser revisado y aprobado por un responsable. El nuevo precio no reescribe las facturas de meses anteriores. *(Fuente: Cliente / entrevista (2.1.2, 3.1.25))* |
| RB-05 | Una suscripción pausada no genera mensualidades, pero el cliente sigue existiendo y puede tener facturas antiguas pendientes. *(Fuente: Cliente / entrevista (2.1.3))* |
| RB-06 | Una factura puede corregirse antes de enviarse; una vez enviada, cualquier cambio relevante debe ser trazable y, si no puede modificarse, se corrige mediante una factura rectificativa vinculada a la original. *(Fuente: Cliente / entrevista (2.1.4, 3.1.24))* |
| RB-07 | Cada factura tiene un identificador único (serie y número) que se asigna al emitirla. *(Fuente: Cliente / entrevista (2.1.5))* |
| RB-08 | En un trimestre o periodo cerrado no se permiten cambios ordinarios en las facturas; solo mediante un procedimiento excepcional y auditado. *(Fuente: Cliente / entrevista (1.1.8, 3.1.35))* |
| RB-09 | Cuando un cliente se da de baja dejan de generarse nuevas mensualidades desde la fecha de baja, pero el recobro de las facturas anteriores pendientes se mantiene; los datos del cliente se conservan según lo que permita la ley. *(Fuente: Cliente / entrevista (1.1.13, 3.1.1))* |
| RB-10 | Una factura solo se considera pagada cuando se ha recibido el importe total; si el importe recibido es distinto (faltante o sobrante) se notifica y se revisa, y queda registrada la conclusión. *(Fuente: Cliente / entrevista (1.1.10, 3.1.12))* |
| RB-11 | Los procesos generales y frecuentes son automáticos; las excepciones (cambios no habituales, descuentos) se revisan o aprueban antes de enviar la facturación. Cancelar o considerar incobrable una deuda exige motivo y aprobación. *(Fuente: Cliente / entrevista (2.2.1, 3.1.11))* |
| RB-12 | El usuario operativo solo actúa sobre los clientes que tiene asignados y no puede darse acceso a otros, cambiar tarifas globales, crear series ni ver el dashboard financiero global. El administrador gestiona asignaciones y aprobaciones. *(Fuente: Cliente / entrevista (1.1.5, 1.1.6))* |
| RB-13 | Los cambios de configuración y los cambios en los conceptos frecuentes no afectan a las facturas ya emitidas o creadas. *(Fuente: Cliente / entrevista (3.1.4, 3.1.16))* |
| RB-14 | Solo pueden acceder usuarios internos con correo del dominio corporativo; los clientes de Turbine no se registran ni acceden a la aplicación. *(Fuente: Cliente / entrevista (1.1.1, 1.2.3))* |
| RB-15 | No se fusionan empresas de forma automática; ante un posible duplicado por identificador fiscal solo se avisa. *(Fuente: Cliente / entrevista (3.1.3))* |
| RB-16 | Las notificaciones solo informativas pueden marcarse como leídas; las que requieren acción permanecen pendientes hasta que se atiende lo notificado. *(Fuente: Cliente / entrevista (3.1.7))* |
| RB-17 | El sistema no actúa como pasarela de pago: solo comprueba y registra los cobros reflejados en Stripe (simulado). *(Fuente: Cliente / entrevista (1.1.7))* |

## 6. Suposiciones y decisiones de análisis

Decisiones tomadas por el equipo cuando la información proporcionada por el cliente no era suficiente para determinar unívocamente un requisito.

| ID | Suposición o decisión | Justificación | Requisitos afectados |
| ------- | ------------------------------------------ | ---------------------------- | ----------------------- |
| A-01 | Cada requisito del documento original se ha reformulado y, cuando era compuesto, descompuesto en requisitos atómicos («El sistema deberá…»). No se ha eliminado ninguna información. | La plantilla exige requisitos claros, precisos y verificables; el Anexo A permite comprobar a qué requisitos corresponde cada requisito original. | Todos |
| A-02 | El PDF de la factura y los adjuntos, clasificados como no funcionales en el documento original, se han dividido: el comportamiento (generar el PDF y su contenido; adjuntar documentos con tipo y descripción; acceso por permisos) pasa a requisitos funcionales, y las propiedades de calidad (estética corporativa, formatos y tamaños permitidos) permanecen como no funcionales. Lo mismo se ha hecho con la revisión de excepciones del requisito de automatización. | Según la plantilla, los funcionales describen comportamientos y los no funcionales propiedades de calidad o restricciones. | FR-066, FR-067, NFR-02, FR-153, FR-154, FR-155, NFR-27, FR-096, NFR-10 |
| A-03 | Se utiliza el término «usuario operativo» para el rol que en el documento original aparece como «usuario» y, en las entrevistas, como «lector». Existen tres roles: superadministrador, administrador y usuario operativo. | El cliente aclaró que lo importante es la capacidad (trabajar sobre lo asignado, con permiso de edición) y no el nombre. | FR-007, FR-009, FR-010 |
| A-04 | Se asume que cada cliente tiene un responsable interno (administrador o usuario operativo) que recibe sus notificaciones y es referencia en informes y filtros. | Los requisitos hablan de «responsable interno» y de «responsable encargado de la facturación del cliente», pero no definen cómo se asigna; se deduce de los requisitos y de las entrevistas. | FR-027, FR-099 |
| A-05 | Se entiende por factura recurrente la mensualidad de una suscripción activa, generada automáticamente el día 1 de cada mes (y la parte proporcional en el alta). | El documento menciona facturas recurrentes y mensualidades sin enumerar explícitamente la generación automática; se deduce del requisito de automatización y del prorrateo. | FR-049, FR-045, FR-046 |
| A-06 | Los dos avisos a los 3 y a los 5 días se interpretan como recordatorios de pago contados desde la emisión de la factura, siendo cinco días el plazo de pago por defecto; el segundo coincide con el vencimiento. | El requisito dice «incumplimiento del pago» con avisos a los 3 y 5 días; en la entrevista se explicó que el pago es a cinco días y que se recuerda el día 3 y el día 5. | FR-087, FR-088, FR-089 |
| A-07 | Tras el vencimiento se asume un flujo de impago con recordatorios periódicos (por defecto cada dos o tres días) hasta que se detecte el pago o una persona intervenga; la cadencia es configurable. | Aunque no figura de forma literal en el documento de requisitos, el cliente lo describió en la entrevista y es necesario para el seguimiento del recobro que se menciona en el requisito de baja. | FR-090 |
| A-08 | Los criterios verificables de los requisitos no funcionales que el cliente no cuantificó son propuestas provisionales del equipo, tomadas de lo dicho en las entrevistas: tiempo de operación habitual inferior a un minuto; accesibilidad conforme al nivel AA de WCAG; navegadores actuales de escritorio, Android e iOS; métricas agregadas de rendimiento sin vigilancia de personas. | La plantilla pide criterios verificables o, en su defecto, documentar la interpretación realizada. | NFR-03, NFR-05, NFR-15, NFR-35 |
| A-09 | Se interpreta «accesible solo internamente» como acceso restringido a usuarios internos autenticados con correo corporativo, compatible con el uso desde cualquier sitio y dispositivo; si además se requiere restricción de red, se decidirá con el cliente. | Existe una tensión aparente entre este requisito y el de uso desde cualquier dispositivo y desde el móvil (véase Q-03). | NFR-04, NFR-03, NFR-26 |
| A-10 | Se considera que un trimestre cerrado es un caso particular de periodo administrativo cerrado. | El documento menciona ambos conceptos con el mismo efecto (impedir cambios ordinarios) (véase Q-08). | FR-070, FR-071, FR-072 |
| A-11 | Las integraciones con Stripe, correo y WhatsApp se tratan como servicios simulados y encapsulados, de forma que pueda sustituirse la simulación por el proveedor real sin cambiar el modelo de facturas. | El cliente pidió explícitamente una integración simulada con Stripe y encapsular las integraciones. | FR-080, FR-073, NFR-29 |
| A-12 | Los datos de los clientes dados de baja no se destruyen: se conservan según lo que permita la ley (borrado lógico) y se puede reactivar su ficha histórica. | Requisito de baja con conservación de datos según la ley y reactivación de la ficha. | FR-144, FR-145 |
| A-13 | Las aprobaciones las resuelve un administrador y, cuando el importe o el descuento supere un umbral configurable, el superadministrador; los umbrales son parámetros de configuración. | El documento habla de aprobación «dependiendo del importe» y de «umbrales de aprobación» configurables sin fijar valores. | FR-097, FR-058, FR-127 |
| A-14 | El estado de una factura es, como mínimo, pendiente, pagada o impagada (vencida sin pago); el estado del cliente refleja su situación frente a la facturación (por ejemplo, pendiente de activación, activo, pausado, baja solicitada, inactivo). | El documento enumera los estados de factura y pide representar el estado del cliente sin listarlos; se propone un conjunto mínimo. | FR-079, FR-025 |

## 7. Dudas, ambigüedades y cuestiones pendientes

Aspectos de la información proporcionada por el cliente que son ambiguos, incompletos, contradictorios o que requieren aclaración.

| ID | Cuestión | Origen | Impacto | Estado / resolución |
| ------- | ------------------------------------ | ----------------- | ------------------ | ---------------------- |
| Q-01 | El correo de acceso debe ser de «TurbineH»: ¿es un error y el dominio es el de Turbine, o es un dominio concreto? | Documento de requisitos (1.1.1) | Validación del correo de acceso y del dominio configurable. | Pendiente (se asume el dominio corporativo de Turbine) |
| Q-02 | ¿Cuál es el tiempo de carga concreto que se considera «ligera y rápida»? | Documento de requisitos (1.2.5) | Criterio verificable de rendimiento. | Pendiente (criterio provisional en el requisito) |
| Q-03 | «Solo accesible internamente desde la empresa» frente al uso desde cualquier sitio y desde el móvil: ¿se exige además restricción de red (por ejemplo, VPN) o basta la autenticación corporativa? | Documento de requisitos (1.2.3, 1.2.2, 3.2.10); entrevista | Seguridad, despliegue y usabilidad móvil. | Pendiente (se asume autenticación corporativa) |
| Q-04 | Se indica que solo se puede dar de baja a un cliente sin facturas pendientes, pero también que se registra la baja y se mantiene el recobro de las deudas anteriores: ¿la baja efectiva se bloquea o solo se dejan de generar mensualidades? | Documento de requisitos (1.1.13 frente a 3.1.1) | Flujo de baja y estado del cliente. | Pendiente (propuesta: se registra la solicitud y se dejan de generar mensualidades; el recobro continúa) |
| Q-05 | ¿Los importes absurdos bloquean siempre la emisión de la factura o solo generan una advertencia que el usuario puede confirmar (por ejemplo, un proyecto puntual caro)? | Documento de requisitos (3.1.5); entrevista | Regla de validación en la emisión. | Pendiente |
| Q-06 | ¿Quién configura las series de facturación, el administrador o el superadministrador? | Documento de requisitos (2.1.5); entrevista | Permisos de configuración. | Pendiente (se sigue el documento: administrador) |
| Q-07 | ¿El «rango de cobro» (de anual a mensual, etc.) se refiere al tramo de tarifa según la facturación anual del cliente o a la periodicidad de cobro? | Documento de requisitos (2.1.2) | Modelo de suscripción y tarifas. | Pendiente (se asume tramo de tarifa; la periodicidad es mensual) |
| Q-08 | ¿El cierre de trimestre es automático por fecha o lo marca manualmente un administrador? | Documento de requisitos (1.1.8 frente a 3.1.35) | Control de edición de facturas. | Pendiente (se asume cierre manual) |
| Q-09 | Valores no definidos: periodo de inactividad de la sesión, número de intentos fallidos de acceso, usuarios concurrentes esperados, umbrales de aprobación y plazos de conservación de exportaciones y copias de seguridad. | Documento de requisitos (2.2.7, 3.2.6, 3.2.15, 3.1.16, 3.1.39, 3.2.3) | Parámetros de configuración y criterios de los requisitos no funcionales. | Pendiente (parámetros configurables) |
| Q-10 | ¿Cuál es el conjunto completo de estados de cliente y de factura que debe manejar el sistema? | Documento de requisitos (1.1.10, 3.1.33) | Modelo de datos y reglas de transición. | Pendiente (propuesta en A-14) |
| Q-11 | ¿Cómo se elimina o anonimiza a un cliente en las copias de seguridad? | Entrevista | Cumplimiento del RGPD y política de copias. | Pendiente (el cliente indicó que no lo tiene pensado) |
| Q-12 | El cliente indicó que no hay pagos parciales como modalidad pero se pide notificar faltantes y sobrantes: ¿cómo debe tratarse un cobro de importe distinto? | Documento de requisitos (3.1.12); entrevista | Regla de conciliación. | Resuelta: no hay pago fraccionado comercial, pero el sistema detecta, notifica y revisa las diferencias sin marcar la factura como pagada |
| Q-13 | El rol «lector» ¿puede editar o solo consultar? | Entrevista (primera sesión) | Permisos del rol operativo. | Resuelta: puede ver y modificar lo asignado; se denomina «usuario operativo» (A-03) |

## 8. Trazabilidad

La tabla permite comprobar la relación entre las necesidades de negocio, los requisitos de usuario y los requisitos del sistema.

| Requisito de negocio | Requisito de usuario | Requisito funcional / no funcional |
| ------------------ | ------------------ | ---------------------------------------------------------------- |
| BR-10 | UR-01 | FR-001, FR-002, FR-006, NFR-04, NFR-16, NFR-20, NFR-31 |
| BR-10 | UR-02 | FR-003–FR-005, FR-007, FR-008, FR-014, FR-155, NFR-04, NFR-16, NFR-22 |
| BR-10, BR-04 | UR-03 | FR-009, FR-014–FR-017, FR-149 |
| BR-06, BR-11, BR-05 | UR-04 | FR-002, FR-023, FR-060, FR-088, FR-126–FR-130 |
| BR-02, BR-04 | UR-05 | FR-010, FR-018–FR-027, FR-109, FR-145, FR-148, NFR-05 |
| BR-02, BR-01 | UR-06 | FR-028–FR-034 |
| BR-06, BR-01 | UR-07 | FR-035–FR-037, FR-041–FR-048 |
| BR-06 | UR-08 | FR-036, FR-038–FR-040 |
| BR-02, BR-01 | UR-09 | FR-011, FR-050–FR-053, FR-068, FR-069 |
| BR-09, BR-13, BR-05 | UR-10 | FR-059–FR-067, NFR-01, NFR-02, NFR-05, NFR-08, NFR-11, NFR-13, NFR-17, NFR-18, NFR-25 |
| BR-09, BR-05 | UR-11 | FR-054–FR-058, FR-070, NFR-18 |
| BR-01 | UR-12 | FR-042, FR-045, FR-046, FR-049, FR-073, NFR-10, NFR-23 |
| BR-06, BR-05 | UR-13 | FR-096, FR-097, NFR-10 |
| BR-06, BR-04 | UR-14 | FR-040, FR-058, FR-086, FR-097 |
| BR-12, BR-08 | UR-15 | FR-012, FR-043, FR-079, FR-081, FR-083–FR-085, NFR-11, NFR-25 |
| BR-01, BR-12 | UR-16 | FR-080–FR-082, NFR-29 |
| BR-12, BR-01 | UR-17 | FR-087–FR-091, FR-094 |
| BR-12, BR-06 | UR-18 | FR-086, FR-090 |
| BR-12, BR-09 | UR-19 | FR-139, FR-140, FR-142–FR-145, NFR-08 |
| BR-04, BR-05 | UR-20 | FR-024, FR-073–FR-078 |
| BR-04 | UR-21 | FR-027, FR-044, FR-075, FR-084, FR-098–FR-102, FR-156 |
| BR-02, BR-04 | UR-22 | FR-103, FR-104, NFR-05 |
| BR-02 | UR-23 | FR-107–FR-110, NFR-06, NFR-07, NFR-21 |
| BR-04, BR-05 | UR-24 | FR-092–FR-095, FR-156 |
| BR-04, BR-10 | UR-25 | FR-013, FR-153–FR-155, NFR-17, NFR-27 |
| BR-08 | UR-26 | FR-111–FR-115, FR-124, FR-126, NFR-12 |
| BR-08 | UR-27 | FR-116–FR-121, NFR-07, NFR-13 |
| BR-09, BR-05 | UR-28 | FR-070–FR-072, FR-122, FR-123, NFR-08 |
| BR-05, BR-10, BR-09 | UR-29 | FR-017, FR-038, FR-055, FR-072, FR-077, FR-085, FR-123–FR-125, FR-128, FR-135 |
| BR-05 | UR-30 | FR-136–FR-138, NFR-19, NFR-23, NFR-30, NFR-35 |
| BR-13, BR-04 | UR-31 | FR-131–FR-134, NFR-01, NFR-24 |
| BR-04, BR-01 | UR-32 | FR-105, FR-106 |
| BR-03 | UR-33 | NFR-03, NFR-15, NFR-26 |
| BR-02, BR-03 | UR-34 | FR-146–FR-152, NFR-03, NFR-06, NFR-12, NFR-14, NFR-15, NFR-21, NFR-22, NFR-24, NFR-34 |
| BR-11, BR-12 | UR-35 | FR-139–FR-141, NFR-28 |
| BR-11, BR-05 | UR-36 | NFR-09, NFR-19, NFR-28–NFR-30, NFR-32, NFR-33, NFR-35 |
| BR-07 | UR-37 | NFR-07, NFR-22 |

## 9. Glosario

| Término | Definición |
| ------------------------ | ---------------------------------------------------------------------------- |
| Brain | Producto de inteligencia artificial de Turbine. El sistema que se describe es independiente de él. |
| Superadministrador | Rol con acceso prácticamente total y capacidad de configuración profunda. |
| Administrador | Rol que gestiona operaciones, clientes, facturas, asignaciones y aprobaciones según permisos. |
| Usuario operativo | Rol que trabaja sobre los clientes que tiene asignados. |
| Responsable interno | Usuario encargado de la facturación de un cliente concreto, destinatario de sus notificaciones. |
| Cliente | Empresa a la que Turbine presta sus servicios y factura; tiene una o varias personas de contacto. |
| Factura recurrente | Factura que se genera automáticamente todos los meses a partir de una suscripción. |
| Factura puntual | Factura de un servicio adicional o único (instalación, consultoría…) que no se repite. |
| Mensualidad | Importe mensual de la suscripción de un cliente, facturado desde el día 1 de cada mes. |
| Prorrateo | Parte proporcional de la mensualidad que se factura cuando el cliente se da de alta en un día distinto del 1. |
| Rango de cobro | Tramo de tarifa aplicable a un cliente según su facturación anual. |
| Suscripción | Relación comercial mensual con un cliente: plan, tarifa, condiciones de pago y fechas. |
| Emisión | Acción de dar por definitiva una factura, momento en que se le asigna su serie y número. |
| Serie | Conjunto de numeración correlativa de facturas. |
| Factura rectificativa | Documento vinculado a una factura emitida que corrige sus errores o devuelve parte de su importe. |
| Presupuesto | Documento previo a una factura puntual grande que, una vez aceptado, puede convertirse en factura. |
| Conciliación | Asociación de un cobro recibido con la factura que paga. |
| Impagada / vencida | Factura no pagada una vez pasado su vencimiento. |
| Incobrable | Deuda que, mediante decisión administrativa controlada, se considera no recuperable. |
| Incidencia de facturación | Seguimiento interno de un problema asociado a un cliente y, opcionalmente, a una factura. |
| Periodo cerrado | Mes o trimestre marcado como cerrado, en el que no se permiten cambios ordinarios en el histórico. |
| Auditoría | Registro de los cambios realizados en la aplicación y de quién los realizó. |
| Idempotencia | Propiedad por la que repetir una operación (doble clic, reintento) no produce efectos duplicados. |
| Gestoría | Asesoría externa que marca las reglas legales y recibe los paquetes de facturas y cobros. |
| Stripe | Servicio de pagos externo; en esta fase se utiliza una integración simulada. |
| API | Interfaz documentada que permite a otros sistemas consultar la aplicación de forma controlada. |
| NIF | Identificador fiscal de una empresa o persona. |
| RGPD | Reglamento General de Protección de Datos. |
| CSV | Formato de fichero de texto con valores separados por comas utilizado para importar y exportar datos. |
| Dashboard | Pantalla de indicadores financieros para administradores. |

## Anexo A. Correspondencia con el documento de requisitos original

Cada requisito del documento original («Requisitos funcionales y no funcionales») figura aquí con su numeración original y los requisitos de este documento que lo recogen. Sirve para comprobar que no se ha omitido ninguna información.

| N.º original | Requisito original (extracto) | Recogido en |
| ------------ | -------------------------------------------------- | -------------------------------------- |
| 1.1.1 (F) | Inicio de sesión con correo y contraseña (obligatorio correo de TurbineH). El… | FR-001–FR-006 |
| 1.1.2 (F) | Registro de datos del cliente para la facturación (nombre, NIF, correo, teléfono…). | FR-018 |
| 1.1.3 (F) | Importación de clientes desde un CSV con una previsualización antes de confirmar. El… | FR-028–FR-034 |
| 1.1.4 (F) | Envío de correos informando del incumplimiento del pago, siendo el primer aviso a los 3… | FR-087–FR-090 |
| 1.1.5 (F) | Asignación de roles y permisos: El superadministrador con prácticamente acceso total a la… | FR-007, FR-008 |
| 1.1.6 (F) | Un usuario operativo puede tener asignados clientes concretos. Puede ver y modificar la… | FR-009–FR-017 |
| 1.1.7 (F) | Con Stripe, ofrecer una integración simulada que se comporte como si recibiera eventos de… | FR-080–FR-082 |
| 1.1.8 (F) | Bloqueo de modificación de facturas cuando el trimestre está cerrado. | FR-070 |
| 1.1.9 (F) | Informe semanal, mensual y trimestral de los cobros. Los informes deben poder filtrarse por… | FR-027, FR-111–FR-115 |
| 1.1.10 (F) | Registro de cobros y estado de cada factura (pendiente, pagada, impagada). | FR-079 |
| 1.1.11 (F) | Además de facturas recurrentes, se podrán registrar facturas puntuales, cada una con… | FR-049–FR-051 |
| 1.1.12 (F) | Búsqueda y filtrado de facturas a partir del nombre comercial, razón social, NIF, persona… | FR-107, FR-108 |
| 1.1.13 (F) | Baja de clientes solo cuando no tengan facturas pendientes, conservando sus datos según la ley. | FR-144 |
| 1.2.1 (NF) | Diseño de la aplicación web coherente con el estilo de la empresa. | NFR-01 |
| 1.2.2 (NF) | Aplicación web compatible con todo tipo de dispositivos. | NFR-03 |
| 1.2.3 (NF) | Aplicación solo accesible internamente desde la empresa. | NFR-04 |
| 1.2.4 (NF) | Facturación en menos de un minuto y con el mínimo de clics. | NFR-05 |
| 1.2.5 (NF) | Aplicación ligera y rápida (falta definir un tiempo de carga concreto). | NFR-06 |
| 1.2.6 (NF) | Escalable para más de 1.000 clientes sin perder rendimiento. | NFR-07 |
| 1.2.7 (NF) | Cumplimiento de la normativa fiscal española y del RGPD. | NFR-08 |
| 1.2.8 (NF) | Aplicación simple, funcional y fácil de mantener. | NFR-09 |
| 2.1.1 (F) | A la hora de registrar el NIF, especificar el país y la moneda. El impuesto establecido… | FR-022, FR-023 |
| 2.1.2 (F) | Rango de cobro del cliente configurable (de anual a mensual, de mensual a anual, etc.), y… | FR-035–FR-038 |
| 2.1.3 (F) | Se debe poder pausar temporalmente una suscripción, registrando el motivo, fecha de inicio… | FR-041–FR-044 |
| 2.1.4 (F) | Antes de enviar una factura, esta debe poder corregirse. Y una vez enviada, cualquier… | FR-054–FR-056 |
| 2.1.5 (F) | Cada factura debe contar con su identificador único (número y serie). Y la serie es… | FR-059–FR-061 |
| 2.1.6 (F) | Las mensualidades se tienen en cuenta desde el día 1 de cada mes. Si un cliente entra en… | FR-045–FR-049 |
| 2.2.1 (NF) | Todo proceso que sea general y frecuente debe ser automático, mientras que las excepciones… | FR-049, FR-096, NFR-10 |
| 2.2.2 (NF) | La interfaz debe prevenir peticiones duplicadas mediante botones deshabilitados o claves de… | NFR-11 |
| 2.2.3 (NF) | Ejecución en segundo plano para operaciones masivas sin congelar la interfaz. | NFR-12 |
| 2.2.4 (NF) | Uso de precisión decimal estricta (no tipos float simples) para moneda. | NFR-13 |
| 2.2.5 (NF) | Las acciones masivas deben indicar cuántos elementos serán afectados y requerir una… | NFR-14 |
| 2.2.6 (NF) | El sistema debe ser accesible, permitiendo el uso completo mediante teclado, ofreciendo un… | NFR-15 |
| 2.2.7 (NF) | El sistema debe expirar automáticamente las sesiones tras un periodo de inactividad e… | NFR-16 |
| 2.2.8 (NF) | Los archivos adjuntos y documentos legales (PDFs) no deben ser accesibles mediante URLs… | NFR-17 |
| 3.1.1 (F) | La aplicación debe poder registrar solicitudes de baja y consultar deudas pendientes. A… | FR-090, FR-139–FR-143 |
| 3.1.2 (F) | En caso de que un cliente vuelva a contratar la suscripción, es preferible reactivar su… | FR-145 |
| 3.1.3 (F) | El sistema debería detectar posibles duplicados por identificador fiscal y avisar. No… | FR-020, FR-021 |
| 3.1.4 (F) | Posibilidad de buscar conceptos frecuentes a la hora de contratar un servicio, con un… | FR-052, FR-053 |
| 3.1.5 (F) | Generar vista previa antes de emitir una factura, y bloquear su emisión en caso de que… | FR-062–FR-065 |
| 3.1.6 (F) | Implementación de una bandeja de notificaciones: Al responsable encargado de la facturación… | FR-027, FR-098–FR-100 |
| 3.1.7 (F) | Las notificaciones podrán marcarse como leídas en caso de que sean únicamente informativas… | FR-101, FR-102 |
| 3.1.8 (F) | Al entrar a la aplicación debe mostrarse una pantalla de trabajo diario con facturas que… | FR-103, FR-104 |
| 3.1.9 (F) | El envío de una factura debe ser automático y por defecto por correo electrónico, pudiendo… | FR-073–FR-077 |
| 3.1.10 (F) | Los recordatorios de las facturas deben poder suspenderse temporalmente. | FR-091 |
| 3.1.11 (F) | Puede existir una decisión administrativa de cancelar o considerar incobrable una deuda,… | FR-086 |
| 3.1.12 (F) | A la hora de recibir el importe de una factura, debe comprobarse que se ha recibido el… | FR-083–FR-085 |
| 3.1.13 (F) | El dashboard financiero para administradores debe representar números: facturado esta… | FR-116–FR-120 |
| 3.1.14 (F) | Mantener un registro de todos los cambios realizados en la aplicación junto con la persona… | FR-135 |
| 3.1.15 (F) | Para procesos automáticos, recordatorios, conciliaciones e informes programados, debe haber… | FR-136, FR-137 |
| 3.1.16 (F) | Configuración: Los cambios de configuración deben validarse, auditarse y que no afecten a… | FR-002, FR-127–FR-130 |
| 3.1.17 (F) | Plantillas de comunicación: Los emails de factura, recordatorio previo, vencimiento,… | FR-131–FR-134 |
| 3.1.18 (F) | Experiencia de usuario: Guardar un borrador antes de completar un formulario para su… | FR-146–FR-148 |
| 3.1.19 (F) | Posibilidad de realizar acciones masivas mostrando a cuántos elementos afecta, pedir… | FR-149–FR-152 |
| 3.1.20 (F) | Asignación de etiquetas relacionadas al sector, ubicación o cliente, para conseguir una… | FR-109 |
| 3.1.21 (F) | Incidencias de facturación: Posibilidad de abrir incidencias asociadas a un cliente y… | FR-092–FR-095 |
| 3.1.22 (F) | Comunicación entre empleados: Un comentario en una factura o incidencia puede mencionar a… | FR-156 |
| 3.1.23 (F) | Posibilidad de crear presupuestos antes de facturas puntuales grandes para posteriormente… | FR-068, FR-069 |
| 3.1.24 (F) | Notas de crédito o rectificativas: Generación de documentos a partir de una factura, y… | FR-057, FR-058 |
| 3.1.25 (F) | El sistema puede listar clientes cuya facturación anual debe revisarse. Ante cambios de… | FR-039, FR-040 |
| 3.1.26 (F) | Monitorización: Pantalla de salud del sistema que represente última ejecución de generación… | FR-138 |
| 3.1.27 (F) | A la hora de crear un cliente, comprobar que es posible su creación para evitar duplicados… | FR-019 |
| 3.1.28 (F) | Historial de eventos relevantes asociados y relacionados con cada cliente. | FR-026 |
| 3.1.29 (F) | Listar clientes favoritos o recientes para un acceso más cómodo. | FR-110 |
| 3.1.30 (F) | Tablón de objetivos de facturación mensuales o anuales y progreso de los mismos en el… | FR-121 |
| 3.1.31 (F) | Alertas configurables según eventos concretos o preferencias, siempre respetando los… | FR-105 |
| 3.1.32 (F) | Calendario laboral para la correcta organización de los diferentes eventos próximos. | FR-106 |
| 3.1.33 (F) | Representación del estado del cliente relacionado a la facturación. | FR-025 |
| 3.1.34 (F) | Bandeja de aprobaciones ante cambios, cada una con su estado (pendiente/aprobada/rechazada)… | FR-097 |
| 3.1.35 (F) | Posibilidad para los administradores de marcar periodos administrativos como cerrados… | FR-071, FR-072 |
| 3.1.36 (F) | Exportación de un paquete mensual o trimestral para gestoría: listado de facturas emitidas,… | FR-122, FR-123 |
| 3.1.37 (F) | Observación de emails y WhatsApp: Mensajes en cola, enviados, entregados si el proveedor lo… | FR-078 |
| 3.1.38 (F) | Por cada cliente registrar los canales de comunicación permitidos. | FR-024, FR-074 |
| 3.1.39 (F) | Los archivos exportados deben llevar fecha/hora, filtros usados y usuario que lo generó en… | FR-124–FR-126 |
| 3.2.1 (NF) | Las facturas pueden ser representadas con un PDF. Dicho PDF debe mantener la estética de la… | FR-066, FR-067, NFR-02 |
| 3.2.2 (NF) | Concurrencia y coherencia a la hora de modificar facturas de forma simultánea. | NFR-18 |
| 3.2.3 (NF) | Realización y restauración de copias de seguridad automáticas de datos y documentos según… | NFR-19 |
| 3.2.4 (NF) | Comunicaciones y contraseñas cifradas para mantener la seguridad de datos sensibles. | NFR-20 |
| 3.2.5 (NF) | Rendimiento: En situaciones generales, las tareas normales deben poder realizarse de forma… | NFR-05, NFR-06, NFR-12, NFR-21 |
| 3.2.6 (NF) | Escalabilidad: El uso de la aplicación web debe estar preparada para el acceso cómodo de… | NFR-22 |
| 3.2.7 (NF) | Disponibilidad: Las tareas automáticas mensuales deben ser reintentables. Ante fallos,… | NFR-23 |
| 3.2.8 (NF) | Idiomas disponibles español e inglés (preferiblemente para las plantillas, y para la… | NFR-24 |
| 3.2.9 (NF) | Fechas y zona horaria: Mantener consistencia ante eventos tanto dentro como fuera de España. | NFR-25 |
| 3.2.10 (NF) | La interfaz móvil debe ser cómoda, accesible y contar con las funciones esenciales… | NFR-26 |
| 3.2.11 (NF) | Adjuntos: Además del PDF de factura, puede haber pedido del cliente, presupuesto aceptado,… | FR-153–FR-155, NFR-27 |
| 3.2.12 (NF) | El contrato API debe estar documentado para el correcto uso de las funciones. | NFR-28 |
| 3.2.13 (NF) | Las diferentes integraciones como CSV o Stripe deben estar encapsuladas para poder… | NFR-29 |
| 3.2.14 (NF) | Manejo de errores: el desarrollador recibe error con identificador y stack trace en… | NFR-30 |
| 3.2.15 (NF) | Cierre de sesión automático debido a inactividad o desactivación de cuenta. Protección… | NFR-16, NFR-31 |
| 3.2.16 (NF) | Las reglas económicas importantes como el prorrateo o el cálculo de impuestos deben contar… | NFR-32 |
| 3.2.17 (NF) | Posibilidad de utilizar datos de prueba para testeo y que no se mezclen con los datos de… | NFR-33 |
| 3.2.18 (NF) | Interfaz consistente y cómoda para su navegación. | NFR-34 |
| 3.2.19 (NF) | Métricas de rendimiento para comprobar el correcto funcionamiento del sistema. | NFR-35 |

*F = clasificado como funcional en el documento original; NF = clasificado como no funcional.*
