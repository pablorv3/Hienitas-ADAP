# Documento de Diseño de Software (SDD) — Modelo de Clases y Patrones de Diseño

**Proyecto:** Sistema Interno de Gestión de Facturación y Cobros (**Turbine**)  
**Módulo:** Diseño Detallado de Dominio, Servicios e Integraciones  
**Diagrama Asociado:** `doc/src/diagrama-clases.pu` _(exportado a `doc/diagrama-clases.pdf`)_

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

### 2.3. Patrón State (`IEstadoFactura`)

- **Problema Preliminar:** Una factura atraviesa distintas fases como documento (borrador, pendiente de aprobación, lista para emitir, emitida, anulada, rectificada). Las operaciones permitidas cambian radicalmente según la fase: por ejemplo, una factura emitida no puede editar sus líneas y solo se corrige mediante rectificativa, y una factura en trimestre cerrado queda bloqueada. Gestionar esto con condicionales dentro de `Factura` generaría un código frágil y propenso a estados inconsistentes.
- **Solución:** El patrón State modela solo el ciclo del _documento_ (`EstadoBorrador`, `EstadoPendienteAprobacion`, `EstadoListaEmitir`, `EstadoEmitida`, `EstadoAnulada`, `EstadoRectificada`), que determina qué operaciones están permitidas. El estado de _cobro_ (pendiente, vencida, pagada, incobrable) no es un estado persistido: `Factura.estadoCobro()` lo calcula a partir de los cobros conciliados, la fecha de vencimiento y la marca `incobrable`.
- **Beneficio:** Garantiza transiciones de estado válidas por diseño y evita mezclar «emitida» con «pagada» (`AD-06`, `A-14`).

### 2.4. Patrón Strategy (`IEstrategiaCalculoTarifa`)

- **Problema Preliminar:** El cálculo de la cuota de una suscripción varía según la casuística comercial (RB-02): mensualidad completa el día 1, prorrateo proporcional si el alta se produce a mitad de mes, o exención de prorrateo por acuerdo comercial.
- **Solución:** Se delega el algoritmo de cálculo de importe a una familia de estrategias intercambiables (`CalculoEstandarStrategy`, `CalculoProrrateoAltaStrategy`, `CalculoExcepcionComercialStrategy`) bajo la interfaz `IEstrategiaCalculoTarifa`.
- **Beneficio:** Permite añadir nuevas modalidades tarifarias o promociones sin modificar la clase `Suscripcion`.

### 2.5. Patrón Observer (`GestorEventosDominio`)

- **Problema Preliminar:** Cuando ocurre un evento relevante (se emite una factura, se detecta una discrepancia de importe o se solicita un cambio de rango), deben dispararse acciones secundarias: grabar una traza en `AuditoriaLog` y generar una alerta en la bandeja del usuario (`Notificacion`). Acoplar la auditoría y las notificaciones dentro de `Factura` o `Cliente` violaría el principio de responsabilidad única.
- **Solución:** Se implementa un bus ligero de eventos de dominio (`GestorEventosDominio`) al que se suscriben observadores especializados (`AuditoriaObserver`, `NotificacionObserver`).
- **Beneficio:** Desacopla completamente la lógica financiera de la trazabilidad y las alertas internas.

### 2.6. Patrón Decorator (`IDocumentoPDF`)

- **Problema Preliminar:** Los documentos PDF requieren distintas variantes visuales según el contexto: vista previa de una factura con marca de agua de borrador, factura definitiva o informe exportado con cabecera de auditoría (usuario, fecha y filtros, `FR-100`). Crear subclases para cada combinación provocaría una explosión combinatoria de clases.
- **Solución:** Sobre un componente base (`FacturaPDFBase`), se envuelven decoradores dinámicos que heredan de `DocumentoPDFDecorator` (`MarcaAguaBorradorDecorator`, `CabeceraAuditoriaDecorator`).
- **Beneficio:** Permite componer dinámicamente las capas visuales del PDF en tiempo de ejecución.

### 2.7. Patrón Factory Method (`FacturaFactory`)

- **Problema Preliminar:** Aunque todas las facturas comparten estructura, la forma de construir e inicializar una **factura recurrente** (desde una suscripción), una **factura puntual** (desde conceptos manuales) y una **factura rectificativa** (vinculada a una factura original) sigue reglas y series distintas.
- **Solución:** Se define un creador abstracto (`FacturaFactory`) con fábricas concretas (`FacturaRecurrenteFactory`, `FacturaPuntualFactory`, `FacturaRectificativaFactory`) que encapsulan la validación previa e instanciación.
- **Beneficio:** Asegura que ningún documento financiero se cree sin cumplir sus precondiciones legales y de numeración.

---

## 3. Especificación Detallada de Clases, Interfaces y Métodos

### 3.1. Capa de Orquestación (`<<Facade>>`)

#### Clase `FacturacionFacade`

Coordina los flujos complejos de facturación, cobros y exportación documental, ocultando la complejidad interna del dominio a los controladores y tareas programadas.

- **Métodos:**
  - `+ generarFacturacionMensual(fecha : Date) : ResumenEjecucion`
    - **Qué hace:** Ejecuta el proceso masivo correspondiente al día 1 de cada mes (UC-14). Recupera todas las suscripciones activas (ignorando las pausadas y los clientes con baja efectiva), delega la creación de cada factura a `FacturaRecurrenteFactory`, calcula los importes aplicando la estrategia de tarifa correspondiente y devuelve un objeto `ResumenEjecucion` con el conteo de facturas generadas, importes totales y posibles incidencias, que queda guardado en `EjecucionProceso` (FR-110).
  - `+ emitirYEnviarFactura(facturaId : UUID) : void`
    - **Qué hace:** Oficializa una factura en borrador y la remite al cliente (UC-12, UC-13). Verifica primero mediante `cliente.validarDatosFiscales()` que no falte ningún dato obligatorio (NIF, razón social, domicilio fiscal, país); si falta alguno, aborta la operación indicando el campo exacto. Si la factura incluye excepciones, la pasa a `EstadoPendienteAprobacion` (UC-12 A1). En otro caso, `SerieFacturacion.asignarSiguienteNumero()` asigna el número correlativo, la factura pasa a `EstadoEmitida`, se genera el PDF definitivo mediante `IDocumentoPDF` y se envía a través de `ICanalMensajeria` al primer contacto de facturación y, si rebota, a los demás contactos de facturación activos por prioridad. Cada intento queda registrado como `Envio`. Finalmente, publica el evento `FACTURA_EMITIDA`.
  - `+ conciliarCobrosConfirmados(desde : Timestamp) : ResumenEjecucion`
    - **Qué hace:** Proceso periódico de conciliación (UC-17, `DS-03`). Recupera los cobros confirmados mediante `IConsultarCobroExterno.obtenerCobrosConfirmados(desde)` y, para cada uno, descarta los ya registrados (`fuente`, `eventoExternoId`), localiza la factura y delega en `conciliarPagoExterno`.
  - `+ conciliarPagoExterno(referencia : String) : void`
    - **Qué hace:** Verifica y registra un cobro proveniente de la pasarela externa (UC-17). Consulta los detalles del pago a través de `IConsultarCobroExterno`, localiza la factura asociada, crea la entidad `Cobro` y delega en `factura.registrarCobro(cobro)`. Si no hay factura clara, el cobro queda con `estadoConciliacion = PENDIENTE` para revisión manual (FR-067.2). Si el importe recibido difiere del saldo pendiente (pago parcial o sobrepago), publica un evento para que `NotificacionObserver` genere una alerta de revisión para el responsable.
  - `+ exportarFacturaPDF(facturaId : UUID, esBorrador : Boolean) : byte[]`
    - **Qué hace:** Construye y devuelve el flujo binario (`byte[]`) del documento PDF listo para descarga o vista previa. Si `esBorrador` es `true` (o la factura aún no está en `EstadoEmitida`), envuelve `FacturaPDFBase` con `MarcaAguaBorradorDecorator`. El PDF definitivo se genera una sola vez al emitir y se conserva en `StorageSeguro` (NFR-01).
  - `+ consultarEstadoDeudaCliente(nif : String) : ResumenDeuda`
    - **Qué hace:** Operación expuesta a la API externa (UC-30, FR-114, `DS-08`). Devuelve un resumen mínimo con las facturas emitidas no incobrables cuyo `saldoPendiente()` es mayor que cero.
  - `+ registrarSolicitudBajaExterna(nif : String, motivo : String) : SolicitudAprobacion`
    - **Qué hace:** Registra una solicitud de baja recibida por la API (FR-113, FR-114.1, `DS-08`) como `SolicitudAprobacion` pendiente; ningún sistema externo da de baja directamente al cliente.
  - `- registrarEnvio(factura : Factura, contacto : Contacto, estado : String) : Envio`
    - **Qué hace:** Crea el registro `Envio` de cada intento con el destinatario como instantánea, enlazando los reintentos con el envío original (FR-064, FR-065).

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

- **Métodos:** Implementa `+ enviar(...)` validando el formato del correo electrónico, simulando latencia de envío y controlando rebotes simulados para activar el flujo alternativo de reintento con otro contacto de facturación del cliente.

#### Clase `WhatsAppSimuladoAdapter` _(implementa `ICanalMensajeria`)_

Adaptador simulado para mensajería instantánea vía WhatsApp (canal complementario para recordatorios de cobro).

- **Métodos:** Implementa `+ enviar(...)` validando el prefijo telefónico internacional y adaptando el mensaje al formato soportado por el canal.

---

### 3.3. Núcleo de Dominio Comercial y Facturación

#### Clase `Cliente`

Representa a la empresa cliente de Turbine y encapsula sus datos fiscales, comerciales y reglas de ciclo de vida.

- **Atributos principales:** `id : UUID`, `nif : String`, `nombreComercial : String`, `razonSocial : String`, `domicilioFiscal : String`, `pais : String`, `moneda : String`, `impuestoRate : Decimal`, `estado : EstadoCliente`, `fechaBajaEfectiva : Date`.
- **Enumeración `EstadoCliente`:** `PENDIENTE_ACTIVACION`, `ACTIVO`, `PAUSADO`, `BAJA_SOLICITADA`, `INACTIVO` (A-14). `INACTIVO` es el borrado lógico.
- **Métodos:**
  - `+ validarDatosFiscales() : Boolean`
    - **Qué hace:** Comprueba que todos los campos legalmente exigibles para emitir una factura (`nif`, `razonSocial`, `domicilioFiscal`, `pais`, `moneda`, `impuestoRate`) estén informados y cumplan el formato básico, y que exista al menos un contacto de facturación activo. Es invocada obligatoriamente antes de cualquier emisión definitiva.
  - `+ contactosFacturacion() : List<Contacto>`
    - **Qué hace:** Devuelve los contactos activos con rol `FACTURACION` ordenados por `prioridad`. Es la lista que recorre el envío de facturas y sus reintentos (UC-13, FR-064).
  - `+ solicitarBaja(fechaEfectiva : Date) : void`
    - **Qué hace:** Registra la solicitud de baja (UC-10, FR-113) y pasa el cliente a `BAJA_SOLICITADA`. Desde `fechaEfectiva` no se generan nuevas mensualidades. Si `tieneDeudaPendiente()` es falso, cierra la recurrencia en esa fecha y el cliente pasa a `INACTIVO`. Si es verdadero, mantiene el recobro de las facturas anteriores (FR-115); la deuda no bloquea la baja (Q-04, FR-117).
  - `+ tieneDeudaPendiente() : Boolean`
    - **Qué hace:** Devuelve verdadero si existe alguna factura emitida, no incobrable, con `saldoPendiente() > 0`. Es la consulta que expone la API (FR-114).
  - `+ reactivar() : void`
    - **Qué hace:** Devuelve a `ACTIVO` un cliente `INACTIVO` o con baja solicitada, reutilizando su ficha histórica (FR-118), y dispara el evento de auditoría correspondiente.

#### Clases `Contacto` y `ClienteContacto`

- **`Contacto`:** Persona de contacto del cliente (`id : UUID`, `nombre : String`, `email : String`, `telefono : String`, `idioma : String`, `admiteWhatsApp : Boolean`, `activo : Boolean`). Un contacto inactivo conserva su histórico de envíos (FR-017.1).
- **`ClienteContacto`:** Clase de asociación `N:M` entre `Cliente` y `Contacto` con `rol : String` (`CONTRATACION`, `FACTURACION`, `IMPAGO`) y `prioridad : Integer`.

#### Clase `Etiqueta`

Permite clasificar y segmentar clientes (`id : UUID`, `nombre : String`) para filtrado rápido y generación de informes.

#### Clase `TarifaRango`

Tramo de tarifa según la facturación anual del cliente (Q-07): `id : UUID`, `nombre : String`, `facturacionDesde : Decimal`, `facturacionHasta : Decimal`, `precioMensual : Decimal`, `vigenteDesde : Date`, `vigenteHasta : Date`.

- `+ aplicaA(facturacionAnual : Decimal, fecha : Date) : Boolean`: Indica si el tramo corresponde a una facturación anual y está vigente en la fecha dada. Un cambio de precio crea un tramo nuevo con fecha de efecto y no reescribe facturas emitidas (FR-036).

#### Clase `Suscripcion`

Modela el acuerdo periódico de cobro asociado a un cliente. La periodicidad es siempre mensual (Q-07, RB-03).

- **Atributos principales:** `id : UUID`, `tarifaRango : TarifaRango`, `precioBase : Decimal`, `pausada : Boolean`, `motivoPausa : String`, `fechaInicioPausa : Timestamp`.
- **Métodos:**
  - `+ calcularImportePeriodo(fecha : Date) : Decimal`
    - **Qué hace:** Calcula el importe exacto a facturar para el periodo indicado delegando en la `IEstrategiaCalculoTarifa` asignada (mensualidad completa, prorrateo inicial por días restantes del mes o exención comercial).
  - `+ pausar(motivo : String) : void`
    - **Qué hace:** Suspende temporalmente la generación de facturas recurrentes estableciendo `pausada = true`, registrando el `motivo` obligatorio y fijando `fechaInicioPausa` con la marca de tiempo actual (FR-037). Durante la pausa, los procesos mensuales omiten esta suscripción sin afectar a facturas previas.
  - `+ reanudar() : void`
    - **Qué hace:** Levanta la suspensión (`pausada = false`), limpia los datos de pausa activa y rehabilita la suscripción para el siguiente ciclo de facturación, registrando la trazabilidad en auditoría.

#### Clase `Factura`

Entidad central financiera. Actúa como **Contexto** del patrón _State_ y contiene la composición de sus líneas de detalle.

- **Atributos principales:** `id : UUID`, `numero : String`, `serie : String`, `tipo : String` (`RECURRENTE`, `PUNTUAL`, `RECTIFICATIVA`), `incobrable : Boolean`, `periodoFacturacion : Date`, `fechaEmision : Date`, `fechaVencimiento : Date`, `moneda : String`, `baseImponible : Decimal`, `impuestoTotal : Decimal`, `total : Decimal`, `trimestreCerrado : Boolean`. El ciclo del documento lo determina la asociación `estadoActual : IEstadoFactura` (persistida como `estadoDocumento`).
- **Enumeración `EstadoCobro`:** `PENDIENTE`, `VENCIDA`, `PAGADA`, `INCOBRABLE`. No se almacena: la calcula `estadoCobro()`.
- **Métodos:**
  - `+ agregarLinea(linea : LineaFactura) : void`
    - **Qué hace:** Añade un concepto desglosado delegando en `estadoActual.editarLineas(...)`, que solo lo permite antes de la emisión y fuera de un trimestre cerrado, y dispara automáticamente el recálculo de `baseImponible`, `impuestoTotal` y `total`.
  - `+ recalcularTotales() : void`
    - **Qué hace:** Recorre todas las instancias de `LineaFactura` asociadas y calcula con precisión decimal la suma de subtotales, los impuestos aplicables y el importe total.
  - `+ emitir() : void`
    - **Qué hace:** Delega en `estadoActual.emitir(this)`. Solo `EstadoBorrador` (sin excepciones) y `EstadoListaEmitir` lo permiten: asignan serie y número y transicionan a `EstadoEmitida`.
  - `+ registrarCobro(cobro : Cobro) : void`
    - **Qué hace:** Asocia un `Cobro` conciliado a una factura emitida. No cambia el estado del documento: el efecto se refleja en `saldoPendiente()` y `estadoCobro()`. Rechaza el cobro si la factura no está emitida.
  - `+ estadoCobro() : EstadoCobro`
    - **Qué hace:** Devuelve `INCOBRABLE` si `incobrable` es verdadero; `PAGADA` si los cobros conciliados activos cubren el total (RB-10, RB-18); `VENCIDA` si `saldoPendiente() > 0` y la fecha actual supera `fechaVencimiento`; y `PENDIENTE` en otro caso.
  - `+ saldoPendiente() : Decimal`
    - **Qué hace:** Devuelve `total` menos la suma de los cobros conciliados no revertidos.
  - `+ esTrimestreCerrado() : Boolean`
    - **Qué hace:** Indica si la factura pertenece a un periodo cerrado (RB-08).
  - `+ cerrarTrimestre() : void`
    - **Qué hace:** Establece `trimestreCerrado = true`, bloqueando cualquier modificación ordinaria sobre la factura (UC-27).
  - `+ cancelarPorIncobrable(motivo : String) : void`
    - **Qué hace:** Marca `incobrable = true` solo tras aprobarse una `SolicitudAprobacion` de tipo `INCOBRABLE` (UC-19, FR-070.1), conservando el motivo y deteniendo los recordatorios. No equivale a pagada.

#### Clase `LineaFactura`

Representa cada ítem desglosado dentro de una factura (`numeroLinea : Integer`, `concepto : String`, `cantidad : Integer`, `precioUnitario : Decimal`, `descuento : Decimal`, `impuestoRate : Decimal`).

- **Métodos:**
  - `+ calcularSubtotal() : Decimal`
    - **Qué hace:** Calcula el importe neto de la línea aplicando la fórmula $(\text{cantidad} \times \text{precioUnitario}) - \text{descuento}$.

#### Clase `Cobro`

Registra cada movimiento de pago recibido de una fuente externa (`id : UUID`, `fuente : String`, `eventoExternoId : String`, `referenciaExterna : String`, `fechaPago : Timestamp`, `importe : Decimal`, `moneda : String`, `estadoConciliacion : String` — `PENDIENTE`, `AUTOMATICA`, `MANUAL`, `REVERTIDA` —, `fechaConciliacion : Timestamp`). Puede existir sin factura asociada mientras está pendiente de conciliación manual (FR-067.2).

- `+ revertir(usuario : Usuario, motivo : String) : void`: Revierte una conciliación errónea sin borrar el evento, registrando quién, cuándo y por qué (FR-069.1).

#### Clase `Envio`

Registra cada intento de envío de una factura (`id : UUID`, `destinatario : String`, `canal : String`, `estado : String` — `EN_COLA`, `ENVIADO`, `ENTREGADO`, `FALLIDO`, `REBOTADO` —, `fecha : Timestamp`, `detalleError : String`), con el contacto y la plantilla usados y una autoasociación `envioOriginal` (UC-13, FR-064, FR-065).

- `+ esReintento() : Boolean`: Devuelve verdadero si el envío referencia a un envío original.

#### Clase `SerieFacturacion`

Serie de numeración configurada por el superadministrador (`codigo : String`, `descripcion : String`, `ultimoNumero : Integer`, `activa : Boolean`; FR-052).

- `+ asignarSiguienteNumero() : String`: Incrementa `ultimoNumero` dentro de la transacción de emisión, con la fila de la serie bloqueada, y devuelve el número asignado (FR-051, RB-07, AD-03).

#### Entidad de soporte `EjecucionProceso`

La tabla `EJECUCION_PROCESO` del modelo físico (registro de ejecución de procesos automáticos, FR-110) no aparece en el diagrama de clases porque no tiene comportamiento de dominio propio: la escribe `FacturacionFacade` al terminar cada proceso a partir del `ResumenEjecucion`.

---

### 3.4. Máquina de Estados de Factura (`<<State>>`)

#### Interfaz `IEstadoFactura`

Define las operaciones sensibles al ciclo del documento:

- `+ editarLineas(factura : Factura, linea : LineaFactura) : void`
- `+ solicitarAprobacion(factura : Factura, motivo : String) : void`
- `+ aprobar(factura : Factura, aprobador : Usuario) : void`
- `+ emitir(factura : Factura) : void`
- `+ anular(factura : Factura, motivo : String) : void`
- `+ rectificar(factura : Factura) : Factura`

Cualquier operación no permitida en un estado lanza una excepción de dominio. El cobro no forma parte de esta interfaz: es un estado derivado (`Factura.estadoCobro()`).

#### Clases Concretas de Estado

1. **`EstadoBorrador`:** Permite `editarLineas` y `anular` (un borrador nunca emitido puede descartarse dejando traza). `solicitarAprobacion` transiciona a `EstadoPendienteAprobacion` cuando hay excepciones (UC-12 A1, FR-079). `emitir` valida que no haya excepciones, asigna serie y número y transiciona a `EstadoEmitida`.
2. **`EstadoPendienteAprobacion`:** Bloquea la edición. `aprobar` transiciona a `EstadoListaEmitir` tras aprobarse la `SolicitudAprobacion` (UC-16); un rechazo devuelve la factura a `EstadoBorrador`.
3. **`EstadoListaEmitir`:** `emitir` asigna serie y número y transiciona a `EstadoEmitida`.
4. **`EstadoEmitida`:** Rechaza `editarLineas`. `rectificar` crea la rectificativa mediante `FacturaRectificativaFactory` y transiciona la original a `EstadoRectificada` (RB-06); `anular` transiciona a `EstadoAnulada` cuando proceda. Ambas comprueban que el periodo no esté cerrado (RB-08).
5. **`EstadoAnulada`** y **`EstadoRectificada`:** Estados finales; rechazan todas las operaciones.

---

### 3.5. Estrategias de Cálculo de Cuotas (`<<Strategy>>`)

#### Interfaz `IEstrategiaCalculoTarifa`

- `+ calcular(suscripcion : Suscripcion, fecha : Date) : Decimal`

#### Estrategias Concretas

1. **`CalculoEstandarStrategy`:** Devuelve el 100 % de `suscripcion.precioBase` (aplicable a facturaciones ordinarias el día 1 de mes).
2. **`CalculoProrrateoAltaStrategy`:** Calcula la parte proporcional cuando un cliente se da de alta en un día distinto del 1 (RB-02 / FR-041), aplicando la regla de A-16:
   $$\text{ImporteProrrateado} = \text{precioBase} \times \frac{\text{DíasRestantesMes}}{\text{DíasTotalesMes}}$$
3. **`CalculoExcepcionComercialStrategy`:** Devuelve `0.00` (o el importe pactado explícitamente) cuando existe una exención de prorrateo aprobada comercialmente (FR-043), dejando constancia explícita en el concepto de la línea.

---

### 3.6. Creación y Decoración de Documentos (`<<Factory Method>>` y `<<Decorator>>`)

#### Creadores de Documentos (`<<Factory Method>>`)

- **`FacturaFactory` (Abstracto):**
  - `+ crearFactura(cliente : Cliente) : Factura` _(Método Fábrica)_: Define la firma para instanciar una factura en `EstadoBorrador` para un cliente.
- **`FacturaRecurrenteFactory`:** Toma una `Suscripcion` activa, valida que el cliente no tenga la baja efectiva, invoca `suscripcion.calcularImportePeriodo(...)` y construye la factura periódica con `periodoFacturacion` y serie ordinaria.
- **`FacturaPuntualFactory`:** Toma conceptos manuales, valida datos fiscales y construye una factura de cobro único.
- **`FacturaRectificativaFactory`:** Toma una `Factura` ya emitida, exige un motivo de rectificación, vincula `facturaOriginalId` (RB-06) y genera la factura de abono o corrección con su serie específica.

#### Generación Visual de PDF (`<<Decorator>>`)

- **Interfaz `IDocumentoPDF`:**
  - `+ generarBytes() : byte[]`: Genera la representación binaria en PDF del documento.
- **`FacturaPDFBase`:** Renderiza la plantilla corporativa base de Turbine con logotipo, datos fiscales del emisor y cliente, tabla de `LineaFactura` y desglose de impuestos.
- **`DocumentoPDFDecorator` (Abstracto):** Implementa `IDocumentoPDF` y mantiene una referencia `envuelto : IDocumentoPDF` a la que delega.
- **`MarcaAguaBorradorDecorator`:** Superpone en diagonal la marca de agua _"BORRADOR - VISTA PREVIA SIN VALIDEZ FISCAL"_ antes de la emisión definitiva (FR-053).
- **`CabeceraAuditoriaDecorator`:** Añade a los informes exportados en PDF una cabecera con el usuario que los generó, la fecha y los filtros usados (`usuario : Usuario`, `filtros : String`; FR-100).

---

### 3.7. Gobierno, Seguridad, Eventos y Auditoría

#### Clase `Usuario`

Representa a cualquier empleado interno de Turbine que opera en la aplicación. Centraliza la autenticación y el control de acceso basado en roles (RBAC) mediante un modelo de entidad única.

- **Atributos principales:** `id : UUID`, `email : String`, `password : String`, `rol : String` (`OPERATIVO`, `ADMIN`, `SUPERADMIN`), `activo : Boolean`, `creadoEn : Timestamp`.
- **Métodos:**
  - `+ autenticar(password : String) : Boolean`
    - **Qué hace:** Verifica que el usuario esté activo (`activo == true`), que su correo pertenezca al dominio corporativo configurado (FR-002, FR-103) y valida el hash de la contraseña proporcionada.
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

#### Clases de Eventos (`<<Observer>>`)

- **`GestorEventosDominio` (`<<Subject>>`):** Mantiene la lista `observadores : List<IObservadorEvento>` y expone `suscribir`, `desuscribir` y `publicarEvento(tipo : String, payload : Map)`.
- **`IObservadorEvento`:** Contrato `+ actualizar(tipo : String, payload : Map) : void`.
- **`AuditoriaObserver`:** Inserta un registro inmutable en `AuditoriaLog` por cada evento sensible.
- **`NotificacionObserver`:** Crea la `Notificacion` y un `RecibeNotificacion` por cada destinatario (responsable titular o suplente, administradores o usuarios mencionados).

#### Clases de Soporte: `Incidencia`, `Comentario` y `Adjunto`

- **`Incidencia`:** Incidencia de facturación asociada a un cliente y, opcionalmente, a una factura (`id`, `titulo`, `descripcion`, `estado`, `fechaCreacion`), creada por un `Usuario` (UC-20).
  - `+ cerrar(resolucion : String) : void`: Cierra la incidencia registrando la resolución (FR-078).
- **`Comentario`:** Nota interna sobre una factura o una incidencia (`id`, `texto`, `restringidoAdmin`, `fechaCreacion`) con menciones `@usuario` (UC-21, FR-127).
- **`Adjunto`:** Metadatos de un fichero guardado en `StorageSeguro` (`id`, `nombreArchivo`, `rutaStorage`, `tipoMime`, `fechaSubida`), asociado a un cliente o a una factura (UC-21, FR-125, FR-126).

---

## 4. Colaboración en Tiempo de Ejecución (Flujos Clave)

Para ilustrar cómo interactúan las clases documentadas, se resumen dos flujos críticos:

1. **Emisión y Envío de una Factura (UC-12 / UC-13):**
   - El controlador invoca `FacturacionFacade.emitirYEnviarFactura(facturaId)`.
   - La fachada llama a `cliente.validarDatosFiscales()`. Si devuelve `false`, se lanza una excepción detallando los campos faltantes.
   - `factura.emitir()` delega en `EstadoBorrador`, que asigna el número mediante `SerieFacturacion.asignarSiguienteNumero()` y transiciona a `EstadoEmitida`; se llama a `exportarFacturaPDF(facturaId, false)` (usando `FacturaPDFBase`).
   - La fachada obtiene el texto renderizado desde `PlantillaComunicacion.renderizar(...)` e invoca `ICanalMensajeria.enviar(...)` sobre `EmailSimuladoAdapter` para cada contacto de `cliente.contactosFacturacion()` hasta que uno entregue, registrando cada intento como `Envio`.
   - Se publica `FACTURA_EMITIDA` en `GestorEventosDominio`, haciendo que `AuditoriaObserver` inserte una fila inmutable en `AuditoriaLog`.

2. **Conciliación de un Cobro desde Stripe (UC-17):**
   - El proceso invoca `FacturacionFacade.conciliarCobrosConfirmados(desde)`, que obtiene los cobros mediante `IConsultarCobroExterno.obtenerCobrosConfirmados(desde)` (`StripeSimuladoAdapter`).
   - Para cada cobro nuevo (no registrado con la misma `fuente` y `eventoExternoId`), se localiza la factura y se ejecuta `factura.registrarCobro(cobro)`.
   - Si el cobro cubre el saldo pendiente, `factura.estadoCobro()` pasa a devolver `PAGADA`; el estado del documento sigue siendo `EstadoEmitida`.
   - Si el importe difiere del saldo pendiente o no hay factura clara, el cobro queda con `estadoConciliacion = PENDIENTE` y se emite un evento que activa `NotificacionObserver`, creando una `Notificacion` con `requiereAccion = true` para el `Usuario` responsable principal del cliente.

---

## 5. Matriz de Trazabilidad (Requisitos $\leftrightarrow$ Diseño)

| Requisito / Caso de Uso                                                      | Elemento del Diagrama de Clases                                         | Patrón / Mecanismo Aplicado         |
| :--------------------------------------------------------------------------- | :---------------------------------------------------------------------- | :---------------------------------- |
| **RB-02 / FR-040 a FR-043** (Prorrateos y excepciones)                       | `Suscripcion` + `IEstrategiaCalculoTarifa`                              | Patrón **Strategy**                 |
| **FR-031 / Q-07 / UC-08 / UC-09** (Tramo de tarifa)                          | `Suscripcion` + `TarifaRango`                                           | Tabla versionada de tramos          |
| **A-14 / UC-12 / UC-15** (Ciclo del documento)                               | `Factura` + `IEstadoFactura` (seis estados de documento)                | Patrón **State**                    |
| **FR-066 / RB-10 / UC-17 / UC-19** (Estado de cobro)                         | `Factura.estadoCobro()`, `Factura.saldoPendiente()`, `Cobro`            | Estado derivado, no persistido      |
| **FR-067 / UC-17** (Consulta Stripe simulada)                                | `IConsultarCobroExterno` + `StripeSimuladoAdapter`                      | Patrón **Adapter**                  |
| **FR-061 a FR-065 / UC-13** (Envío Email y WhatsApp simulado, reintentos)    | `ICanalMensajeria` + adaptadores, `Envio`, `Cliente.contactosFacturacion()` | Patrón **Adapter**              |
| **FR-113 a FR-117 / UC-10** (Baja sin bloqueo por deuda)                     | `Cliente.solicitarBaja()`, `Cliente.tieneDeudaPendiente()`              | Estado de cliente (`EstadoCliente`) |
| **FR-056 / FR-053 / UC-12** (Vista previa borrador y PDF oficial)            | `IDocumentoPDF` + `FacturaPDFBase` + `MarcaAguaBorradorDecorator`       | Patrón **Decorator**                |
| **RB-06 / UC-11 / UC-14 / UC-15** (Facturas recurrentes, puntuales y rectificativas) | `FacturaFactory` y fábricas concretas                           | Patrón **Factory Method**           |
| **FR-109 / UC-22 / UC-28** (Auditoría inmutable y alertas)                   | `GestorEventosDominio` + `AuditoriaObserver` + `AuditoriaLog`           | Patrón **Observer** + _Append-Only_ |
| **UC-01 a UC-04** (Control de acceso por rol)                                | `Usuario.tienePermiso(accion : String)`                                 | Encapsulamiento RBAC                |
