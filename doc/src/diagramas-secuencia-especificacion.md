# Especificación de Diagramas de Secuencia UML

- **Proyecto:** Sistema Interno de Gestión de Facturación y Cobros (**Turbine**)
- **Entrega:** Entrega 2 — Diseño del Sistema
- **Artefacto:** Modelado Dinámico — Diagramas de Secuencia UML (`DS-01` a `DS-08`)
- **Notación:** UML 2.x (especificación fuente en PlantUML: `doc/src/diagramas-secuencia/*.pu`)
- **Estado:** Validado y trazado con Requisitos, Casos de Uso, ERD, Clases y Arquitectura

---

## 1. Introducción y Criterios Arquitectónicos

### 1.1. Propósito del Documento

Este documento constituye la especificación técnica del comportamiento dinámico de **Turbine**. Mientras que el Diagrama Entidad-Relación Físico define la estructura de persistencia, el Diagrama de Arquitectura establece el despliegue y comunicación entre componentes, y el Diagrama de Clases modela la estructura estática y los patrones de diseño, los **Diagramas de Secuencia UML** validan el intercambio temporal de mensajes entre actores, componentes de infraestructura e instancias de dominio en los escenarios críticos del sistema.

### 1.2. Estrategia de Cobertura y Modularidad

En lugar de modelar de forma aislada los 30 casos de uso del sistema —lo que generaría diagramas redundantes para operaciones CRUD lineales—, se han seleccionado **8 escenarios troncales (`DS-01` a `DS-08`)** bajo los siguientes criterios de ingeniería:

1. **Cobertura total de riesgo técnico:** Se modelan todos los flujos que involucran máquinas de estado complejas, cálculos financieros variables, procesos desatendidos (_batch_), validación fiscal, concurrencia con sistemas externos y seguridad perimetral.
2. **Cobertura del 100 % de los actores y nodos arquitectónicos:** Se incluyen tanto las peticiones síncronas desde la interfaz web (`Frontend SPA`) como los procesos programados (`PlanificadorCron` y `Workers`), la persistencia relacional y documental (`BaseDatosRelacional` y `StorageSeguro`), las pasarelas y canales externos simulados (`StripeSimuladoAdapter`, `EmailSimuladoAdapter`, `WhatsAppSimuladoAdapter`) y las integraciones entrantes vía API (`Sistema Externo / Brain`).
3. **Cobertura por analogía estructural del resto de casos de uso:**
   - **Operaciones CRUD y consultas simples (`UC-01` a `UC-06`, `UC-08`, `UC-09`, `UC-23` a `UC-29`):** Siguen un flujo síncrono lineal `Frontend SPA` $\rightarrow$ `Controlador` $\rightarrow$ `BaseDatosRelacional` sin transiciones de estado críticas ni integraciones externas.
   - **Emisión de factura puntual (`UC-11`):** Reproduce exactamente la secuencia definida en **`DS-01`**, sustituyendo únicamente la instanciación previa por la fábrica de factura puntual.
   - **Cancelación de deuda incobrable (`UC-19`):** Sigue el mismo circuito de autorización administrativa, verificación de permisos, transición de estado y registro en auditoría inmutable modelado en **`DS-05`**.

### 1.3. Convenciones UML 2.x Aplicadas

- **Fidelidad de líneas de vida (_Lifelines_):** Todas las instancias corresponden exactamente a los actores externos y a las clases, interfaces y controladores definidos en el Diagrama de Clases y el Diagrama de Componentes.
- **Tipología de mensajes:** Se distinguen llamadas síncronas (`->`), respuestas o retornos (`-->`), autollamadas internas, instanciaciones dinámicas (`<<create>>`) y focos de control (`activate` / `deactivate`).
- **Fragmentos combinados:**
  - `alt`: Bifurcaciones mutuamente excluyentes según reglas de negocio o control de errores (`if / else`).
  - `opt`: Ejecución condicional de una única rama (`if` sin `else`).
  - `loop`: Iteración repetitiva sobre colecciones de entidades o registros (`for-each`).
  - `ref`: Invocación modular a otro diagrama de secuencia ya especificado para evitar duplicidad y reflejar las relaciones `<<include>>` de los casos de uso.

---

## 2. Catálogo General de Diagramas de Secuencia

| ID        | Nombre del Diagrama                                | Casos de Uso / Requisitos / Reglas                                                | Patrones de Diseño Ejercitados                                  | Fragmentos UML       |
| :-------- | :------------------------------------------------- | :-------------------------------------------------------------------------------- | :-------------------------------------------------------------- | :------------------- |
| **DS-01** | Emisión, Renderizado PDF y Envío de Factura        | `UC-12`, `UC-13` _(análogo a `UC-11`)_                                            | **Facade**, **State**, **Decorator**, **Adapter**, **Observer** | `alt`, `opt`         |
| **DS-02** | Facturación Mensual Recurrente (Proceso Batch)     | `UC-14` _(incluye `UC-12`, `UC-13`)_, `UC-16`, `RB-02`, `RB-05`, `RB-09`, `RB-11` | **Facade**, **Factory Method**, **Strategy**, **Observer**      | `loop`, `alt`, `ref` |
| **DS-03** | Conciliación de Cobros Externos con Stripe         | `UC-17`, `RB-10`, `FR-068`                                                        | **Facade**, **Adapter**, **State**, **Observer**                | `loop`, `alt`        |
| **DS-04** | Control de Vencimientos y Recordatorios de Impago  | `UC-18`, `RB-01`, `FR-062`, `FR-075`, `FR-077`                                    | **State**, **Adapter**, **Observer**                            | `loop`, `alt`, `opt` |
| **DS-05** | Aprobación de Excepciones y Rectificación Fiscal   | `UC-16`, `UC-15`, `UC-10`, `RB-06`, `RB-08` _(análogo a `UC-19`)_                 | **Factory Method**, **State**, **Observer**                     | `alt`, `opt`         |
| **DS-06** | Gestión de Incidencias, Comentarios y Adjuntos     | `UC-20`, `UC-21`, `UC-22`, `FR-077`, `NFR-25`                                     | **Observer**                                                    | `alt`, `opt`         |
| **DS-07** | Importación Masiva de Clientes desde CSV           | `UC-07`, `FR-030`                                                                 | **Observer**                                                    | `alt`, `loop`        |
| **DS-08** | Consulta y Operación desde Sistema Externo vía API | `UC-30`, `RB-09`                                                                  | **Facade**, **Observer**                                        | `alt`                |

---

## 3. Especificación Detallada por Diagrama

---

### 3.1. DS-01: Emisión, Renderizado PDF y Envío de Factura

- **Identificador:** `DS-01`
- **Casos de uso y requisitos cubiertos:** `UC-12` (Emitir factura), `UC-13` (Generar PDF y enviar factura al cliente), y por analogía estructural `UC-11` (Emisión de factura puntual).
- **Patrones de diseño ejercitados:**
  - **Facade (`FacturacionFacade`):** Orquesta en una única transacción de negocio la validación fiscal, la transición de estado, la generación documental, el envío y la publicación de eventos.
  - **State (`EstadoBorrador` $\rightarrow$ `EstadoPendiente`):** Encapsula las reglas de emisión y asignación de serie/numeración oficial, impidiendo emitir facturas con datos incompletos o ya emitidas.
  - **Decorator (`IDocumentoPDF` / `FacturaPDFBase`):** Permite añadir dinámicamente elementos visuales al documento (marca de agua, sello fiscal o anexos legales) sin alterar la clase base.
  - **Adapter (`ICanalMensajeria` $\rightarrow$ `EmailSimuladoAdapter`):** Desacopla el envío de correo electrónico del proveedor concreto.
  - **Observer (`GestorEventosDominio` $\rightarrow$ `AuditoriaObserver`, `NotificacionObserver`):** Registra la auditoría de emisión y gestiona alertas ante fallos de entrega.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor:** `Usuario Operativo`
- **Capa de Presentación y Control:** `Frontend SPA`, `ControladorFacturas`
- **Capa de Servicio y Dominio:** `FacturacionFacade`, `Cliente`, `Factura`, `EstadoBorrador`, `EstadoPendiente`, `PlantillaComunicacion`
- **Documentación y Persistencia:** `FacturaPDFBase` (`IDocumentoPDF`), `StorageSeguro`
- **Integración y Eventos:** `EmailSimuladoAdapter` (`ICanalMensajeria`), `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`

#### Precondiciones

1. El `Usuario Operativo` está autenticado y dispone de permisos operativos sobre el módulo de facturación.
2. Existe una instancia de `Factura` en estado `EstadoBorrador` vinculada a un `Cliente` registrado en el sistema.

#### Flujo Temporal y Fragmentos Combinados

1. **Solicitud de emisión:** El `Usuario Operativo` solicita emitir la factura desde `Frontend SPA`, que invoca a `ControladorFacturas` y este delega en `FacturacionFacade`.
2. **Validación fiscal previa:** `FacturacionFacade` consulta los datos fiscales obligatorios (NIF/CIF, razón social, dirección fiscal y correo de facturación) en la instancia `Cliente`.
3. **Bifurcación de validación (`alt`):**
   - **Rama de error (Datos fiscales incompletos):** Se aborta el proceso sin modificar el estado de la factura y se devuelve un error de validación detallado hasta `Frontend SPA`.
   - **Rama nominal (Datos fiscales válidos):**
     1. **Transición de estado:** `FacturacionFacade` delega la emisión en `Factura`, que a través de `EstadoBorrador` asigna la serie y numeración oficial correlativa, fija la fecha de emisión y realiza la transición de estado a `EstadoPendiente`.
     2. **Renderizado y almacenamiento documental:** Se instancia `FacturaPDFBase` (implementando `IDocumentoPDF`). Mediante un bloque **`opt`**, si la configuración o tipo de factura lo requiere, se envuelve con los decoradores correspondientes. El binario resultante se persiste en `StorageSeguro`, obteniendo la referencia segura del documento.
     3. **Composición y envío:** `PlantillaComunicacion` formatea el asunto y cuerpo del correo con los datos de la factura y adjunta el PDF. `FacturacionFacade` invoca el envío mediante `EmailSimuladoAdapter`.
     4. **Bifurcación de resultado de entrega (`alt`):**
        - **Entrega exitosa:** Se publica el evento `FACTURA_EMITIDA` en `GestorEventosDominio`, que notifica a `AuditoriaObserver` para registrar la operación en `AuditoriaLog`.
        - **Rebote o fallo de envío:** Mediante un bloque **`opt`**, si el `Cliente` dispone de un correo electrónico suplente configurado, se ejecuta un reintento de envío por `EmailSimuladoAdapter`. Si persiste el fallo, se publica el evento `FALLO_ENTREGA_FACTURA` en `GestorEventosDominio`, disparando tanto el registro en `AuditoriaObserver` como la alerta operativa/incidencia en `NotificacionObserver`.

#### Postcondiciones

- En el flujo exitoso, la `Factura` queda persistida en `EstadoPendiente` con numeración oficial, su PDF almacenado en `StorageSeguro` y su traza registrada en `AuditoriaLog`.
- En caso de fallo de envío, la `Factura` permanece emitida pero queda registrada una alerta/incidencia de entrega para su seguimiento.

---

### 3.2. DS-02: Facturación Mensual Recurrente (Proceso Batch)

- **Identificador:** `DS-02`
- **Casos de uso y requisitos cubiertos:** `UC-14` (Ejecutar facturación mensual recurrente, que incluye `UC-12` y `UC-13`), `UC-16` (Aprobación de excepciones), y reglas de negocio `RB-02` (ciclo mensual), `RB-05` (prorrateo de alta), `RB-09` (bloqueo por baja solicitada) y `RB-11` (revisión anual de tarifas).
- **Patrones de diseño ejercitados:**
  - **Facade (`FacturacionFacade`):** Expone el punto de entrada único para el procesamiento masivo de suscripciones.
  - **Strategy (`IEstrategiaCalculoTarifa` $\rightarrow$ `CalculoProrrateoAltaStrategy` / estrategia de cuota completa):** Selecciona dinámicamente el algoritmo de cálculo según la fecha de alta y las condiciones de la suscripción.
  - **Factory Method (`FacturaRecurrenteFactory`):** Encapsula la instanciación de facturas periódicas con sus líneas de concepto y metadatos de suscripción.
  - **Observer (`GestorEventosDominio`):** Notifica el resumen de ejecución del lote y las posibles anomalías detectadas.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor / Disparador Temporal:** `PlanificadorCron`
- **Procesamiento en Segundo Plano:** `WorkerFacturacion`
- **Capa de Servicio, Estrategia y Fábrica:** `FacturacionFacade`, `IEstrategiaCalculoTarifa`, `FacturaRecurrenteFactory`
- **Dominio, Persistencia y Eventos:** `BaseDatosRelacional`, `Suscripcion`, `Factura`, `GestorEventosDominio`

#### Precondiciones

1. El `PlanificadorCron` alcanza la ventana temporal programada para el cierre/emisión mensual.
2. Existen registros de `Suscripcion` en estado activo en `BaseDatosRelacional`.

#### Flujo Temporal y Fragmentos Combinados

1. **Disparo del proceso batch:** `PlanificadorCron` activa a `WorkerFacturacion`, el cual invoca la operación de facturación mensual en `FacturacionFacade`.
2. **Recuperación de suscripciones:** `FacturacionFacade` consulta a `BaseDatosRelacional` todas las instancias de `Suscripcion` candidatas a facturación en el periodo corriente.
3. **Iteración por lote (`loop` sobre cada `Suscripcion` activa):**
   1. **Evaluación de elegibilidad y reglas de negocio (`alt`):**
      - **Rama de bloqueo / excepción pendiente (`RB-09`, `RB-11` o `UC-16` sin aprobar):** Si la suscripción tiene una solicitud de baja en curso (`RB-09`) o una excepción de tarifa/revisión anual pendiente de aprobación administrativa, se omite su facturación automática y se registra en el listado de suscripciones retenidas para revisión.
      - **Rama apta para facturación:**
        1. **Cálculo de importe (`Strategy`):** `FacturacionFacade` selecciona la implementación adecuada de `IEstrategiaCalculoTarifa` (aplicando `CalculoProrrateoAltaStrategy` si el alta se produjo dentro del mes en curso según `RB-05`, o la tarifa mensual íntegra en caso contrario) y calcula la base imponible e impuestos.
        2. **Creación de factura (`Factory Method`):** `FacturacionFacade` delega en `FacturaRecurrenteFactory` la creación (`<<create>>`) de la nueva instancia `Factura` en estado borrador vinculada a la `Suscripcion`.
        3. **Reutilización modular (`ref`):** Se invoca el fragmento de referencia **`DS-01: Emisión, Renderizado PDF y Envío de Factura (UC-12 / UC-13)`** para validar fiscalmente, numerar, generar el PDF y enviar la factura recién creada.
4. **Cierre del lote:** Finalizado el bucle, `FacturacionFacade` publica el evento `LOTE_FACTURACION_COMPLETADO` en `GestorEventosDominio` con las métricas de facturas emitidas y suscripciones retenidas.

#### Postcondiciones

- Todas las suscripciones activas sin bloqueos generan su correspondiente `Factura` emitida y enviada al cliente.
- Las suscripciones con excepciones pendientes o bajas en trámite quedan retenidas sin generar cargos indebidos, y el resultado global queda auditado.

---

### 3.3. DS-03: Conciliación de Cobros Externos con Stripe

- **Identificador:** `DS-03`
- **Casos de uso y requisitos cubiertos:** `UC-17` (Conciliar cobros externos), `RB-10` (gestión de discrepancias de importe) y `FR-068` (detección de pagos parciales, sobrepagos o cobros no identificados).
- **Patrones de diseño ejercitados:**
  - **Adapter (`IConsultarCobroExterno` $\rightarrow$ `StripeSimuladoAdapter`):** Aísla la comunicación y el formato de respuesta de la pasarela externa Stripe respecto al dominio interno.
  - **State (`EstadoPendiente` / `EstadoImpagada` $\rightarrow$ `EstadoPagada`):** Garantiza que una factura pase a pagada únicamente cuando se verifica la liquidación íntegra del importe.
  - **Observer (`GestorEventosDominio` $\rightarrow$ `AuditoriaObserver`, `NotificacionObserver`):** Registra la conciliación automática o dispara alertas ante discrepancias financieras.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor / Disparador Temporal:** `PlanificadorCron`, `Stripe Simulado`
- **Procesamiento y Servicio:** `WorkerConciliacion`, `FacturacionFacade`, `StripeSimuladoAdapter` (`IConsultarCobroExterno`)
- **Dominio, Persistencia y Eventos:** `BaseDatosRelacional`, `Cobro`, `Factura`, `EstadoPendiente`, `EstadoPagada`, `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`

#### Precondiciones

1. Existen facturas pendientes de cobro (o impagadas) con referencia de pago en `BaseDatosRelacional`.
2. El servicio simulado de Stripe dispone de transacciones confirmadas pendientes de sincronizar.

#### Flujo Temporal y Fragmentos Combinados

1. **Inicio de conciliación:** `PlanificadorCron` ejecuta `WorkerConciliacion`, que solicita a `FacturacionFacade` iniciar la conciliación periódica.
2. **Consulta a pasarela externa (`Adapter`):** `FacturacionFacade` invoca a `StripeSimuladoAdapter` (a través de `IConsultarCobroExterno`), que consulta los movimientos confirmados en `Stripe Simulado` y los traduce a objetos de transferencia internos.
3. **Procesamiento de transacciones (`loop` sobre cada cobro confirmado):**
   1. **Búsqueda y verificación de idempotencia:** `FacturacionFacade` consulta `BaseDatosRelacional` para comprobar que el identificador externo del cobro no haya sido procesado previamente y localiza la `Factura` asociada.
   2. **Evaluación de cuadre financiero (`alt`):**
      - **Rama de discrepancia (`RB-10` / `FR-068` — Importe distinto al saldo pendiente o factura no identificada):** Se instancia (`<<create>>`) un registro `Cobro` marcado con `requiereAccion = true`, se persiste en `BaseDatosRelacional` sin cerrar la factura como pagada, y se publica `DISCREPANCIA_COBRO_DETECTADA` en `GestorEventosDominio` para que `NotificacionObserver` alerte al equipo administrativo.
      - **Rama de cuadre exacto:** Se instancia (`<<create>>`) el `Cobro` en estado conciliado, se guarda en `BaseDatosRelacional`, y se invoca el registro del pago sobre `Factura`, provocando la transición de estado (`EstadoPendiente` $\rightarrow$ `EstadoPagada`). Finalmente se publica `COBRO_CONCILIADO` en `GestorEventosDominio` para su registro en `AuditoriaObserver`.

#### Postcondiciones

- Los cobros exactos quedan vinculados a sus facturas, actualizando su estado a `EstadoPagada`.
- Los cobros con importe parcial, sobrecargo o sin factura asociada quedan registrados con `requiereAccion = true` y notificados para resolución manual.

---

### 3.4. DS-04: Control de Vencimientos y Recordatorios de Impago

- **Identificador:** `DS-04`
- **Casos de uso y requisitos cubiertos:** `UC-18` (Controlar vencimientos y enviar recordatorios de impago), `RB-01` (reglas de vencimiento y plazos de reclamación a 3 y 5 días), `FR-062`, `FR-075` y `FR-077`.
- **Patrones de diseño ejercitados:**
  - **State (`EstadoPendiente` $\rightarrow$ `EstadoImpagada`):** Controla la transición automática de una factura vencida hacia el estado de impago.
  - **Adapter (`ICanalMensajeria` $\rightarrow$ `EmailSimuladoAdapter`, `WhatsAppSimuladoAdapter`):** Permite el envío multicanal de recordatorios desacoplando cada canal de comunicación.
  - **Observer (`GestorEventosDominio`):** Registra los cambios a estado impagada y dispara notificaciones internas al equipo de gestión de cobros.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor / Disparador Temporal:** `PlanificadorCron`, `Proveedor Mensajería` (Email / WhatsApp)
- **Procesamiento y Configuración:** `WorkerImpagos`, `Configuracion`, `BaseDatosRelacional`
- **Dominio y Mensajería:** `Factura`, `EstadoPendiente`, `EstadoImpagada`, `PlantillaComunicacion`, `EmailSimuladoAdapter`, `WhatsAppSimuladoAdapter`
- **Eventos:** `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`

#### Precondiciones

1. Existen facturas en `EstadoPendiente` cuya fecha de vencimiento ha sido superada.
2. Los umbrales de días de gracia y reclamación están parametrizados en `Configuracion` (3 y 5 días según `RB-01`).

#### Flujo Temporal y Fragmentos Combinados

1. **Ejecución programada:** `PlanificadorCron` activa a `WorkerImpagos`.
2. **Lectura de reglas y facturas vencidas:** `WorkerImpagos` obtiene los parámetros de plazos desde `Configuracion` y recupera de `BaseDatosRelacional` las instancias de `Factura` en `EstadoPendiente` con fecha de vencimiento anterior a la fecha actual.
3. **Evaluación individual (`loop` sobre cada `Factura` vencida):**
   1. **Cálculo de días de retraso y bifurcación (`alt`):**
      - **Rama dentro del margen preventivo / primer aviso:** Si la factura supera el primer umbral configurado (p. ej., 3 días), se prepara un recordatorio amistoso sin alterar aún su estado definitivo (o preparando la reclamación).
      - **Rama de superación de umbral de impago (`RB-01` — p. ej., 5 días):** `Factura` delega en `EstadoPendiente` la transición formal hacia `EstadoImpagada`, actualizando su estado en `BaseDatosRelacional`.
   2. **Envío de comunicación al cliente:** `PlantillaComunicacion` genera el mensaje de reclamación correspondiente. Se envía obligatoriamente por correo electrónico a través de `EmailSimuladoAdapter` (`FR-075`).
   3. **Canal complementario condicional (`opt`):** Si el cliente asociado tiene activo el flag de consentimiento explícito para comunicaciones por mensajería instantánea, se invoca adicionalmente a `WhatsAppSimuladoAdapter` para enviar el aviso por WhatsApp.
   4. **Registro y alerta interna:** Se publica el evento `FACTURA_IMPAGADA` / `RECORDATORIO_IMPAGO_ENVIADO` en `GestorEventosDominio`, generando traza en `AuditoriaObserver` y alerta en `NotificacionObserver` (`FR-077`).

#### Postcondiciones

- Las facturas que superan el plazo crítico quedan marcadas en `EstadoImpagada`.
- Los clientes reciben las notificaciones por los canales autorizados y queda constancia auditada de cada aviso enviado.

---

### 3.5. DS-05: Aprobación de Excepciones y Rectificación Fiscal

- **Identificador:** `DS-05`
- **Casos de uso y requisitos cubiertos:** `UC-16` (Aprobar/rechazar excepciones de facturación), `UC-15` (Emitir factura rectificativa), `UC-10` (Gestionar baja de cliente), `RB-06` (bloqueo de modificación sobre trimestres fiscales cerrados), `RB-08` (inmutabilidad de facturas emitidas y obligación de rectificativa), y por analogía `UC-19` (Cancelación por deuda incobrable).
- **Patrones de diseño ejercitados:**
  - **Factory Method (`FacturaRectificativaFactory`):** Crea una factura rectificativa formalmente vinculada a la factura original, calculando las diferencias de importes e impuestos.
  - **State:** Gestiona tanto el estado de la `SolicitudAprobacion` (_Pendiente_ $\rightarrow$ _Aprobada_ / _Rechazada_) como los estados de `Factura` y `Cliente`.
  - **Observer (`GestorEventosDominio` $\rightarrow$ `AuditoriaObserver`, `NotificacionObserver`):** Garantiza el registro inmutable en `AuditoriaLog` de cualquier alteración sensible y notifica al usuario solicitante.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor:** `Administrador`
- **Capa de Presentación y Control:** `Frontend SPA`, `ControladorAprobaciones`
- **Dominio y Fábrica:** `Usuario`, `SolicitudAprobacion`, `Factura` (original), `FacturaRectificativaFactory`, `Factura` (rectificativa)
- **Eventos y Auditoría:** `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`

#### Precondiciones

1. Existe una `SolicitudAprobacion` en estado pendiente generada previamente por un usuario operativo o por un flujo externo (`DS-08`).
2. El `Administrador` accede a la bandeja de aprobaciones desde `Frontend SPA`.

#### Flujo Temporal y Fragmentos Combinados

1. **Resolución administrativa:** El `Administrador` revisa la solicitud en `Frontend SPA` y envía su decisión (aprobar o rechazar) a `ControladorAprobaciones`.
2. **Verificación de autorización RBAC:** `ControladorAprobaciones` invoca `Usuario.tienePermiso()` para confirmar que el actor posee rol de `Administrador`.
3. **Bifurcación principal de decisión y validez fiscal (`alt`):**
   - **Rama 1 — Permisos insuficientes o solicitud rechazada:** Si no tiene permisos se devuelve error de autorización (`403`). Si el administrador rechaza la solicitud, `SolicitudAprobacion` pasa a estado _Rechazada_, se publica `SOLICITUD_RECHAZADA` en `GestorEventosDominio` y `NotificacionObserver` informa al solicitante.
   - **Rama 2 — Aprobación bloqueada por trimestre fiscal cerrado (`RB-06`):** Si la solicitud implica una rectificación o ajuste sobre una factura perteneciente a un periodo trimestral ya cerrado contablemente, el sistema bloquea la operación, mantiene la integridad fiscal y devuelve una alerta de trimestre cerrado al `Administrador`.
   - **Rama 3 — Aprobación válida con trimestre abierto:**
     1. `SolicitudAprobacion` cambia su estado a _Aprobada_ registrando fecha y usuario aprobador.
     2. **Generación condicional de factura rectificativa (`opt` — `UC-15` / `RB-08`):** Si la operación aprobada corresponde a la rectificación de una factura ya emitida, se invoca a `FacturaRectificativaFactory` (`Factory Method`) para instanciar (`<<create>>`) una nueva `Factura` rectificativa referenciando al ID y serie de la `Factura` original. (En caso de tratarse de una excepción de tarifa `UC-16` o baja de cliente `UC-10`, se aplica la actualización autorizada sobre la entidad correspondiente).
     3. **Auditoría inmutable y notificación:** Se publica el evento `EXCEPCION_APROBADA` (o `FACTURA_RECTIFICADA`) en `GestorEventosDominio`. `AuditoriaObserver` persiste el registro inmutable en `AuditoriaLog` con el estado anterior y posterior, y `NotificacionObserver` notifica al usuario que originó la petición.

#### Postcondiciones

- Ninguna factura emitida es alterada directamente; toda corrección fiscal genera su correspondiente `Factura` rectificativa vinculada.
- Toda aprobación o rechazo queda trazado de forma inmutable en `AuditoriaLog`.

---

### 3.6. DS-06: Gestión de Incidencias, Comentarios y Adjuntos

- **Identificador:** `DS-06`
- **Casos de uso y requisitos cubiertos:** `UC-20` (Gestionar incidencias), `UC-21` (Añadir comentarios y menciones), `UC-22` (Subir documentos adjuntos), `FR-077` (notificaciones internas) y `NFR-25` (validación de formato y tamaño máximo de ficheros).
- **Patrones de diseño ejercitados:**
  - **Observer (`GestorEventosDominio` $\rightarrow$ `NotificacionObserver`):** Desacopla la creación de comentarios, menciones (`@usuario`) o cambios de incidencia de la generación y entrega individualizada de notificaciones.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor:** `Usuario Operativo`
- **Capa de Presentación y Control:** `Frontend SPA`, `ControladorIncidencias`
- **Dominio y Almacenamiento:** `Incidencia`, `Comentario`, `StorageSeguro`, `Adjunto`
- **Eventos y Notificaciones:** `GestorEventosDominio`, `NotificacionObserver`, `Notificacion`, `RecibeNotificacion`

#### Precondiciones

1. El `Usuario Operativo` está autenticado y tiene acceso al cliente, factura o incidencia sobre la que actúa.

#### Flujo Temporal y Fragmentos Combinados

1. **Envío de acción colaborativa:** Desde `Frontend SPA`, el `Usuario Operativo` registra o actualiza una `Incidencia` (`UC-20`), añade un `Comentario` con posibles menciones `@usuario` o marca de visibilidad restringida (`UC-21`), y/o selecciona un fichero para adjuntar (`UC-22`).
2. **Validación de reglas de acceso y fichero (`alt`):**
   - **Rama de error de validación (`NFR-25` o restricción de rol):** Si el fichero adjunto supera el tamaño máximo permitido o tiene una extensión no autorizada (`NFR-25`), o si un usuario sin rol de administrador intenta publicar un comentario marcado como restringido a administradores, `ControladorIncidencias` rechaza la operación y devuelve el mensaje de error a `Frontend SPA`.
   - **Rama válida:**
     1. **Persistencia de incidencia y comentario:** Se crea o actualiza la instancia de `Incidencia` y se instancia (`<<create>>`) el objeto `Comentario` vinculado.
     2. **Carga condicional de adjunto (`opt` — `UC-22`):** Si la petición incluye un documento adjunto, `ControladorIncidencias` delega el guardado físico del binario en `StorageSeguro` (obteniendo su ruta/hash seguro) e instancia (`<<create>>`) la entidad `Adjunto` asociada a la incidencia/comentario.
     3. **Publicación de evento y generación de notificaciones (`Observer` — `FR-077`):** Se publica `ACTIVIDAD_INCIDENCIA_REGISTRADA` en `GestorEventosDominio`. `NotificacionObserver` analiza los destinatarios relevantes (responsable asignado de la incidencia, administradores si es restringido, y usuarios mencionados mediante `@usuario`), instanciando (`<<create>>`) la `Notificacion` y los registros individuales de lectura `RecibeNotificacion` para cada destinatario.

#### Postcondiciones

- La incidencia, el comentario y los metadatos del adjunto quedan persistidos en base de datos, y el archivo físico queda resguardado en `StorageSeguro`.
- Cada usuario implicado o mencionado dispone de su notificación individualizada pendiente de lectura.

---

### 3.7. DS-07: Importación Masiva de Clientes desde CSV

- **Identificador:** `DS-07`
- **Casos de uso y requisitos cubiertos:** `UC-07` (Importar clientes masivamente desde fichero CSV) y `FR-030` (previsualización, validación previa de duplicados y trazabilidad de carga masiva).
- **Patrones de diseño ejercitados:**
  - **Observer (`GestorEventosDominio` $\rightarrow$ `AuditoriaObserver`):** Registra en el log de auditoría el resultado global de la importación masiva realizada por el usuario.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor:** `Usuario Operativo`
- **Capa de Presentación y Control:** `Frontend SPA`, `ControladorClientes`
- **Procesamiento Batch:** `WorkerImportacionCSV`
- **Dominio, Persistencia y Eventos:** `BaseDatosRelacional`, `Cliente`, `GestorEventosDominio`, `AuditoriaObserver`

#### Precondiciones

1. El `Usuario Operativo` dispone de un fichero `.csv` con datos de clientes y accede al asistente de importación.

#### Flujo Temporal y Fragmentos Combinados

1. **Fase 1 — Carga, mapeo y previsualización:** El `Usuario Operativo` sube el fichero CSV y define el mapeo de columnas en `Frontend SPA`. `ControladorClientes` envía el lote a `WorkerImportacionCSV` para su análisis en seco (_dry-run_).
2. **Validación estructural y detección de duplicados (`FR-030`):** `WorkerImportacionCSV` valida el formato de cada fila (NIF/CIF, email, campos obligatorios) y consulta `BaseDatosRelacional` para detectar NIFs ya existentes en el sistema, devolviendo un informe de previsualización (filas válidas, advertencias y errores).
3. **Bifurcación de confirmación (`alt`):**
   - **Rama de cancelación o fichero inválido:** Si el fichero carece de filas válidas o el `Usuario Operativo` cancela tras revisar la previsualización, se descarta el proceso temporal sin realizar escrituras en `BaseDatosRelacional`.
   - **Rama de confirmación de importación:**
     1. **Persistencia iterativa (`loop` sobre cada fila válida):** `WorkerImportacionCSV` instancia (`<<create>>`) cada objeto `Cliente` con sus datos fiscales y comerciales normalizados y lo persiste en `BaseDatosRelacional`.
     2. **Registro de auditoría:** Finalizado el bucle, se publica el evento `IMPORTACION_CSV_COMPLETADA` (incluyendo identificador de usuario, nombre de fichero y contadores de registros creados/omitidos) en `GestorEventosDominio`, que es persistido por `AuditoriaObserver`.
     3. **Retorno de resumen:** Se devuelve el acta final de importación hasta `Frontend SPA`.

#### Postcondiciones

- Todos los clientes válidos del CSV quedan dados de alta en `BaseDatosRelacional` sin duplicar NIFs existentes.
- La operación masiva queda auditada en `AuditoriaLog`.

---

### 3.8. DS-08: Consulta y Operación desde Sistema Externo vía API

- **Identificador:** `DS-08`
- **Casos de uso y requisitos cubiertos:** `UC-30` (Consulta y operación desde sistema externo vía API, p. ej. _Brain_) y `RB-09` (gestión controlada de solicitudes de baja sin eliminación directa por sistemas externos).
- **Patrones de diseño ejercitados:**
  - **Facade (`FacturacionFacade`):** Expone operaciones acotadas y seguras para consumo externo sin exponer el modelo interno completo.
  - **Observer (`GestorEventosDominio` $\rightarrow$ `AuditoriaObserver`, `NotificacionObserver`):** Audita los accesos externos y alerta al equipo interno cuando un sistema externo solicita la baja de un cliente.

#### Actores y Líneas de Vida (_Lifelines_)

- **Actor:** `Sistema Externo (ej. Brain)`
- **Perímetro API y Seguridad:** `ApiExternaRouter`, `MiddlewareApiAuth`
- **Capa de Servicio y Dominio:** `FacturacionFacade`, `BaseDatosRelacional`, `SolicitudAprobacion`
- **Eventos:** `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`

#### Precondiciones

1. El `Sistema Externo (ej. Brain)` dispone de credenciales API emitidas y realiza una petición HTTPS contra los _endpoints_ publicados por `ApiExternaRouter`.

#### Flujo Temporal y Fragmentos Combinados

1. **Recepción y autenticación perimetral:** `Sistema Externo (ej. Brain)` envía una petición HTTPS con cabeceras de autenticación (`API-Key` / Token) a `ApiExternaRouter`, que delega inmediatamente en `MiddlewareApiAuth.validarCredencialesYRateLimit(headers, origen)`.
2. **Bifurcación de seguridad y cuota (`alt`):**
   - **Rama de rechazo (`E1` Credenciales inválidas o `E2` Rate Limit excedido):** `MiddlewareApiAuth` rechaza la petición y `ApiExternaRouter` responde inmediatamente con código HTTP `401 Unauthorized` o `429 Too Many Requests`, bloqueando el acceso al núcleo.
   - **Rama autorizada:** `MiddlewareApiAuth` valida el cliente API y pasa el control al enrutador, que bifurca según el recurso invocado (`alt`):
     - **Subflujo A — Consulta de estado de deuda (`GET`):**
       1. `ApiExternaRouter` invoca `FacturacionFacade.consultarEstadoDeudaCliente(nifCliente)`.
       2. `FacturacionFacade` consulta `BaseDatosRelacional.obtenerFacturasPendientesOImpagadas(nifCliente)`.
       3. Se publica `CONSULTA_API_EXTERNA` en `GestorEventosDominio` para dejar traza de auditoría del acceso.
       4. Se devuelve `200 OK` con el resumen mínimo autorizado en formato JSON.
     - **Subflujo B — Notificación de solicitud de baja de cliente (`POST` — `RB-09`):**
       1. `ApiExternaRouter` invoca `FacturacionFacade.registrarSolicitudBajaExterna(nifCliente, motivo)`.
       2. En cumplimiento de `RB-09` (ningún sistema externo puede dar de baja directamente un cliente o suscripción), `FacturacionFacade` instancia (`<<create>>`) una `SolicitudAprobacion` con `tipo = "BAJA_CLIENTE"` y `estado = "PENDIENTE"`, y la guarda en `BaseDatosRelacional`.
       3. Se publica `SOLICITUD_BAJA_API_EXTERNA` en `GestorEventosDominio`, disparando `NotificacionObserver` para alertar al responsable interno (enlazando con el flujo de aprobación `DS-05` / `UC-10`).
       4. Se responde `202 Accepted` devolviendo el identificador del _ticket_ de solicitud registrado.

#### Postcondiciones

- Ningún acceso no autenticado o abusivo alcanza la capa de negocio.
- Las consultas externas quedan auditadas y las peticiones de baja quedan encapsuladas como `SolicitudAprobacion` pendiente de validación humana.

---

## 4. Matriz de Trazabilidad Arquitectónica

La siguiente tabla verifica la coherencia transversal entre los **Diagramas de Secuencia**, los **Casos de Uso y Requisitos (Entrega 1)**, el **Diagrama de Clases**, el **Diagrama Entidad-Relación Físico (ERD)** y el **Diagrama de Arquitectura/Componentes**:

| Diagrama  | Casos de Uso / Reglas                                | Clases e Interfaces Clave                                                                                                                                                             | Entidades ERD Afectadas                                                                  | Componentes Arquitectónicos                                                                           |
| :-------- | :--------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| **DS-01** | `UC-11`, `UC-12`, `UC-13`                            | `FacturacionFacade`, `Cliente`, `Factura`, `EstadoBorrador`, `EstadoPendiente`, `IDocumentoPDF`, `FacturaPDFBase`, `ICanalMensajeria`, `EmailSimuladoAdapter`, `GestorEventosDominio` | `CLIENTE`, `FACTURA`, `PLANTILLA_COMUNICACION`, `AUDITORIA_LOG`, `NOTIFICACION`          | `Frontend SPA`, `Backend API REST`, `BaseDatosRelacional`, `StorageSeguro`, `Servicio Email Simulado` |
| **DS-02** | `UC-14`, `UC-16`, `RB-02`, `RB-05`, `RB-09`, `RB-11` | `FacturacionFacade`, `Suscripcion`, `IEstrategiaCalculoTarifa`, `CalculoProrrateoAltaStrategy`, `FacturaRecurrenteFactory`, `Factura`, `GestorEventosDominio`                         | `SUSCRIPCION`, `TARIFA`, `FACTURA`, `SOLICITUD_APROBACION`, `AUDITORIA_LOG`              | `PlanificadorCron`, `Workers (Facturación)`, `Backend API REST`, `BaseDatosRelacional`                |
| **DS-03** | `UC-17`, `RB-10`, `FR-068`                           | `FacturacionFacade`, `IConsultarCobroExterno`, `StripeSimuladoAdapter`, `Cobro`, `Factura`, `EstadoPendiente`, `EstadoPagada`, `GestorEventosDominio`                                 | `COBRO`, `FACTURA`, `AUDITORIA_LOG`, `NOTIFICACION`                                      | `PlanificadorCron`, `Workers (Conciliación)`, `Pasarela Stripe Simulada`, `BaseDatosRelacional`       |
| **DS-04** | `UC-18`, `RB-01`, `FR-062`, `FR-075`, `FR-077`       | `Configuracion`, `Factura`, `EstadoPendiente`, `EstadoImpagada`, `PlantillaComunicacion`, `EmailSimuladoAdapter`, `WhatsAppSimuladoAdapter`, `GestorEventosDominio`                   | `CONFIGURACION`, `FACTURA`, `CLIENTE`, `NOTIFICACION`, `AUDITORIA_LOG`                   | `PlanificadorCron`, `Workers (Impagos)`, `Servicio Email/WhatsApp Simulado`, `BaseDatosRelacional`    |
| **DS-05** | `UC-10`, `UC-15`, `UC-16`, `UC-19`, `RB-06`, `RB-08` | `Usuario`, `SolicitudAprobacion`, `Factura`, `FacturaRectificativaFactory`, `GestorEventosDominio`, `AuditoriaObserver`, `NotificacionObserver`                                       | `USUARIO`, `SOLICITUD_APROBACION`, `FACTURA`, `CLIENTE`, `AUDITORIA_LOG`, `NOTIFICACION` | `Frontend SPA`, `Backend API REST`, `BaseDatosRelacional`                                             |
| **DS-06** | `UC-20`, `UC-21`, `UC-22`, `FR-077`, `NFR-25`        | `Incidencia`, `Comentario`, `Adjunto`, `GestorEventosDominio`, `NotificacionObserver`, `Notificacion`, `RecibeNotificacion`                                                           | `INCIDENCIA`, `COMENTARIO`, `ADJUNTO`, `NOTIFICACION`, `RECIBE_NOTIFICACION`             | `Frontend SPA`, `Backend API REST`, `BaseDatosRelacional`, `StorageSeguro`                            |
| **DS-07** | `UC-07`, `FR-030`                                    | `Cliente`, `GestorEventosDominio`, `AuditoriaObserver`                                                                                                                                | `CLIENTE`, `AUDITORIA_LOG`                                                               | `Frontend SPA`, `Backend API REST`, `Workers (Importación CSV)`, `BaseDatosRelacional`                |
| **DS-08** | `UC-30`, `RB-09`                                     | `ApiExternaRouter`, `MiddlewareApiAuth`, `FacturacionFacade`, `SolicitudAprobacion`, `GestorEventosDominio`                                                                           | `CLIENTE`, `FACTURA`, `SOLICITUD_APROBACION`, `AUDITORIA_LOG`, `NOTIFICACION`            | `API Externa (Gateway/Router)`, `Backend API REST`, `BaseDatosRelacional`                             |

---
