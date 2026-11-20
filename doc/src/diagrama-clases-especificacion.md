# Documento de Diseño de Software (SDD) — Modelo de Clases y Patrones de Diseño

**Proyecto:** Sistema Interno de Gestión de Facturación y Cobros (**Turbine**)  
**Módulo:** Diseño Detallado de Dominio, Servicios e Integraciones  
**Estado:** Validado para Implementación

---

## 1. Introducción y Contexto Preliminar

### 1.1. Propósito del Documento

Este documento describe la arquitectura orientada a objetos del núcleo de negocio de **Turbine**, detallando la estructura del **Diagrama de Clases UML**, las decisiones arquitectónicas tomadas mediante patrones de diseño y el contrato funcional (atributos, métodos y reglas de negocio) de cada clase e interfaz. Su objetivo es servir como guía de referencia directa tanto para el equipo de desarrollo como para la validación técnica frente a los requisitos funcionales y no funcionales del sistema.

### 1.2. Contexto y Restricciones de Alcance

El diseño del modelo de clases responde directamente a las fronteras funcionales definidas para Turbine:

- **No es una pasarela de pago:** El sistema no procesa cargos ni almacena tarjetas de crédito. Su responsabilidad se limita a **consultar, verificar y conciliar** cobros confirmados por plataformas externas (Stripe), además de gestionar el ciclo de vida contable de las facturas.
- **Desacoplamiento de proveedores externos:** En esta fase, las integraciones con Stripe, correo electrónico y WhatsApp son **simuladas**, pero el código de negocio debe quedar totalmente aislado de dicha simulación para permitir su sustitución por proveedores reales sin modificar la lógica central.
- **Aplicación administrativa interna:** No existe portal de clientes. Toda la operativa recae sobre usuarios internos de Turbine con distintos niveles de privilegio (`OPERATIVO`, `ADMIN`, `SUPERADMIN`).
- **Inmutabilidad y trazabilidad financiera:** Toda operación crítica (cambios de tarifas, aprobaciones, emisiones, cobros y cierres trimestrales) debe quedar auditada de forma inalterable, y las facturas emitidas dentro de un trimestre cerrado no pueden ser editadas, exigiendo la emisión de una factura rectificativa.

### 1.3. Principios Rectores del Diseño

1. **Separación de Responsabilidades (SRP) y Principio Abierto/Cerrado (OCP):** Evitar clases monolíticas cargadas de condicionales (`if/else` o `switch`) mediante el uso de polimorfismo (patrones _State_, _Strategy_ y _Decorator_).
2. **Inversión de Dependencias (DIP):** La lógica de negocio nunca depende de clases concretas de infraestructura (como Stripe o WhatsApp), sino de interfaces abstractas (_Adapters_).
3. **Precisión Financiera:** Todos los importes monetarios y porcentajes impositivos utilizan tipos `Decimal` de precisión exacta, evitando errores de redondeo de coma flotante.

---

## 2. Decisiones Arquitectónicas y Patrones de Diseño (Preliminar)

Antes de detallar cada clase, se justifica el porqué de los **7 patrones de diseño** seleccionados para resolver los problemas estructurales y de comportamiento del dominio de Turbine:

### 2.1. Patrón Facade (`FacturacionFacade`)

- **Problema Preliminar:** Los casos de uso principales (como la facturación mensual automática o la conciliación de un cobro) requieren coordinar múltiples objetos: validar permisos, consultar suscripciones, calcular prorrateos, generar PDFs, enviar mensajes y registrar auditorías. Si los controladores REST o los _Cron Jobs_ llamaran directamente a todas estas clases, la lógica de negocio quedaría dispersa y duplicada.
- **Solución:** Se introduce `FacturacionFacade` como punto de entrada único hacia el subsistema de facturación.
- **Beneficio:** Simplifica la interfaz para la capa API y los procesos programados, centralizando la transaccionalidad y el orden de ejecución de cada flujo.

### 2.2. Patrón Adapter (`IConsultarCobroExterno` y `ICanalMensajeria`)

- **Problema Preliminar:** Turbine necesita comunicarse con Stripe para consultar pagos y con servicios de Email/WhatsApp para enviar facturas y recordatorios de impago. En esta fase estos servicios son simulados, y en producción cada proveedor tiene su propia API, formatos y códigos de error.
- **Solución:** Se definen dos interfaces de dominio con estereotipo `<<Adapter>>`:
  1. `IConsultarCobroExterno`, implementada por `StripeSimuladoAdapter`.
  2. `ICanalMensajeria`, implementada por `EmailSimuladoAdapter` y `WhatsAppSimuladoAdapter`.
- **Beneficio:** El núcleo de Turbine invoca siempre los mismos métodos estándar (`consultarEstadoPago`, `enviar`). Sustituir el simulador por el SDK oficial de Stripe o SendGrid en el futuro solo requerirá crear una nueva clase adaptadora sin tocar una sola línea del dominio.

### 2.3. Patrón State (`EstadoFactura`)

- **Problema Preliminar:** Una factura atraviesa distintos estados (`Pendiente`, `Pagada`, `Impagada`). Las operaciones permitidas cambian radicalmente según el estado: por ejemplo, una factura `Pagada` no puede volver a cobrarse ni pasar a `Impagada`, y una factura en trimestre cerrado queda bloqueada. Gestionar esto con condicionales dentro de `Factura` generaría un código frágil y propenso a estados inconsistentes.
- **Solución:** Se encapsula cada estado en una clase concreta (`EstadoPendiente`, `EstadoPagada`, `EstadoImpagada`) que implementa la interfaz `EstadoFactura`.
- **Beneficio:** Garantiza transiciones de estado válidas por diseño y cumple estrictamente las reglas de negocio de cobros y bloqueo contable.

### 2.4. Patrón Strategy (`IEstrategiaCalculoCuota`)

- **Problema Preliminar:** El cálculo de la cuota de una suscripción varía según la casuística comercial (RB-02): mensualidad completa el día 1, prorrateo proporcional si el alta se produce a mitad de mes, o exención de prorrateo por acuerdo comercial.
- **Solución:** Se delega el algoritmo de cálculo de importe a una familia de estrategias intercambiables (`EstrategiaMensualidadCompleta`, `EstrategiaProrrateoInicial`, `EstrategiaExcepcionComercial`) bajo la interfaz `IEstrategiaCalculoCuota`.
- **Beneficio:** Permite añadir nuevas modalidades tarifarias o promociones sin modificar la clase `Suscripcion`.

### 2.5. Patrón Observer (`GestorEventosDominio`)

- **Problema Preliminar:** Cuando ocurre un evento relevante (se emite una factura, se detecta una discrepancia de importe o se solicita un cambio de rango), deben dispararse acciones secundarias: grabar una traza en `AuditoriaLog` y generar una alerta en la bandeja del usuario (`Notificacion`). Acoplar la auditoría y las notificaciones dentro de `Factura` o `Cliente` violaría el principio de responsabilidad única.
- **Solución:** Se implementa un bus ligero de eventos de dominio (`GestorEventosDominio`) al que se suscriben observadores especializados (`AuditoriaObserver`, `NotificacionObserver`).
- **Beneficio:** Desacopla completamente la lógica financiera de la trazabilidad y las alertas internas.

### 2.6. Patrón Decorator (`IGeneradorPDF`)

- **Problema Preliminar:** Las facturas en PDF requieren distintas variantes visuales según el contexto: vista previa con marca de agua de borrador, factura definitiva, factura con sello de pagada o factura rectificativa con nota legal. Crear subclases para cada combinación (`FacturaPDFBorradorRectificativa`, etc.) provocaría una explosión combinatoria de clases.
- **Solución:** Sobre un componente base (`GeneradorPDFBase`), se envuelven decoradores dinámicos (`MarcaAguaBorradorDecorator`, `SelloPagadaDecorator`, `NotaLegalRectificativaDecorator`).
- **Beneficio:** Permite componer dinámicamente las capas visuales del PDF en tiempo de ejecución.

### 2.7. Patrón Factory Method (`CreadorDocumentoFacturacion`)

- **Problema Preliminar:** Aunque todas las facturas comparten estructura, la forma de construir e inicializar una **factura recurrente** (desde una suscripción), una **factura puntual** (desde conceptos manuales o presupuesto) y una **factura rectificativa** (vinculada a una factura original) sigue reglas y series distintas.
- **Solución:** Se define un creador abstracto con fábricas concretas (`FabricaFacturaRecurrente`, `FabricaFacturaPuntual`, `FabricaFacturaRectificativa`) que encapsulan la validación previa e instanciación.
- **Beneficio:** Asegura que ningún documento financiero se cree sin cumplir sus precondiciones legales y de numeración.

---

## 3. Especificación Detallada de Clases, Interfaces y Métodos

### 3.1. Capa de Orquestación (`<<Facade>>`)

#### Clase `FacturacionFacade`

Coordina los flujos complejos de facturación, cobros y exportación documental, ocultando la complejidad interna del dominio a los controladores y tareas programadas.

- **Métodos:**
  - `+ generarFacturacionMensual(fecha : Date) : ResumenEjecucion`
    - **Qué hace:** Ejecuta el proceso masivo correspondiente al día 1 de cada mes (UC-11). Recupera todas las suscripciones activas (ignorando las pausadas y clientes dados de baja), delega la creación de cada factura a `FabricaFacturaRecurrente`, calcula los importes aplicando la estrategia de tarifa correspondiente y devuelve un objeto `ResumenEjecucion` con el conteo de facturas generadas, importes totales y posibles incidencias.
  - `+ emitirYEnviarFactura(facturaId : UUID) : void`
    - **Qué hace:** Oficializa una factura en borrador y la remite al cliente (UC-12, UC-13). Verifica primero mediante `cliente.validarDatosFiscales()` que no falte ningún dato obligatorio (NIF, razón social, país); si falta alguno, aborta la operación indicando el campo exacto. A continuación, asigna serie y número correlativo oficial, marca `emitida = true`, genera el PDF definitivo mediante `IGeneradorPDF` y lo envía a través de `ICanalMensajeria` (Email principal y, si rebota, contacto suplente o canal complementario). Finalmente, publica el evento `FacturaEmitida`.
  - `+ conciliarPagoExterno(referencia : String) : void`
    - **Qué hace:** Verifica y registra un cobro proveniente de la pasarela externa (UC-17). Consulta los detalles del pago a través de `IConsultarCobroExterno`, localiza la factura asociada, crea la entidad `Cobro` y delega en `factura.registrarCobro(importe)`. Si el importe recibido difiere del total de la factura (pago parcial o sobrepago), publica un evento para que `NotificacionObserver` genere una alerta de revisión para el responsable.
  - `+ exportarFacturaPDF(facturaId : UUID, esBorrador : Boolean) : byte[]`
    - **Qué hace:** Construye y devuelve el flujo binario (`byte[]`) del documento PDF listo para descarga o vista previa. Si `esBorrador` es `true` (o `factura.emitida` es `false`), envuelve `GeneradorPDFBase` con `MarcaAguaBorradorDecorator`. Si la factura ya está pagada o es rectificativa, apila `SelloPagadaDecorator` o `NotaLegalRectificativaDecorator` según corresponda.

---

### 3.2. Integraciones Externas (`<<Adapter>>`)

#### Interfaz `IConsultarCobroExterno`

Contrato estándar que abstrae cualquier proveedor externo de verificación de cobros. Responde al alcance estricto de Turbine: **solo consulta cobros, nunca ejecuta cargos**.

- **Métodos:**
  - `+ obtenerCobrosConfirmados(desde : Timestamp) : List<Cobro>`
    - **Qué hace:** Recupera el listado de transacciones confirmadas por la plataforma externa a partir de una marca de tiempo dada, permitiendo procesos periódicos de sincronización y conciliación.
  - `+ consultarEstadoPago(referencia : String) : String`
    - **Qué hace:** Consulta de forma síncrona el estado actual de una transacción específica identificada por su `referencia` externa (ej. `PAID`, `PENDING`, `FAILED`, `REFUNDED`).

#### Clase `StripeSimuladoAdapter` _(implementa `IConsultarCobroExterno`)_

Implementación simulada de la integración con Stripe para el entorno actual de Turbine.

- **Métodos:** Implementa `obtenerCobrosConfirmados` y `consultarEstadoPago` simulando respuestas realistas de la API de Stripe (incluyendo escenarios de cobro exacto, cobro parcial o referencia inexistente) a partir de datos de prueba controlados.

---

#### Interfaz `ICanalMensajeria`

Contrato único para el envío de comunicaciones externas (facturas emitidas, avisos de vencimiento a los 3 y 5 días, y notificaciones de incidencia).

- **Métodos:**
  - `+ enviar(destinatario : String, asunto : String, cuerpo : String, adjunto : byte[]) : Boolean`
    - **Qué hace:** Envía una comunicación al `destinatario` indicado con el `asunto`, `cuerpo` renderizado y un fichero opcional `adjunto` (como el PDF de la factura). Devuelve `true` si la entrega fue aceptada por el proveedor o `false` si se produjo un rebote o error de entrega.

#### Clase `EmailSimuladoAdapter` _(implementa `ICanalMensajeria`)_

Adaptador simulado para correo electrónico.

- **Métodos:** Implementa `+ enviar(...)` validando el formato del correo electrónico, simulando latencia de envío y controlando rebotes simulados para activar el flujo alternativo de reintento al contacto suplente.

#### Clase `WhatsAppSimuladoAdapter` _(implementa `ICanalMensajeria`)_

Adaptador simulado para mensajería instantánea vía WhatsApp (canal complementario para recordatorios de cobro).

- **Métodos:** Implementa `+ enviar(...)` validando el prefijo telefónico internacional y adaptando el mensaje al formato soportado por el canal.

---

### 3.3. Núcleo de Dominio Comercial y Facturación

#### Clase `Cliente`

Representa a la empresa cliente de Turbine y encapsula sus datos fiscales, comerciales y reglas de ciclo de vida.

- **Atributos principales:** `id : UUID`, `nif : String`, `nombreComercial : String`, `razonSocial : String`, `pais : String`, `moneda : String`, `impuestoRate : Decimal`, `email : String`, `telefono : String`, `activo : Boolean`.
- **Métodos:**
  - `+ validarDatosFiscales() : Boolean`
    - **Qué hace:** Comprueba que todos los campos legalmente exigibles para emitir una factura (`nif`, `razonSocial`, `pais`, `moneda`, `impuestoRate`) estén informados y cumplan el formato básico. Es invocada obligatoriamente antes de cualquier emisión definitiva.
  - `+ puedeDarseDeBaja() : Boolean`
    - **Qué hace:** Evalúa las precondiciones de baja (UC-10). Verifica que el cliente no tenga facturas en estado `Pendiente` o `Impagada` sin resolver ni suscripciones activas sin cancelar, impidiendo inconsistencias contables.
  - `+ reactivar() : void`
    - **Qué hace:** Cambia el estado del cliente inactivo a `activo = true`, habilitando nuevamente la posibilidad de asociarle suscripciones o emitirle facturas, y dispara el evento de auditoría correspondiente.

#### Clase `Etiqueta`

Permite clasificar y segmentar clientes (`id : UUID`, `nombre : String`) para filtrado rápido y generación de informes.

#### Clase `Suscripcion`

Modela el acuerdo periódico de cobro asociado a un cliente.

- **Atributos principales:** `id : UUID`, `rangoCobro : String` (`MENSUAL`, `ANUAL`), `precioBase : Decimal`, `pausada : Boolean`, `motivoPausa : String`, `fechaInicioPausa : Timestamp`.
- **Métodos:**
  - `+ calcularImportePeriodo(fecha : Date) : Decimal`
    - **Qué hace:** Calcula el importe exacto a facturar para el periodo indicado delegando en la `IEstrategiaCalculoCuota` asignada (mensualidad completa, prorrateo inicial por días restantes del mes o exención comercial).
  - `+ pausar(motivo : String) : void`
    - **Qué hace:** Suspende temporalmente la generación de facturas recurrentes estableciendo `pausada = true`, registrando el `motivo` obligatorio y fijando `fechaInicioPausa` con la marca de tiempo actual (FR-037). Durante la pausa, los procesos mensuales omiten esta suscripción sin afectar a facturas previas.
  - `+ reanudar() : void`
    - **Qué hace:** Levanta la suspensión (`pausada = false`), limpia los datos de pausa activa y rehabilita la suscripción para el siguiente ciclo de facturación, registrando la trazabilidad en auditoría.

#### Clase `Factura`

Entidad central financiera. Actúa como **Contexto** del patrón _State_ y contiene la composición de sus líneas de detalle.

- **Atributos principales:** `id : UUID`, `numero : String`, `serie : String`, `tipo : String` (`RECURRENTE`, `PUNTUAL`, `RECTIFICATIVA`), `estadoCobro : String`, `emitida : Boolean`, `fechaEmision : Timestamp`, `fechaVencimiento : Timestamp`, `baseImponible : Decimal`, `impuestoTotal : Decimal`, `total : Decimal`, `trimestreCerrado : Boolean`.
- **Métodos:**
  - `+ agregarLinea(linea : LineaFactura) : void`
    - **Qué hace:** Añade un concepto desglosado a la factura siempre que no esté emitida ni pertenezca a un trimestre cerrado, y dispara automáticamente el recálculo de `baseImponible`, `impuestoTotal` y `total`.
  - `+ calcularTotales() : void`
    - **Qué hace:** Recorre todas las instancias de `LineaFactura` asociadas y calcula con precisión decimal la suma de subtotales, los impuestos aplicables y el importe total.
  - `+ registrarCobro(importe : Decimal) : void`
    - **Qué hace:** Delega la operación en su objeto de estado interno (`estadoActual.procesarPago(this, importe)`), garantizando que las reglas de transición y validación de importes se ejecuten según el estado actual de la factura.
  - `+ marcarComoImpagada() : void`
    - **Qué hace:** Delega en `estadoActual.marcarImpagada(this)` cuando se supera la fecha de vencimiento y los plazos de gracia configurados sin haberse recibido el pago.
  - `+ cerrarTrimestre() : void`
    - **Qué hace:** Establece `trimestreCerrado = true`, bloqueando de forma permanente cualquier modificación sobre la factura.

#### Clase `LineaFactura`

Representa cada ítem desglosado dentro de una factura (`numeroLinea : Integer`, `concepto : String`, `cantidad : Integer`, `precioUnitario : Decimal`, `descuento : Decimal`, `impuestoRate : Decimal`).

- **Métodos:**
  - `+ calcularSubtotal() : Decimal`
    - **Qué hace:** Calcula el importe neto de la línea aplicando la fórmula $(\text{cantidad} \times \text{precioUnitario}) - \text{descuento}$.

#### Clase `Cobro`

Registra cada transacción económica conciliada contra una factura (`id : UUID`, `referenciaExterna : String`, `fechaPago : Timestamp`, `importe : Decimal`).

---

### 3.4. Máquina de Estados de Factura (`<<State>>`)

#### Interfaz `EstadoFactura`

Define las operaciones sensibles al estado de una factura:

- `+ procesarPago(factura : Factura, importe : Decimal) : void`
- `+ marcarImpagada(factura : Factura) : void`

#### Clases Concretas de Estado

1. **`EstadoPendiente`:**
   - `procesarPago`: Verifica que `trimestreCerrado` sea `false`. Compara `importe` con `factura.total`. Si coincide (o completa el saldo pendiente), transiciona la factura a `EstadoPagada` y publica `FacturaPagada`. Si hay discrepancia (falta o sobra dinero), mantiene o ajusta el seguimiento y publica `DiscrepanciaCobroDetectada` (FR-068).
   - `marcarImpagada`: Verifica que la fecha actual supere `factura.fechaVencimiento` y transiciona la factura a `EstadoImpagada`, disparando las notificaciones de morosidad.
2. **`EstadoPagada`:**
   - `procesarPago`: Rechaza la operación lanzando una excepción de dominio (`FacturaYaPagadaException`), impidiendo duplicidades de cobro.
   - `marcarImpagada`: Rechaza la transición, ya que una deuda saldada no puede pasar a impagada.
3. **`EstadoImpagada`:**
   - `procesarPago`: Permite recobrar la deuda atrasada si el trimestre no está cerrado; si se completa el importe, transiciona la factura a `EstadoPagada` y detiene futuros recordatorios de impago.
   - `marcarImpagada`: Operación idempotente (no realiza cambios).

---

### 3.5. Estrategias de Cálculo de Cuotas (`<<Strategy>>`)

#### Interfaz `IEstrategiaCalculoCuota`

- `+ calcular(suscripcion : Suscripcion, fechaReferencia : Date) : Decimal`

#### Estrategias Concretas

1. **`EstrategiaMensualidadCompleta`:** Devuelve el 100 % de `suscripcion.precioBase` (aplicable a facturaciones ordinarias el día 1 de mes).
2. **`EstrategiaProrrateoInicial`:** Calcula la parte proporcional cuando un cliente se da de alta en un día distinto del 1 (RB-02 / FR-041), aplicando:
   $$\text{ImporteProrrateado} = \text{precioBase} \times \frac{\text{DíasRestantesMes}}{\text{DíasTotalesMes}}$$
3. **`EstrategiaExcepcionComercial`:** Devuelve `0.00` (o el importe pactado explícitamente) cuando existe una exención de prorrateo aprobada comercialmente (FR-043), dejando constancia explícita en el concepto de la línea.

---

### 3.6. Creación y Decoración de Documentos (`<<Factory Method>>` y `<<Decorator>>`)

#### Creadores de Documentos (`<<Factory Method>>`)

- **`CreadorDocumentoFacturacion` (Abstracto):**
  - `+ crearFactura(origen : Object) : Factura` _(Método Fábrica)_: Define la firma para instanciar una factura a partir de su entidad origen.
- **`FabricaFacturaRecurrente`:** Toma una `Suscripcion` activa, valida que el cliente esté activo, invoca `suscripcion.calcularImportePeriodo(...)` y construye la factura periódica con serie ordinaria.
- **`FabricaFacturaPuntual`:** Toma conceptos manuales o un presupuesto aceptado, valida datos fiscales y construye una factura de cobro único.
- **`FabricaFacturaRectificativa`:** Toma una `Factura` ya emitida, exige un motivo de rectificación, vincula `facturaOriginalId` (RB-06) y genera la factura de abono o corrección con su serie específica.

#### Generación Visual de PDF (`<<Decorator>>`)

- **Interfaz `IGeneradorPDF`:**
  - `+ generar(factura : Factura) : byte[]`: Genera la representación binaria en PDF de la factura.
- **`GeneradorPDFBase`:** Renderiza la plantilla corporativa base de Turbine con logotipo, datos fiscales del emisor y cliente, tabla de `LineaFactura` y desglose de impuestos.
- **`MarcaAguaBorradorDecorator`:** Envuelve a un `IGeneradorPDF` y superpone en diagonal la marca de agua _"BORRADOR - VISTA PREVIA SIN VALIDEZ FISCAL"_ antes de la emisión definitiva.
- **`SelloPagadaDecorator`:** Añade el sello visual de _"PAGADA"_ junto con la fecha de conciliación cuando `factura.estadoCobro == 'PAGADA'`.
- **`NotaLegalRectificativaDecorator`:** Añade un bloque destacado indicando explícitamente el número y fecha de la factura original a la que rectifica.

---

### 3.7. Gobierno, Seguridad, Eventos y Auditoría

#### Clase `Usuario`

Representa a cualquier empleado interno de Turbine que opera en la aplicación. Centraliza la autenticación y el control de acceso basado en roles (RBAC) mediante un modelo de entidad única.

- **Atributos principales:** `id : UUID`, `email : String`, `password : String`, `rol : String` (`OPERATIVO`, `ADMIN`, `SUPERADMIN`), `activo : Boolean`, `creadoEn : Timestamp`.
- **Métodos:**
  - `+ autenticar(password : String) : Boolean`
    - **Qué hace:** Verifica que el usuario esté activo (`activo == true`), que su correo pertenezca al dominio corporativo `@Turbine.com` y valida el hash de la contraseña proporcionada.
  - `+ tienePermiso(accion : String) : Boolean`
    - **Qué hace:** Evalúa si el `rol` del usuario le autoriza a ejecutar la `accion` solicitada, respetando la jerarquía de privilegios definida en los requisitos (`SUPERADMIN` hereda capacidades de `ADMIN`, y `ADMIN` hereda capacidades de `OPERATIVO`). Encapsula la matriz de permisos en un único punto, evitando condicionales dispersos en los servicios.

#### Clase `SolicitudAprobacion`

Gestiona el flujo de autorización cuando una acción supera los umbrales operativos (ej. cambios de rango de cobro, descuentos excepcionales o cancelación de deuda incobrable).

- **Atributos principales:** `id : UUID`, `tipo : String`, `estado : String` (`PENDIENTE`, `APROBADA`, `RECHAZADA`), `datosCambio : JSON`, `fechaSolicitud : Timestamp`, `fechaResolucion : Timestamp`.
- **Métodos:**
  - `+ aprobar(aprobador : Usuario) : void`
    - **Qué hace:** Valida mediante `aprobador.tienePermiso(...)` que el usuario tenga rango suficiente (Administrador o Superadministrador según el umbral), cambia el estado a `APROBADA`, registra `fechaResolucion`, aplica los cambios contenidos en `datosCambio` sobre la entidad destino y dispara el registro inmutable en auditoría (FR-034).
  - `+ rechazar(aprobador : Usuario, motivo : String) : void`
    - **Qué hace:** Valida los permisos del aprobador, cambia el estado a `RECHAZADA`, guarda el motivo del rechazo y notifica al usuario operativo que inició la solicitud.

#### Clase `Configuracion`

Almacena parámetros globales del sistema (`clave : String`, `valor : String`, `fechaActualizacion : Timestamp`), como los días de recordatorio de impago (3 y 5 días por defecto) o los umbrales monetarios de aprobación.

- **Métodos:**
  - `+ actualizarValor(nuevoValor : String) : void`
    - **Qué hace:** Actualiza el valor del parámetro, refresca `fechaActualizacion` y emite un evento de dominio para registrar el valor anterior y el nuevo en `AuditoriaLog`.

#### Clase `AuditoriaLog`

Representa la bitácora inmutable (_Append-Only_) de trazabilidad del sistema (`id : UUID`, `entidadAfectada : String`, `entidadId : UUID`, `accion : String`, `valoresAnteriores : JSON`, `valoresNuevos : JSON`, `fecha : Timestamp`).

- **Restricción de diseño:** Carece intencionadamente de métodos de modificación (`actualizar`) o borrado (`eliminar`), garantizando la integridad histórica exigida por FR-109 y UC-28.

#### Clases de Notificación y Plantillas

- **`PlantillaComunicacion`:** Almacena plantillas versionadas (`tipo`, `canal`, `asunto`, `cuerpo`, `version`).
  - `+ renderizar(variables : Map) : String`: Sustituye dinámicamente los marcadores de posición (ej. `{{nombreCliente}}`, `{{numeroFactura}}`, `{{importeTotal}}`) por los valores reales del contexto.
- **`Notificacion`:** Representa el mensaje interno generado (`id`, `titulo`, `mensaje`, `requiereAccion : Boolean`).
- **`RecibeNotificacion`:** Clase de asociación entre `Usuario` y `Notificacion` que controla el estado individual de lectura y resolución por destinatario:
  - `+ marcarLeida() : void`: Marca `leida = true` cuando el usuario abre el aviso.
  - `+ marcarResuelta() : void`: Marca `resuelta = true` cuando el usuario completa la tarea requerida (para notificaciones con `requiereAccion == true`).

---

## 4. Colaboración en Tiempo de Ejecución (Flujos Clave)

Para ilustrar cómo interactúan las clases documentadas, se resumen dos flujos críticos:

1. **Emisión y Envío de una Factura (UC-12 / UC-13):**
   - El controlador invoca `FacturacionFacade.emitirYEnviarFactura(facturaId)`.
   - La fachada llama a `cliente.validarDatosFiscales()`. Si devuelve `false`, se lanza una excepción detallando los campos faltantes.
   - Se asigna numeración correlativa y se llama a `exportarFacturaPDF(facturaId, false)` (usando `GeneradorPDFBase`).
   - La fachada obtiene el texto renderizado desde `PlantillaComunicacion.renderizar(...)` e invoca `ICanalMensajeria.enviar(...)` sobre `EmailSimuladoAdapter`.
   - Se publica `FacturaEmitida` en `GestorEventosDominio`, haciendo que `AuditoriaObserver` inserte una fila inmutable en `AuditoriaLog`.

2. **Conciliación de un Cobro desde Stripe (UC-17):**
   - El proceso invoca `FacturacionFacade.conciliarPagoExterno(referencia)`.
   - La fachada consulta el estado mediante `IConsultarCobroExterno.consultarEstadoPago(referencia)` (`StripeSimuladoAdapter`).
   - Si está confirmado, ejecuta `factura.registrarCobro(importe)`.
   - `EstadoPendiente` valida que el trimestre no esté cerrado y transiciona la factura a `EstadoPagada`.
   - Si el importe recibido es menor al total, se emite un evento que activa `NotificacionObserver`, creando una `Notificacion` con `requiereAccion = true` para el `Usuario` responsable principal del cliente.

---

## 5. Matriz de Trazabilidad (Requisitos $\leftrightarrow$ Diseño)

| Requisito / Caso de Uso                                                      | Elemento del Diagrama de Clases                                         | Patrón / Mecanismo Aplicado         |
| :--------------------------------------------------------------------------- | :---------------------------------------------------------------------- | :---------------------------------- |
| **RB-02 / FR-040 a FR-043** (Prorrateos y excepciones)                       | `Suscripcion` + `IEstrategiaCalculoCuota`                               | Patrón **Strategy**                 |
| **FR-066 / UC-14** (Ciclo de vida de cobro)                                  | `Factura` + `EstadoFactura` (`Pendiente`, `Pagada`, `Impagada`)         | Patrón **State**                    |
| **FR-067 / UC-16 / UC-17** (Consulta Stripe simulada)                        | `IConsultarCobroExterno` + `StripeSimuladoAdapter`                      | Patrón **Adapter**                  |
| **FR-061 / UC-13** (Envío Email y WhatsApp simulado)                         | `ICanalMensajeria` + `EmailSimuladoAdapter` / `WhatsAppSimuladoAdapter` | Patrón **Adapter**                  |
| **FR-056 / UC-12** (Vista previa borrador y PDF oficial)                     | `IGeneradorPDF` + `GeneradorPDFBase` + `MarcaAguaBorradorDecorator`     | Patrón **Decorator**                |
| **RB-06 / UC-11 / UC-12** (Facturas recurrentes, puntuales y rectificativas) | `CreadorDocumentoFacturacion` y fábricas concretas                      | Patrón **Factory Method**           |
| **FR-109 / UC-22 / UC-28** (Auditoría inmutable y alertas)                   | `GestorEventosDominio` + `AuditoriaObserver` + `AuditoriaLog`           | Patrón **Observer** + _Append-Only_ |
| **UC-01 a UC-04** (Control de acceso por rol)                                | `Usuario.tienePermiso(accion : String)`                                 | Encapsulamiento RBAC                |
