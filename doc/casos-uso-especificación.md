# Especificación de casos de uso

**Proyecto:** Sistema interno de facturación y cobros de Turbine
**Asignatura:** Análisis y Diseño de Aplicaciones (curso 2026-27)

## 1. Actores

| Actor | Descripción |
|---|---|
| Usuario interno | Actor abstracto: cualquier persona autenticada de Turbine. |
| Usuario operativo | Trabaja sobre los clientes que tiene asignados (responsable interno). |
| Administrador | Gestiona operaciones, clientes, facturas, asignaciones y aprobaciones. Hereda todo lo del usuario operativo, sobre todos los clientes. |
| Superadministrador | Configuración profunda y faceta técnica. Hereda todo lo del administrador. |
| Sistema de pagos (Stripe, simulado) | Sistema externo que informa de cobros y permite consultarlos. |
| Proveedor de mensajería (correo / WhatsApp, simulado) | Sistema externo que entrega facturas y avisos y devuelve el resultado. |
| Sistema externo vía API (p. ej., Brain) | Sistema que consulta o solicita acciones a través de la API. |
| Procesos automáticos | Tareas programadas del propio sistema (generación mensual, recordatorios, informes). |

**Generalización de actores:** Superadministrador → Administrador → Usuario operativo → Usuario interno. En cada caso de uso se indica el actor más básico que puede ejecutarlo, y los actores superiores lo heredan.

## 2. Resumen de casos de uso

| ID | Caso de uso | Actor principal |
|---|---|---|
| UC-01 | Autenticarse | Usuario interno |
| UC-02 | Gestionar usuarios y roles | Superadministrador |
| UC-03 | Gestionar asignación de clientes | Administrador |
| UC-04 | Configurar el sistema | Superadministrador |
| UC-05 | Gestionar plantillas de comunicación | Administrador |
| UC-06 | Gestionar clientes | Usuario operativo |
| UC-07 | Importar clientes desde CSV | Usuario operativo |
| UC-08 | Gestionar suscripción | Usuario operativo |
| UC-09 | Revisar rango de tarifa anual | Usuario operativo |
| UC-10 | Gestionar baja y reactivación de cliente | Usuario operativo |
| UC-11 | Gestionar facturas puntuales y presupuestos | Usuario operativo |
| UC-12 | Emitir factura | Usuario operativo |
| UC-13 | Enviar factura y gestionar entregas | Usuario operativo, Proveedor de mensajería |
| UC-14 | Generar facturas recurrentes | Procesos automáticos |
| UC-15 | Rectificar factura | Administrador |
| UC-16 | Gestionar aprobaciones | Administrador |
| UC-17 | Conciliar cobros | Sistema de pagos, Usuario operativo |
| UC-18 | Gestionar recordatorios e impagos | Procesos automáticos, Usuario operativo |
| UC-19 | Cancelar deuda incobrable | Administrador |
| UC-20 | Gestionar incidencias de facturación | Usuario operativo |
| UC-21 | Gestionar comentarios y adjuntos | Usuario operativo |
| UC-22 | Gestionar notificaciones y alertas | Usuario interno |
| UC-23 | Consultar pantalla de trabajo diario | Usuario interno |
| UC-24 | Buscar y filtrar | Usuario interno |
| UC-25 | Generar informes y exportar datos | Administrador |
| UC-26 | Consultar dashboard financiero | Administrador |
| UC-27 | Cerrar periodos | Administrador |
| UC-28 | Consultar auditoría | Administrador |
| UC-29 | Monitorizar el sistema | Administrador |
| UC-30 | Consultar y operar vía API | Sistema externo |

## 3. Relaciones entre casos de uso

Se mantienen al mínimo para que el diagrama quede limpio:

- UC-14 Generar facturas recurrentes **«include»** UC-12 Emitir factura.
- UC-12 Emitir factura **«include»** UC-13 Enviar factura y gestionar entregas.

El resto de casos de uso se relacionan solo con sus actores. Las aprobaciones (UC-16) y las incidencias (UC-20) se describen dentro de los flujos de los casos que las originan, sin añadir más relaciones.

---

## 4. Especificaciones

### UC-01. Autenticarse
- **Objetivo:** Acceder de forma segura a la aplicación y recuperar la contraseña si se olvida.
- **Actores:** Usuario interno.
- **Precondiciones:** El usuario tiene una cuenta activa creada por el superadministrador.
- **Flujo principal:**
  1. El usuario introduce su correo y contraseña.
  2. El sistema comprueba que el correo pertenece al dominio corporativo y que las credenciales son correctas.
  3. El sistema abre la sesión y muestra la pantalla de trabajo diario.
- **Flujos alternativos:**
  - A1. El usuario solicita recuperar la contraseña: el sistema envía un enlace temporal y permite fijar una nueva contraseña.
  - A2. La sesión expira por inactividad: el sistema la cierra y pide autenticarse de nuevo.
- **Excepciones:**
  - E1. Credenciales o dominio no válidos: el sistema rechaza el acceso con un mensaje comprensible.
  - E2. Demasiados intentos fallidos: el sistema limita el acceso temporalmente.
  - E3. Cuenta desactivada: el sistema rechaza el acceso.
  - E4. Enlace de recuperación caducado: el sistema lo rechaza y permite solicitar otro.
- **Postcondiciones:** Existe una sesión activa con el rol y los permisos del usuario.
- **Requisitos:** FR-001, FR-002, FR-005, NFR-03, NFR-15, NFR-19, NFR-29.

### UC-02. Gestionar usuarios y roles
- **Objetivo:** Crear, invitar y desactivar usuarios, y asignarles rol y permisos.
- **Actores:** Superadministrador.
- **Precondiciones:** El superadministrador está autenticado.
- **Flujo principal:**
  1. El superadministrador accede a la gestión de usuarios.
  2. Crea o invita a un usuario con un correo del dominio corporativo.
  3. Asigna el rol (superadministrador, administrador u operativo) y sus permisos.
  4. El sistema guarda el usuario y le envía la invitación.
- **Flujos alternativos:**
  - A1. Modificar el rol o los permisos de un usuario existente.
  - A2. Desactivar un usuario: el sistema invalida sus sesiones activas y conserva el rastro de sus acciones.
- **Excepciones:**
  - E1. El correo no es del dominio autorizado o ya existe: el sistema rechaza la operación.
- **Postcondiciones:** El usuario queda creado, modificado o desactivado, y el cambio queda auditado.
- **Requisitos:** FR-003, FR-004, FR-006, FR-007, FR-013, NFR-15.

### UC-03. Gestionar asignación de clientes
- **Objetivo:** Asignar y desasignar clientes a los usuarios (individualmente o en bloque) y delegar temporalmente.
- **Actores:** Administrador.
- **Precondiciones:** Existen el cliente y el usuario destino.
- **Flujo principal:**
  1. El administrador selecciona uno o varios clientes.
  2. Elige el usuario responsable al que asignarlos.
  3. El sistema muestra a cuántos elementos afecta y pide confirmación.
  4. El sistema aplica la asignación, transfiere las tareas pendientes al nuevo responsable y muestra el resultado con éxitos y errores.
- **Flujos alternativos:**
  - A1. Desasignar un cliente sin nuevo responsable inmediato.
  - A2. Delegar los clientes de un usuario a otro durante un intervalo; la delegación caduca al terminar.
- **Excepciones:**
  - E1. Algún cliente no puede reasignarse: el sistema lo indica en el resultado y continúa con los demás.
- **Postcondiciones:** El histórico conserva quién hizo cada acción anterior; el cambio queda auditado.
- **Requisitos:** FR-008, FR-013 a FR-016, FR-122 a FR-124.

### UC-04. Configurar el sistema
- **Objetivo:** Gestionar los parámetros globales del sistema.
- **Actores:** Superadministrador.
- **Precondiciones:** El superadministrador está autenticado.
- **Flujo principal:**
  1. El superadministrador accede a la configuración.
  2. Modifica parámetros: plazo de pago, cadencia de recordatorios, dominio de correo, datos fiscales y logo, series, monedas, impuestos, tarifas, umbrales de aprobación, calendario laboral, canales y políticas de automatización.
  3. El sistema valida los cambios.
  4. El superadministrador confirma y el sistema los aplica solo a futuras operaciones.
- **Flujos alternativos:**
  - A1. Cambios delicados: el sistema ofrece una simulación o previsualización antes de aplicarlos.
- **Excepciones:**
  - E1. Valores no válidos: el sistema los rechaza indicando el motivo.
- **Postcondiciones:** La configuración queda actualizada y auditada; las facturas ya emitidas no cambian.
- **Requisitos:** FR-002, FR-022, FR-052, FR-072, FR-086, FR-102 a FR-105, NFR-08.

### UC-05. Gestionar plantillas de comunicación
- **Objetivo:** Editar las plantillas de correos al cliente y consultar qué versión se usó en cada envío.
- **Actores:** Administrador.
- **Precondiciones:** El administrador está autenticado.
- **Flujo principal:**
  1. El administrador selecciona una plantilla (factura, recordatorio, vencimiento, impago, disputa resuelta, revisión de datos).
  2. Edita su contenido usando variables.
  3. El sistema previsualiza el resultado y avisa de las variables que falten.
  4. El administrador guarda y el sistema crea una nueva versión.
- **Flujos alternativos:**
  - A1. Enviar un correo de prueba al propio administrador.
  - A2. Consultar el historial de versiones o la plantilla usada en un envío concreto.
- **Excepciones:**
  - E1. Variable inexistente o faltante: el sistema avisa y no permite guardar.
- **Postcondiciones:** La nueva versión queda disponible para futuros envíos.
- **Requisitos:** FR-106 a FR-108, NFR-22.

### UC-06. Gestionar clientes
- **Objetivo:** Dar de alta, consultar y modificar clientes de forma rápida, con sus datos fiscales, contactos, estado y etiquetas.
- **Actores:** Usuario operativo.
- **Precondiciones:** El usuario está autenticado y, si es operativo, el cliente está asignado a él.
- **Flujo principal:**
  1. El usuario inicia el alta de un cliente.
  2. Introduce datos comerciales y fiscales (razón social, NIF, país, moneda, domicilio), personas de contacto y canales permitidos.
  3. El sistema propone valores por defecto y comprueba duplicados por identificador fiscal.
  4. El usuario confirma y el sistema crea el cliente con su responsable interno y estado inicial.
- **Flujos alternativos:**
  - A1. Modificar los datos, contactos, etiquetas o estado de un cliente existente.
  - A2. Consultar la ficha y el historial de eventos del cliente.
  - A3. Se detecta un posible duplicado: el sistema avisa, y el usuario decide si continúa o reutiliza la ficha. Nunca se fusionan fichas automáticamente.
  - A4. Guardar la ficha como borrador y completarla después.
- **Excepciones:**
  - E1. Faltan datos obligatorios o son inválidos: el sistema indica qué campo falla.
  - E2. Intento de eliminar el único contacto de facturación de un cliente activo: el sistema avisa.
- **Postcondiciones:** El cliente queda registrado o actualizado y el cambio queda auditado.
- **Requisitos:** FR-009, FR-017 a FR-026, FR-088, FR-119 a FR-121.

### UC-07. Importar clientes desde CSV
- **Objetivo:** Cargar clientes de forma masiva desde un fichero CSV con control previo.
- **Actores:** Usuario operativo.
- **Precondiciones:** El usuario dispone de un fichero CSV.
- **Flujo principal:**
  1. El usuario sube el fichero.
  2. Mapea las columnas del fichero con los campos del sistema.
  3. El sistema valida las filas y muestra una previsualización con errores y duplicados detectados.
  4. El usuario confirma la importación.
  5. El sistema importa las filas aceptadas y registra la operación con el resumen de creados, actualizados y rechazados.
- **Flujos alternativos:**
  - A1. El usuario corrige el fichero y lo vuelve a cargar.
  - A2. El usuario decide importar solo las filas válidas, según la política definida.
- **Excepciones:**
  - E1. Formato o codificación no válidos: el sistema rechaza el fichero.
- **Postcondiciones:** Los clientes válidos quedan creados y la importación queda registrada.
- **Requisitos:** FR-019, FR-027 a FR-030.

### UC-08. Gestionar suscripción
- **Objetivo:** Dar de alta y mantener la relación comercial mensual de un cliente: tarifa, excepciones, prorrateo y pausas.
- **Actores:** Usuario operativo.
- **Precondiciones:** El cliente existe y tiene los datos fiscales necesarios.
- **Flujo principal:**
  1. El usuario crea la suscripción del cliente indicando fecha de inicio, plan y tarifa.
  2. El sistema sugiere el rango de cobro y muestra el cálculo del prorrateo si el alta no es el día 1.
  3. El usuario confirma.
  4. El sistema activa la suscripción (o la deja pendiente de activación si la fecha es futura).
- **Flujos alternativos:**
  - A1. Registrar excepciones comerciales (descuento, mes gratuito, precio pactado, no cobrar prorrateo): quedan explícitas y pueden requerir aprobación (UC-16).
  - A2. Pausar la suscripción indicando motivo, fecha de inicio y reactivación prevista; no se generan mensualidades durante la pausa.
  - A3. Llegada la fecha de reactivación: el sistema avisa al responsable y reactiva según la política configurada.
- **Excepciones:**
  - E1. Faltan datos fiscales: el sistema bloquea la activación e indica qué falta.
- **Postcondiciones:** La suscripción queda activa, pendiente o pausada y el cambio queda auditado.
- **Requisitos:** FR-031, FR-033, FR-037 a FR-043, FR-079.

### UC-09. Revisar rango de tarifa anual
- **Objetivo:** Revisar cada año el rango de facturación de los clientes y aprobar el cambio de tarifa.
- **Actores:** Usuario operativo.
- **Precondiciones:** Existen clientes con revisión anual pendiente.
- **Flujo principal:**
  1. El usuario consulta el listado de clientes cuya revisión toca.
  2. Introduce la facturación anual verificada (con fuente y fecha).
  3. El sistema propone el rango de tarifa correspondiente.
  4. El responsable valida y aprueba el cambio.
  5. El sistema registra quién lo aprobó, cuándo y los precios anterior y nuevo, con fecha de efecto.
- **Flujos alternativos:**
  - A1. El dato no está disponible: la revisión queda pendiente.
  - A2. El responsable rechaza el cambio: se mantiene la tarifa actual.
- **Excepciones:**
  - E1. Se corrige una revisión antigua: el sistema calcula el impacto, pero no emite facturas retroactivas sin acción explícita y aprobada.
- **Postcondiciones:** La nueva tarifa aplica a partir de su fecha de efecto y no reescribe facturas anteriores.
- **Requisitos:** FR-032, FR-034 a FR-036.

### UC-10. Gestionar baja y reactivación de cliente
- **Objetivo:** Registrar la baja de un cliente manteniendo el recobro de sus deudas, y reactivar su ficha si vuelve.
- **Actores:** Usuario operativo.
- **Precondiciones:** El cliente existe.
- **Flujo principal:**
  1. El usuario registra la solicitud de baja con su fecha efectiva.
  2. El sistema consulta si el cliente tiene facturas pendientes.
  3. Si no las tiene, el sistema cierra la recurrencia en la fecha efectiva.
  4. El sistema marca el cliente como inactivo conservando sus datos según la ley (borrado lógico).
- **Flujos alternativos:**
  - A1. Hay facturas pendientes: el sistema deja de generar mensualidades desde la fecha de baja y mantiene vivo el flujo de recobro.
  - A2. El cliente vuelve: el usuario reactiva la ficha histórica en lugar de crear un duplicado.
- **Excepciones:**
  - E1. Estado incompatible o ficha inexistente: el sistema informa del motivo.
- **Postcondiciones:** El cliente queda dado de baja (o reactivado), sin nuevas mensualidades y con el recobro vigente si hay deuda.
- **Requisitos:** FR-113 a FR-118.
- **Nota:** Q-04 sigue pendiente; se aplica la interpretación de A1 (la baja se registra y el recobro continúa).

### UC-11. Gestionar facturas puntuales y presupuestos
- **Objetivo:** Preparar facturas puntuales (varias líneas) y presupuestos convertibles en factura, antes de emitirlas.
- **Actores:** Usuario operativo.
- **Precondiciones:** El cliente existe y está asignado al usuario (si es operativo).
- **Flujo principal:**
  1. El usuario crea una factura puntual para un cliente.
  2. Añade líneas con concepto, cantidad, precio, descuento e impuesto, reutilizando conceptos frecuentes del catálogo si procede.
  3. El sistema calcula bases, impuestos y total.
  4. El usuario guarda la factura como borrador.
- **Flujos alternativos:**
  - A1. Crear un presupuesto, marcarlo como aceptado o rechazado y, si se acepta, convertirlo en factura reutilizando sus líneas.
  - A2. Corregir o eliminar un borrador que nunca se emitió.
- **Excepciones:**
  - E1. Datos de línea inválidos: el sistema indica el error.
  - E2. Intento de modificar una factura ya emitida: se deriva a UC-15.
- **Postcondiciones:** El borrador o presupuesto queda guardado; los cambios del catálogo no alteran facturas existentes.
- **Requisitos:** FR-010, FR-044 a FR-047, FR-058.

### UC-12. Emitir factura
- **Objetivo:** Dar por definitiva una factura, asignándole serie y número y generando su PDF.
- **Actores:** Usuario operativo.
- **Precondiciones:** Existe una factura en borrador o aprobada para emitir, y el periodo no está cerrado.
- **Flujo principal:**
  1. El usuario solicita la emisión.
  2. El sistema valida los datos imprescindibles y muestra una vista previa.
  3. El usuario confirma.
  4. El sistema asigna serie y número de forma segura, genera el PDF con la imagen corporativa y marca la factura como emitida.
  5. El sistema envía la factura (UC-13).
- **Flujos alternativos:**
  - A1. La factura incluye excepciones (importe modificado, descuento extraordinario, vencimiento inusual, importe muy alto): queda pendiente de aprobación (UC-16) indicando el motivo, y se emite cuando se aprueba.
  - A2. Importe atípico para el cliente: el sistema advierte y el usuario decide continuar o no (Q-05).
- **Excepciones:**
  - E1. Faltan datos fiscales: el sistema bloquea la emisión e indica qué falta.
  - E2. Doble clic o reintento: el sistema no duplica la emisión ni consume otro número.
  - E3. Periodo cerrado: el sistema impide la emisión ordinaria.
- **Postcondiciones:** La factura queda emitida con su número único y su PDF original conservado.
- **Requisitos:** FR-051 a FR-057, FR-079, NFR-01, NFR-10, NFR-12, NFR-17.

### UC-13. Enviar factura y gestionar entregas
- **Objetivo:** Enviar la factura al cliente y controlar el resultado de la entrega.
- **Actores:** Usuario operativo, Proveedor de mensajería.
- **Precondiciones:** La factura está emitida y el cliente tiene un contacto de facturación.
- **Flujo principal:**
  1. El sistema envía la factura por correo al contacto de facturación (también al emitirla automáticamente).
  2. El proveedor de mensajería devuelve el resultado.
  3. El sistema registra destinatario, fecha, canal y estado (en cola, enviado, entregado, fallido, reintentado) y marca la factura como enviada.
- **Flujos alternativos:**
  - A1. El cliente admite WhatsApp con consentimiento: se usa como canal complementario.
  - A2. Reintentar un envío fallido sin duplicar la factura.
- **Excepciones:**
  - E1. El correo rebota o falla: el sistema no marca la factura como notificada, abre una incidencia de entrega y avisa al responsable. Este puede corregir el contacto y reenviar, o el sistema prueba otro contacto de facturación.
- **Postcondiciones:** Queda registrado qué envío fue el original y cuáles los reintentos.
- **Requisitos:** FR-061 a FR-065, FR-023, NFR-10.

### UC-14. Generar facturas recurrentes
- **Objetivo:** Generar y enviar automáticamente las mensualidades de las suscripciones activas.
- **Actores:** Procesos automáticos.
- **Precondiciones:** Es día 1 de mes (o el alta de una suscripción exige prorrateo), y hay suscripciones activas.
- **Flujo principal:**
  1. El proceso selecciona las suscripciones que corresponde facturar.
  2. Para cada una genera la factura con sus conceptos y el prorrateo si aplica.
  3. Emite la factura (UC-12), que la envía (UC-13).
  4. Registra la ejecución: clientes procesados, facturas creadas y fallos.
- **Flujos alternativos:**
  - A1. Suscripción pausada, con fecha de inicio futura o con baja efectiva: no se genera factura.
  - A2. Excepciones sin aprobar: la factura queda pendiente de aprobación (UC-16).
- **Excepciones:**
  - E1. Falla un cliente concreto (por ejemplo, sin contacto de facturación): se registra y se notifica a su responsable, y el resto continúa.
  - E2. El sistema estaba caído a las 00:00: al recuperarse detecta lo no generado y lo crea sin duplicar.
- **Postcondiciones:** Cada mensualidad existe una sola vez y queda registro de la ejecución.
- **Requisitos:** FR-040, FR-041, FR-044, FR-061, FR-110, NFR-09, NFR-21.

### UC-15. Rectificar factura
- **Objetivo:** Corregir una factura ya emitida mediante un documento rectificativo vinculado.
- **Actores:** Administrador.
- **Precondiciones:** La factura está emitida y no puede modificarse.
- **Flujo principal:**
  1. El administrador selecciona la factura original y solicita la rectificación.
  2. El sistema propone los datos a rectificar.
  3. El administrador ajusta y confirma.
  4. Si el importe lo exige, se solicita aprobación (UC-16).
  5. El sistema emite el documento rectificativo vinculado a la original y actualiza el saldo del cliente.
- **Flujos alternativos:**
  - A1. Cambios menores en una factura emitida: se registran de forma trazable.
- **Excepciones:**
  - E1. Periodo cerrado: solo mediante un procedimiento excepcional y auditado.
- **Postcondiciones:** La factura original se conserva y queda enlazada con su rectificativa.
- **Requisitos:** FR-048, FR-049, FR-050, FR-060.

### UC-16. Gestionar aprobaciones
- **Objetivo:** Solicitar y resolver la aprobación de operaciones sensibles.
- **Actores:** Usuario operativo (solicita), Administrador (resuelve).
- **Precondiciones:** Existe una operación que requiere aprobación (descuento, factura inusual, cambio de rango, rectificativa, cancelación de deuda).
- **Flujo principal:**
  1. El sistema deja la operación pendiente, indicando por qué requiere aprobación.
  2. El aprobador abre su bandeja de aprobaciones.
  3. Revisa la solicitud, la aprueba o la rechaza y escribe un comentario.
  4. El sistema notifica al solicitante y continúa o cancela la operación.
- **Flujos alternativos:**
  - A1. El importe supera el umbral configurado: lo resuelve el superadministrador.
  - A2. El aprobador no puede resolver: un administrador reasigna la aprobación.
- **Excepciones:**
  - E1. El aprobador intenta aprobar su propia solicitud cuando la regla lo prohíbe: el sistema lo impide.
- **Postcondiciones:** La aprobación queda en estado aprobada o rechazada, con su comentario, y auditada.
- **Requisitos:** FR-036, FR-050, FR-070, FR-079, FR-080.

### UC-17. Conciliar cobros
- **Objetivo:** Registrar los pagos recibidos y asociarlos a las facturas correspondientes.
- **Actores:** Sistema de pagos, Usuario operativo.
- **Precondiciones:** Existen facturas emitidas pendientes de cobro.
- **Flujo principal:**
  1. El sistema de pagos notifica un cobro (o el sistema lo consulta).
  2. El sistema lo asocia a una factura por importe y referencia.
  3. Si coincide el total, marca la factura como pagada y notifica al responsable.
- **Flujos alternativos:**
  - A1. El importe es menor o mayor: el sistema asocia el movimiento, señala la diferencia y no marca la factura como pagada hasta su revisión; el usuario registra su conclusión.
  - A2. El cobro no tiene referencia clara: queda pendiente de conciliación manual.
  - A3. Se revierte una conciliación errónea: queda trazado quién, cuándo y por qué.
- **Excepciones:**
  - E1. Evento duplicado: el sistema lo ignora sin contabilizar el cobro dos veces.
- **Postcondiciones:** El estado de la factura y el movimiento asociado quedan registrados y auditados.
- **Requisitos:** FR-066 a FR-069, NFR-10, NFR-17, NFR-27.

### UC-18. Gestionar recordatorios e impagos
- **Objetivo:** Avisar al cliente antes y después del vencimiento hasta que se detecte el pago.
- **Actores:** Procesos automáticos, Usuario operativo.
- **Precondiciones:** Existe una factura emitida y enviada pendiente de cobro.
- **Flujo principal:**
  1. El sistema calcula el vencimiento (por defecto 5 días, o el plazo pactado del cliente).
  2. Envía los recordatorios previos (por defecto a los 3 días y al vencimiento).
  3. Pasado el vencimiento, marca la factura como vencida e inicia el flujo de impago con recordatorios periódicos (por defecto cada 2 o 3 días).
  4. Notifica al responsable interno.
  5. Al detectarse el pago el flujo se detiene.
- **Flujos alternativos:**
  - A1. El usuario suspende temporalmente los recordatorios de una factura, indicando motivo y fecha de revisión; internamente sigue figurando como pendiente.
  - A2. Una persona interviene (por ejemplo, cambiando el estado del cliente) y el flujo se detiene.
- **Excepciones:**
  - E1. El plazo pactado es mayor que el general: el sistema no marca impago antes de tiempo.
- **Postcondiciones:** Cada aviso queda registrado con la plantilla utilizada.
- **Requisitos:** FR-071 a FR-075, FR-077.

### UC-19. Cancelar deuda incobrable
- **Objetivo:** Cancelar o declarar incobrable una deuda mediante una acción controlada.
- **Actores:** Administrador.
- **Precondiciones:** Existe una factura vencida y sin cobrar.
- **Flujo principal:**
  1. El administrador solicita declarar la deuda incobrable o cancelarla, indicando el motivo.
  2. El sistema solicita la aprobación correspondiente (UC-16).
  3. Una vez aprobada, el sistema registra el nuevo estado (incobrable, no «pagada»), detiene los recordatorios y audita la acción.
- **Flujos alternativos:**
  - A1. La aprobación es rechazada: la deuda continúa en recobro.
- **Excepciones:**
  - E1. Falta el motivo: el sistema no permite continuar.
- **Postcondiciones:** La deuda queda marcada como incobrable o cancelada, con motivo y aprobador registrados.
- **Requisitos:** FR-070, FR-079, FR-080.

### UC-20. Gestionar incidencias de facturación
- **Objetivo:** Dar seguimiento interno a problemas de facturación asociados a un cliente y, opcionalmente, a una factura.
- **Actores:** Usuario operativo.
- **Precondiciones:** El cliente existe y está asignado al usuario (si es operativo).
- **Flujo principal:**
  1. El usuario abre una incidencia (por ejemplo, «NIF incorrecto» o «pago no localizado») con responsable, estado y prioridad.
  2. Añade comentarios y adjuntos (UC-21).
  3. Resuelve el problema y cierra la incidencia indicando la resolución.
- **Flujos alternativos:**
  - A1. La incidencia afecta al cobro: puede pausar los recordatorios.
  - A2. La incidencia se origina automáticamente (por ejemplo, por un rebote de correo).
- **Excepciones:**
  - E1. Cerrar sin resolución: el sistema exige registrarla.
- **Postcondiciones:** La incidencia queda cerrada con su resolución, o abierta con su responsable.
- **Requisitos:** FR-076 a FR-078.

### UC-21. Gestionar comentarios y adjuntos
- **Objetivo:** Comentar entre compañeros y adjuntar documentos de apoyo a un cliente, factura o incidencia.
- **Actores:** Usuario operativo.
- **Precondiciones:** El usuario tiene acceso al elemento sobre el que actúa.
- **Flujo principal:**
  1. El usuario escribe un comentario sobre una factura o incidencia (puede ser restringido a administradores) y, si quiere, menciona a otro usuario.
  2. El sistema guarda el comentario y notifica a la persona mencionada.
- **Flujos alternativos:**
  - A1. Adjuntar un documento con tipo y descripción; el sistema valida formato y tamaño.
  - A2. Descargar un adjunto: el sistema solo lo permite según los permisos del elemento asociado.
- **Excepciones:**
  - E1. Formato o tamaño no permitido: el sistema rechaza el adjunto.
  - E2. Se menciona a un usuario sin acceso al elemento: el sistema lo impide.
- **Postcondiciones:** El comentario o adjunto queda asociado al elemento; los comentarios no se envían al cliente.
- **Requisitos:** FR-012, FR-125 a FR-127, NFR-16, NFR-25.

### UC-22. Gestionar notificaciones y alertas
- **Objetivo:** Recibir los eventos relevantes, distinguir los informativos de los que requieren acción y configurar alertas.
- **Actores:** Usuario interno.
- **Precondiciones:** El usuario está autenticado.
- **Flujo principal:**
  1. El usuario abre su bandeja de notificaciones.
  2. Consulta las notificaciones de sus clientes (el administrador ve la bandeja global).
  3. Abre una y accede al elemento relacionado.
  4. Marca como leídas las informativas; las que requieren acción permanecen pendientes hasta resolver lo notificado.
- **Flujos alternativos:**
  - A1. Elegir qué notificaciones recibir, por la aplicación, por email o en resumen diario.
  - A2. Configurar alertas (por ejemplo, «factura vencida más de 15 días»), según sus permisos.
- **Excepciones:**
  - E1. Marcar como leída una notificación que requiere acción: el sistema la mantiene pendiente.
- **Postcondiciones:** Las notificaciones quedan con su estado actualizado.
- **Requisitos:** FR-081 a FR-083, FR-085, FR-127.

### UC-23. Consultar pantalla de trabajo diario
- **Objetivo:** Ver al entrar lo que requiere atención y resolverlo con pocos clics.
- **Actores:** Usuario interno.
- **Precondiciones:** El usuario está autenticado.
- **Flujo principal:**
  1. Al iniciar sesión, el sistema muestra facturas que vencen pronto, vencidas, pagos por conciliar, problemas de entrega, aprobaciones pendientes, revisiones anuales y tareas asignadas.
  2. El usuario selecciona un elemento.
  3. El sistema lo abre y permite resolverlo directamente.
- **Flujos alternativos:**
  - A1. No hay elementos pendientes: el sistema lo indica.
- **Excepciones:**
  - E1. Error al cargar un bloque: el sistema muestra el resto y avisa del fallo con un código de incidencia.
- **Postcondiciones:** Ninguna modificación; muestra solo lo permitido al rol y a los clientes asignados.
- **Requisitos:** FR-084, NFR-04.

### UC-24. Buscar y filtrar
- **Objetivo:** Localizar rápidamente clientes y facturas y acceder a favoritos y recientes.
- **Actores:** Usuario interno.
- **Precondiciones:** El usuario está autenticado.
- **Flujo principal:**
  1. El usuario escribe en el buscador (nombre comercial, razón social, NIF, contacto, número de factura o correo).
  2. El sistema muestra los resultados permitidos por sus permisos.
  3. El usuario aplica filtros (ubicación, sector, etiquetas, estado, tipo) y abre el resultado.
- **Flujos alternativos:**
  - A1. Guardar un filtro frecuente para reutilizarlo.
  - A2. Acceder a los clientes favoritos o recientes.
- **Excepciones:**
  - E1. Sin resultados: el sistema lo indica.
  - E2. Acceso directo por URL a un elemento no permitido: el servidor lo rechaza.
- **Postcondiciones:** Ninguna modificación de datos de negocio.
- **Requisitos:** FR-087 a FR-089, NFR-04 a NFR-06.

### UC-25. Generar informes y exportar datos
- **Objetivo:** Obtener informes de cobros y exportarlos respetando los permisos.
- **Actores:** Administrador.
- **Precondiciones:** El administrador está autenticado.
- **Flujo principal:**
  1. El administrador elige el informe (semanal, mensual o trimestral) y los filtros (periodo, cliente, responsable, país, sector, estado, tipo, moneda).
  2. El sistema genera el informe, en segundo plano si es pesado, y avisa cuando está listo.
  3. El administrador lo consulta y lo exporta a CSV o PDF.
- **Flujos alternativos:**
  - A1. Programar un resumen semanal o mensual para personas autorizadas.
  - A2. Regenerar un informe previamente exportado mientras se conserve.
- **Excepciones:**
  - E1. Exportación sobre datos no permitidos: el sistema limita el contenido a lo autorizado.
- **Postcondiciones:** La exportación queda auditada con fecha, filtros y usuario.
- **Requisitos:** FR-090 a FR-093, FR-100 a FR-102, FR-111, NFR-11.

### UC-26. Consultar dashboard financiero
- **Objetivo:** Conocer la situación económica de Turbine con cifras explicables.
- **Actores:** Administrador.
- **Precondiciones:** El administrador está autenticado.
- **Flujo principal:**
  1. El administrador abre el dashboard.
  2. El sistema muestra lo facturado (semana, mes, trimestre, año), cobrado, pendiente, vencido, tiempo medio de cobro, evolución, clientes con mayor deuda, ingresos recurrentes esperados, facturación puntual y previsión basada en recurrencias.
  3. El administrador pulsa una cifra y el sistema muestra el detalle que la compone.
- **Flujos alternativos:**
  - A1. Definir y consultar objetivos mensuales o anuales y su progreso (solo informativo).
- **Excepciones:**
  - E1. Un usuario operativo intenta acceder: el sistema lo impide.
- **Postcondiciones:** Ninguna modificación de datos.
- **Requisitos:** FR-094 a FR-099, NFR-06, NFR-12.

### UC-27. Cerrar periodos
- **Objetivo:** Marcar un mes o trimestre como cerrado para impedir cambios ordinarios.
- **Actores:** Administrador.
- **Precondiciones:** El periodo ha terminado.
- **Flujo principal:**
  1. El administrador selecciona el periodo.
  2. El sistema muestra un resumen y avisa de elementos pendientes.
  3. El administrador confirma el cierre.
  4. El sistema bloquea los cambios ordinarios en facturas de ese periodo.
- **Flujos alternativos:**
  - A1. Reabrir un periodo con permiso superior y motivo.
- **Excepciones:**
  - E1. Intento de modificar un elemento de un periodo cerrado: el sistema lo impide salvo procedimiento excepcional auditado.
- **Postcondiciones:** El periodo queda cerrado, y el cierre o la reapertura quedan auditados.
- **Requisitos:** FR-059, FR-060.
- **Nota:** Q-08 sigue pendiente; se asume cierre manual.

### UC-28. Consultar auditoría
- **Objetivo:** Saber quién hizo qué y cuándo en la aplicación.
- **Actores:** Administrador.
- **Precondiciones:** El administrador está autenticado.
- **Flujo principal:**
  1. El administrador abre el registro de auditoría.
  2. Filtra por usuario, tipo de acción, cliente, factura o fecha.
  3. El sistema muestra los eventos con su autor y datos relevantes.
- **Flujos alternativos:**
  - A1. Consultar la línea temporal de negocio de un cliente.
- **Excepciones:**
  - E1. Intento de modificar o borrar un registro: no existe esa operación.
- **Postcondiciones:** El registro permanece inalterado.
- **Requisitos:** FR-016, FR-025, FR-101, FR-109, FR-104.

### UC-29. Monitorizar el sistema
- **Objetivo:** Comprobar que las automatizaciones y las integraciones funcionan.
- **Actores:** Administrador.
- **Precondiciones:** El administrador está autenticado.
- **Flujo principal:**
  1. El administrador abre la pantalla de salud.
  2. El sistema muestra la última generación recurrente, la cola de correos, los errores de integración recientes y el último backup correcto.
  3. El administrador consulta el registro de ejecución de un proceso (clientes procesados, facturas creadas, fallos y motivo).
- **Flujos alternativos:**
  - A1. Un proceso falló: el sistema alerta para que no pase en silencio.
- **Excepciones:**
  - E1. Error técnico: se muestra un código de incidencia, nunca información técnica sensible.
- **Postcondiciones:** Ninguna modificación de datos.
- **Requisitos:** FR-110, FR-112, NFR-18, NFR-28, NFR-33.

### UC-30. Consultar y operar vía API
- **Objetivo:** Permitir que otro sistema consulte información y registre solicitudes respetando las reglas de la aplicación.
- **Actores:** Sistema externo.
- **Precondiciones:** El sistema externo dispone de credenciales de API válidas.
- **Flujo principal:**
  1. El sistema externo envía una consulta (por ejemplo, si un cliente tiene facturas pendientes).
  2. El sistema valida credenciales y permisos.
  3. Devuelve el detalle mínimo autorizado.
- **Flujos alternativos:**
  - A1. El sistema externo notifica «cliente solicita baja»: se crea la tarea correspondiente y la baja sigue el flujo de UC-10.
- **Excepciones:**
  - E1. Credenciales o permisos insuficientes: la API rechaza la petición.
  - E2. Se supera el límite de peticiones: la API limita el acceso.
- **Postcondiciones:** Las llamadas relevantes quedan registradas.
- **Requisitos:** FR-113, FR-114, NFR-26.

---

## 5. Notas y decisiones

- **Series de facturación (Q-06):** FR-052 las asigna al administrador, mientras que el cliente habló del superadministrador. Se incluyen en UC-04 (Superadministrador) a la espera de aclaración.
- **Acciones masivas** (FR-122 a FR-124) se tratan como variante de los casos de uso donde aplican (UC-03), no como un caso de uso propio.
- **Cobertura:** los requisitos funcionales FR-001 a FR-127 quedan cubiertos; los no funcionales se aplican de forma transversal y se citan donde tienen mayor relación.
