# Arquitectura y componentes

**Proyecto:** Sistema interno de facturación y cobros de Turbine
**Asignatura:** Análisis y Diseño de Aplicaciones (curso 2026-27), Entrega 2
**Artefacto de la guía operativa:** apartado 7.6, «Diagrama de arquitectura o componentes».
**Diagrama fuente:** `doc/src/arquitectura/componentes.pu` (exportado a `doc/arquitectura-componentes-diagrama.pdf`).
**Parte de:** `requisitos.md` (BR, FR, NFR, RB, A, Q) y `casos-uso-especificación.md` (UC-01 a UC-30).
**Estado:** propuesta para revisión del equipo. El tipo de diagrama, el nivel de detalle y la notación definitivos dependen de las indicaciones del profesorado (guía, 7.6).

## 1. Introducción

### 1.1. Objetivo

Este documento describe la arquitectura del sistema a partir del diagrama de componentes del equipo. Cada componente del diagrama tiene su propia descripción (sección 4), cada flecha está explicada (sección 5), y todo se relaciona con los requisitos y los casos de uso (sección 8). Los alias entre paréntesis (`UI`, `API`, `Logic`, `NotifManager`, `Workers`, `Cron`, `ORM`, `DB`, `Storage`, `StripeAdapter`, `MsgAdapter`) son los del diagrama.

### 1.2. Origen de la información

Siguiendo el apartado 1.4 de la guía, cada afirmación tiene uno de estos orígenes, indicado en la columna «Origen» de la sección 6:

| Origen       | Significado                                                            | Ejemplo                                      |
| ------------ | ---------------------------------------------------------------------- | -------------------------------------------- |
| **Cliente**  | Lo dijo Alejandro en las reuniones y está recogido en `requisitos.md`  | Sin microservicios; PDF original conservado  |
| **Análisis** | Decisión o suposición del equipo en `requisitos.md` (secciones 6 y 7)  | A-11: integraciones simuladas y encapsuladas |
| **Diseño**   | Decisión de arquitectura de este documento para satisfacer lo anterior | Cola de trabajos en la base de datos         |

## 2. Factores que condicionan la arquitectura

| Prioridad | Atributo                            | Por qué                                                                    | Requisitos                        |
| --------- | ----------------------------------- | -------------------------------------------------------------------------- | --------------------------------- |
| 1         | Integridad económica y trazabilidad | «Automatizar errores a escala sería peor que hacerlo manual»               | NFR-10, 12, 17, 21; FR-109; BR-05 |
| 2         | Seguridad por ámbito                | El operativo solo ve lo asignado, también al exportar o copiar una URL     | RB-12, NFR-16; FR-093             |
| 3         | Fiabilidad de la automatización     | Generación mensual reintentable y sin duplicados                           | NFR-09, NFR-21                    |
| 4         | Usabilidad y rendimiento            | Menos de un minuto, pocos clics, respuesta inmediata con miles de clientes | NFR-04, 05, 06                    |
| 5         | Mantenibilidad y evolución          | Un solo sitio para cada regla; integraciones sustituibles                  | NFR-08, 27, 30                    |

**Restricciones del cliente:** aplicación independiente de Brain; software libre; sin IA en el núcleo (las reglas son deterministas); integraciones simuladas; Verifactu, fiscalidad internacional y paquete para la gestoría fuera de alcance.

## 3. Visión general

El sistema es un **monolito modular en cinco bloques**: cliente, negocio, procesos en segundo plano, datos e infraestructura, y servicios externos. La capa de negocio y los procesos en segundo plano comparten el mismo código y se despliegan como dos procesos (aplicación web y worker). Dentro de `Logic` hay módulos con fronteras claras (sección 4.2.2).

| Alias           | Componente                        | Bloque                  | Responsabilidad                                                                           |
| --------------- | --------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------- |
| `UI`            | Frontend SPA (Interfaz Web)       | Capa Cliente            | Interfaz responsive para móvil, tableta y escritorio                                      |
| `API`           | API REST (Controladores y Rutas)  | Capa de Negocio         | Entrada única: autenticación, validación de entrada y traducción a casos de uso           |
| `Logic`         | Servicios de Negocio (Lógica)     | Capa de Negocio         | Reglas del dominio, permisos, estados, cálculos y auditoría                               |
| `NotifManager`  | Gestor de Eventos/Notificaciones  | Capa de Negocio         | Recoge los eventos de negocio y gestiona plantillas, mensajes, entregas y avisos internos |
| `Cron`          | Planificador (Cron/Scheduler)     | Segundo plano           | Dispara las tareas periódicas                                                             |
| `Workers`       | Workers Asíncronos                | Segundo plano           | Ejecutan los trabajos largos o programados                                                |
| `ORM`           | ORM / Pool de Conexiones          | Datos e infraestructura | Acceso a la base de datos y transacciones                                                 |
| `DB`            | Base de Datos Relacional          | Datos e infraestructura | Fuente única de verdad                                                                    |
| `Storage`       | Almacenamiento Ficheros Seguros   | Datos e infraestructura | PDFs emitidos, adjuntos e informes                                                        |
| `StripeAdapter` | Adaptador Pagos (Stripe Simulado) | Servicios externos      | Consulta y recepción de cobros simulados                                                  |
| `MsgAdapter`    | Adaptador Mensajería (Email/WA)   | Servicios externos      | Envío simulado de correo y WhatsApp y resultado de entrega                                |

## 4. Descripción de los componentes

### 4.1. Capa Cliente

#### 4.1.1. Frontend SPA (`UI`)

- **Responsabilidad:** presentar la aplicación y llamar a la API. No contiene reglas de negocio: todo lo importante se valida de nuevo en el servidor.
- **Casos de uso:** todos los que tienen interfaz; destacan UC-23 (pantalla de trabajo diario) y UC-24 (búsqueda).
- **Requisitos:** NFR-02 y NFR-24 (responsive y móvil), NFR-14 (accesibilidad con teclado y contraste), NFR-22 (textos externalizables en español e inglés), NFR-32 (interfaz consistente), NFR-04 y NFR-05 (pocos clics, respuesta inmediata), FR-119 a FR-121 (borradores, errores de campo, valores por defecto), FR-122 a FR-124 y NFR-13 (acciones masivas con confirmación).
- **Decisiones de diseño:** los listados se piden paginados al servidor y nunca se carga el histórico completo (NFR-06). Los botones de operaciones económicas se deshabilitan mientras se procesan (NFR-10). Las operaciones largas muestran su estado (pendiente, procesando, completado con errores) y siguen vivas aunque se cierre la pestaña (NFR-11). La interfaz oculta lo que no corresponde al rol, pero **no es un mecanismo de seguridad**: la autorización real está en `API` y `Logic`.

### 4.2. Capa de Negocio (Backend)

#### 4.2.1. API REST (`API`)

- **Responsabilidad:** única puerta de entrada. Autentica, comprueba que el usuario tiene sesión, valida el formato de la petición, llama al caso de uso correspondiente en `Logic` y traduce los errores a mensajes comprensibles con un código de incidencia.
- **Dos formas de acceso a los mismos servicios:**
  - **Interna:** la usa `UI` con sesión en servidor (cookie HttpOnly con protección CSRF).
  - **Externa (UC-30):** la usan otros sistemas (por ejemplo, Brain) con claves de API propias, ámbitos y límite de peticiones. Pasa por **los mismos servicios** que la interfaz, de modo que no puede saltarse reglas.
- **Casos de uso:** UC-01 (autenticarse, recuperación, límite de intentos) y UC-30.
- **Requisitos:** FR-001, FR-002, FR-005, FR-113, FR-114, NFR-03, NFR-15, NFR-19, NFR-26 (contrato documentado), NFR-28 (código de error), NFR-29.
- **Decisiones de diseño:** documentación del contrato con OpenAPI. Registro de las llamadas relevantes de la API externa.

#### 4.2.2. Servicios de Negocio (`Logic`)

- **Responsabilidad:** contiene todas las reglas del negocio y es el único componente que escribe en `DB` (mediante `ORM`) y en `Storage`. Todo lo que llega, desde `API` o desde `Workers`, pasa por aquí.
- **Estructura interna en módulos:** cada módulo es dueño de sus datos y solo se comunica con otros mediante su interfaz pública.

| Módulo                                   | Responsabilidad                                                                                                                                                                 | Casos de uso                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **Identidad y acceso**                   | Usuarios, roles, sesiones revocables, delegaciones temporales, política de autorización                                                                                         | UC-01, UC-02, UC-03               |
| **Clientes y contactos**                 | Ficha comercial y fiscal, contactos, responsable interno, estado, etiquetas, canales y consentimiento, duplicados por NIF, borrado lógico, historial, importación CSV, búsqueda | UC-03, UC-06, UC-07, UC-24        |
| **Suscripciones y tarifas**              | Suscripción, tabla de rangos versionada, excepciones comerciales, cálculo de prorrateo, pausas, revisión anual, baja y reactivación                                             | UC-08, UC-09, UC-10               |
| **Facturación**                          | Ciclo de vida de la factura, líneas, catálogo ligero, presupuestos, series y numeración, validación previa a emitir, rectificativas, cierre de periodos                         | UC-11, UC-12, UC-14, UC-15, UC-27 |
| **Documentos**                           | Generación del PDF corporativo al emitir y conservación del original, adjuntos, descargas autenticadas o con enlace temporal                                                    | UC-12, UC-21                      |
| **Cobros y conciliación**                | Registro de cobros, deduplicación de eventos, motor de coincidencias, diferencias, reversiones trazadas, incobrables                                                            | UC-17, UC-19                      |
| **Recobro**                              | Vencimientos (plazo del cliente sobre el general), recordatorios previos, flujo de impago, pausas, calendario laboral                                                           | UC-18                             |
| **Aprobaciones**                         | Solicitudes, umbrales, separación de funciones, reasignación                                                                                                                    | UC-16, UC-19                      |
| **Incidencias, comentarios y adjuntos**  | Incidencias internas, comentarios con menciones y nivel restringido                                                                                                             | UC-20, UC-21                      |
| **Informes, dashboard y trabajo diario** | Consultas agregadas, filtros, exportación CSV/PDF, objetivos, pantalla de trabajo diario                                                                                        | UC-23, UC-25, UC-26               |
| **Configuración**                        | Parámetros tipados, validados, versionados y auditados; único punto donde se define cada parámetro                                                                              | UC-04                             |
| **Auditoría y observabilidad**           | Registro de negocio solo de añadir, línea temporal del cliente, pantalla de salud, métricas agregadas                                                                           | UC-28, UC-29                      |

- **Requisitos que condicionan todo el componente:** NFR-08 (mantenibilidad), NFR-12 (precisión decimal), NFR-17 (concurrencia), NFR-23 (zonas horarias), NFR-30 (pruebas automáticas de las reglas económicas), RB-01 a RB-17.
- **Regla de dependencias:** los módulos no acceden a las tablas de otros. Se recomienda una prueba automática de arquitectura que falle si se viola.

#### 4.2.3. Gestor de Eventos/Notificaciones (`NotifManager`)

- **Responsabilidad:** recibe de `Logic` los eventos de negocio (factura emitida, vencimiento, pago, rebote, aprobación pendiente…) y decide qué comunicar y a quién, sin que los módulos de negocio conozcan los canales.
- **Qué gestiona:**
  - **Comunicaciones al cliente:** plantillas versionadas con variables e idioma, cola de mensajes, intentos y resultado de entrega. Incluye el envío de facturas (UC-13) y los recordatorios (UC-18).
  - **Notificaciones internas:** bandeja con estado, preferencias (aplicación, correo o resumen diario), alertas configurables y menciones. Distingue «leída» de «resuelta».
- **Casos de uso:** UC-05, UC-13, UC-18 (envíos), UC-22.
- **Requisitos:** FR-061 a FR-065, FR-071 a FR-075, FR-081 a FR-085, FR-106 a FR-108, FR-111, NFR-10, NFR-22.
- **Decisiones de diseño:** el mensaje se escribe en la base de datos **dentro de la misma transacción** que el hecho de negocio (patrón outbox) y se envía después. Así un fallo de correo no deshace una factura emitida y un reintento reutiliza la misma factura sin duplicarla (AD-04).

### 4.3. Procesos en Segundo Plano

#### 4.3.1. Planificador (`Cron`)

- **Responsabilidad:** dispara las tareas periódicas: generación de mensualidades el día 1, vencimientos y recordatorios diarios, avisos de reactivación, informes programados y revisiones anuales.
- **Casos de uso:** UC-14, UC-18, UC-25 (programados), UC-09 y UC-08 (avisos).
- **Requisitos:** FR-040, FR-041, FR-072, FR-074, FR-111, NFR-09, NFR-21, NFR-23.
- **Decisiones de diseño:** el planificador solo **encola** trabajos con una clave de ejecución; no hace el trabajo. Al arrancar, compara los periodos que debieron ejecutarse con los que constan en el registro de ejecución y encola los que falten (recuperación tras una caída a las 00:00).

#### 4.3.2. Workers Asíncronos (`Workers`)

- **Responsabilidad:** toman trabajos de la cola, los ejecutan llamando a `Logic` y registran su resultado. Cubren la facturación recurrente, la importación CSV, los informes y exportaciones pesadas y la generación masiva de PDFs.
- **Casos de uso:** UC-14, UC-07, UC-25, UC-18.
- **Requisitos:** FR-027 a FR-030, FR-110, NFR-11, NFR-21.
- **Decisiones de diseño:**
  - Cada elemento (por ejemplo, cada cliente) se procesa en su propia transacción. Si uno falla, las otras 999 se completan y el fallo queda registrado y notificado.
  - Cada ejecución guarda cuántos elementos procesó, cuántos creó y cuántos fallaron, con el motivo.
  - La cola vive en la base de datos y un trabajo se toma con bloqueo por fila, de modo que dos workers no cogen el mismo.

### 4.4. Capa de Datos e Infraestructura

#### 4.4.1. ORM / Pool de Conexiones (`ORM`)

- **Responsabilidad:** traduce el modelo del dominio a la base de datos, gestiona las transacciones y el pool de conexiones, y aplica la concurrencia optimista (columna `versión`).
- **Requisitos:** NFR-17, NFR-06, NFR-12.
- **Decisiones de diseño:** el filtro de ámbito (clientes asignados) se aplica aquí a todas las consultas, de modo que ninguna pantalla, informe, exportación ni API pueda omitirlo. Es el punto donde se garantiza que un usuario operativo no obtiene datos de clientes ajenos.

#### 4.4.2. Base de Datos Relacional (`DB`)

- **Responsabilidad:** fuente única de verdad de clientes, suscripciones, facturas, cobros, configuración y auditoría.
- **Requisitos:** NFR-12 (importes con tipo decimal exacto), NFR-17, NFR-18 (copias y restauración), NFR-06 (índices y paginación).
- **Decisiones de diseño:** las restricciones únicas garantizan la idempotencia (sección 7.2). La tabla de auditoría no permite `UPDATE` ni `DELETE` al usuario de la aplicación. Las tablas de datos personales de contacto están separadas del documento contable (NFR-07).

#### 4.4.3. Almacenamiento de Ficheros Seguros (`Storage`)

- **Responsabilidad:** guarda los PDFs emitidos, los adjuntos y los informes exportados, fuera de cualquier ruta pública.
- **Casos de uso:** UC-12, UC-21, UC-25.
- **Requisitos:** NFR-01 (se conserva el PDF original aunque cambie la plantilla), NFR-16 (sin URLs públicas adivinables), NFR-25 (formatos y tamaños limitados; nada se ejecuta), NFR-18, FR-102 (retención de informes distinta de la de documentos contables).
- **Decisiones de diseño:** descarga por endpoint autenticado o enlace firmado de corta duración. El acceso sigue los permisos del cliente o la factura asociados (FR-126). Los documentos contables y los informes temporales tienen políticas de conservación distintas.

### 4.5. Servicios Externos (encapsulados y simulados)

#### 4.5.1. Adaptador de Pagos, Stripe simulado (`StripeAdapter`)

- **Responsabilidad:** consultar los cobros y recibir notificaciones de pago simuladas, convertidos a un formato común (importe, fecha, moneda, concepto, referencia, identificador externo).
- **Casos de uso:** UC-17.
- **Requisitos:** FR-066 a FR-069, FR-067 (integración simulada), RB-17 (no es pasarela de pago), NFR-27, A-11.
- **Decisiones de diseño:** `Logic` trabaja contra una interfaz de pagos, no contra Stripe. Se podrá añadir otra fuente (Revolut o banco) sin tocar el modelo de facturas. El identificador externo del evento es único, así un evento duplicado no se contabiliza dos veces.

#### 4.5.2. Adaptador de Mensajería, Email/WhatsApp (`MsgAdapter`)

- **Responsabilidad:** enviar correo y WhatsApp (simulados) y devolver el resultado (aceptado, entregado, fallido, rebotado).
- **Casos de uso:** UC-13, UC-18.
- **Requisitos:** FR-061 a FR-065, NFR-27.
- **Decisiones de diseño:** no se inventa el estado «leído» si el proveedor no lo da. WhatsApp solo se usa si el contacto tiene consentimiento registrado (FR-023).

## 5. Relaciones del diagrama

| N.º | Relación                      | Etiqueta en el diagrama              | Qué circula                                                                | Casos de uso y requisitos     |
| --- | ----------------------------- | ------------------------------------ | -------------------------------------------------------------------------- | ----------------------------- |
| R1  | `UI` → `API`                  | Peticiones HTTP/REST (JSON)          | Peticiones del usuario por HTTPS y respuestas en JSON                      | Todos; NFR-19, NFR-02         |
| R2  | `API` → `Logic`               | —                                    | Llamada al caso de uso con la identidad del usuario ya autenticada         | Todos; RB-12, NFR-03          |
| R3  | `Logic` → `ORM`               | —                                    | Lecturas y escrituras dentro de transacciones                              | NFR-17, NFR-12                |
| R4  | `ORM` → `DB`                  | —                                    | Sentencias SQL, bloqueo de la fila de la serie al numerar                  | FR-051, RB-07, NFR-10         |
| R5  | `Logic` → `NotifManager`      | —                                    | Eventos de negocio: emitida, vencida, pagada, rebote, aprobación pendiente | FR-081, UC-13, UC-18, UC-22   |
| R6  | `Cron` → `Workers`            | Dispara tareas (Día 1, Vencimientos) | Trabajos encolados con su clave de ejecución                               | UC-14, UC-18; NFR-21          |
| R7  | `Workers` → `Logic`           | Ejecuta facturación/importación CSV  | Llamadas a los mismos casos de uso que usa la API, sin atajos              | UC-14, UC-07, UC-25; FR-110   |
| R8  | `Logic` → `Storage`           | Guarda/Lee documentos                | PDFs emitidos, adjuntos e informes                                         | UC-12, UC-21, UC-25; NFR-16   |
| R9  | `NotifManager` → `MsgAdapter` | Envía recordatorios (3, 5 días)      | Mensajes a enviar y resultado de entrega                                   | UC-13, UC-18; FR-071 a FR-074 |
| R10 | `Logic` → `StripeAdapter`     | Gestiona cobros (Simulado)           | Consultas de cobros y eventos de pago                                      | UC-17; FR-067, FR-068         |

Notas sobre las relaciones:

- **R2 y R7 son la misma entrada** a las reglas del negocio: ni la API ni los workers pueden saltarse la autorización, las aprobaciones ni la auditoría.
- **R9:** los días 3 y 5 son los valores por defecto. Son parámetros configurables por el superadministrador y el plazo pactado de un cliente prevalece sobre ellos (FR-072, FR-073, RB-01).
- **R5 y R9 cubren también el envío de facturas** (UC-13), no solo los recordatorios: toda comunicación saliente pasa por `NotifManager`.

## 6. Decisiones de arquitectura

| ID    | Decisión                                                                                                                                               | Componentes                           | Origen             | Requisitos / casos de uso                      | Alternativa descartada                  | Justificación                                                                                                                                        |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------- | ------------------ | ---------------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| AD-01 | Monolito modular con módulos que se comunican por interfaces públicas                                                                                  | `API`, `Logic`, `Workers`             | Cliente + Diseño   | NFR-08, BR-11                                  | Microservicios                          | Un equipo pequeño y un dominio acoplado por transacciones. Los servicios distribuidos añadirían consistencia eventual justo donde se exige exactitud |
| AD-02 | Base relacional como única fuente de verdad con importes de tipo decimal exacto                                                                        | `ORM`, `DB`                           | Diseño             | NFR-12, NFR-06                                 | Base no relacional; `float`             | Transacciones, restricciones únicas y consultas de informes sobre los mismos datos                                                                   |
| AD-03 | Numeración al emitir: en la misma transacción se bloquea la fila de la serie, se incrementa el contador y se marca la factura emitida                  | `Logic`, `ORM`, `DB`                  | Diseño             | FR-051, RB-07, NFR-17; UC-12                   | Secuencia de la base de datos           | Una secuencia deja huecos si la transacción falla y no serializa dos emisiones simultáneas                                                           |
| AD-04 | Emitir y enviar son operaciones separadas: la emisión escribe el mensaje en la base de datos en su misma transacción y `NotifManager` lo envía después | `Logic`, `NotifManager`, `MsgAdapter` | Cliente + Diseño   | FR-061, FR-063, FR-064, NFR-10; UC-12, UC-13   | Enviar dentro de la emisión             | Si el correo falla, la factura existe y se reintenta el mismo envío sin crear otra factura ni otro número                                            |
| AD-05 | Cola de trabajos en la base de datos con bloqueo por fila e intentos                                                                                   | `Cron`, `Workers`, `DB`               | Diseño             | NFR-11, NFR-21; UC-14, UC-25                   | Broker externo (RabbitMQ, Redis)        | Menos piezas que desplegar; basta para el volumen                                                                                                    |
| AD-06 | Tres máquinas de estado independientes en la factura: documento, entrega y cobro; el estado de cobro se deriva                                         | `Logic` (Facturación, Cobros)         | Cliente + Análisis | FR-066, A-14, RB-10; UC-17, UC-18              | Un único campo «estado»                 | Evita mezclar «emitida», «enviada» y «pagada». «Impagado» no es una etiqueta manual                                                                  |
| AD-07 | Instantáneas inmutables: líneas, precios, impuestos, destinatarios y PDF se copian a la factura al emitir                                              | `Logic`, `Storage`                    | Cliente            | NFR-01, RB-06, RB-13, FR-034; UC-12, UC-15     | Referencias vivas a tarifa o plantilla  | Cambiar una tarifa, el catálogo, el logo o un contacto no reescribe una factura emitida                                                              |
| AD-08 | Autorización centralizada: una política de acceso más un filtro de ámbito aplicado a todas las consultas                                               | `API`, `Logic`, `ORM`                 | Cliente            | RB-12, FR-013, FR-093, FR-126; UC-24           | Comprobaciones por rol en cada pantalla | El cliente pidió un modelo coherente; una URL copiada debe rechazarse en el servidor                                                                 |
| AD-09 | Auditoría escrita en la misma transacción que el cambio, en una tabla sin `UPDATE` ni `DELETE` para la aplicación                                      | `Logic`, `DB`                         | Cliente            | FR-109, FR-016, FR-101; UC-28                  | Solo registros de la aplicación         | No hay cambio sin rastro y ni el superadministrador reescribe el pasado                                                                              |
| AD-10 | Concurrencia optimista con columna `versión`                                                                                                           | `ORM`, `DB`                           | Diseño             | NFR-17; UC-12                                  | Bloqueo pesimista de pantalla           | No sobrescribir en silencio sin dejar bloqueos colgados                                                                                              |
| AD-11 | Informes y dashboard como consultas sobre los mismos datos, con detalle al pulsar una cifra                                                            | `Logic` (Informes), `DB`              | Cliente            | FR-094 a FR-099; UC-26                         | Almacén de datos aparte                 | La cifra se reconcilia con las filas que la suman                                                                                                    |
| AD-12 | Una única API REST con acceso interno (sesión) y acceso externo (claves de API) sobre los mismos servicios                                             | `API`, `Logic`                        | Cliente + Diseño   | FR-113, NFR-26, NFR-27; UC-30                  | Dos lógicas distintas                   | Las reglas no se saltan por la API                                                                                                                   |
| AD-13 | Integraciones tras interfaces propias del dominio, sustituibles                                                                                        | `StripeAdapter`, `MsgAdapter`         | Cliente + Análisis | NFR-27, A-11, FR-067                           | Llamar directamente a un proveedor      | Sustituir un proveedor solo exige tocar el adaptador                                                                                                 |
| AD-14 | Pila de referencia, solo orientativa y prescindible en la entrega: TypeScript o Java para el backend, React para la SPA, PostgreSQL, Docker Compose    | Todos                                 | Diseño (opcional)  | Sin requisito; el cliente no impone tecnología | —                                       | Solo software libre, multidispositivo y desplegable. El diseño no depende de esta elección                                                           |

## 7. Conceptos transversales

### 7.1. Ciclo de vida de la factura (`Logic`: Facturación y Cobros)

- **Documento:** borrador → pendiente de aprobación → lista para emitir → emitida. Una emitida solo cambia mediante un documento rectificativo vinculado, o queda anulada cuando proceda. Los borradores nunca emitidos pueden borrarse dejando traza.
- **Entrega:** en cola → enviada → entregada / fallida / rebotada, con distinción entre envío original y reintentos.
- **Cobro (derivado):** pendiente → vencida → pagada, o incobrable por decisión aprobada. «Pagada» existe solo cuando las conciliaciones activas cubren el total. Una diferencia se muestra como pendiente de revisión, no como pago parcial comercial.
- **Seguimiento pausado** (disputa) es un atributo con motivo y fecha de revisión, no un estado de cobro.

### 7.2. Idempotencia y concurrencia

| Operación                          | Mecanismo                                                                             | Componentes                    |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------ |
| Mensualidad                        | Restricción única (suscripción, periodo de facturación)                               | `Workers`, `Logic`, `DB`       |
| Emisión por doble clic o reintento | Clave de idempotencia por solicitud y comprobación de estado dentro de la transacción | `API`, `Logic`, `DB`           |
| Evento de pago duplicado           | Restricción única (fuente, identificador del evento)                                  | `StripeAdapter`, `Logic`, `DB` |
| Reintento de envío                 | El mensaje referencia la misma factura; los intentos se acumulan                      | `NotifManager`, `MsgAdapter`   |
| Edición simultánea                 | Columna `versión`                                                                     | `ORM`, `DB`                    |
| Trabajos                           | Clave por trabajo; bloqueo por fila al tomarlo                                        | `Cron`, `Workers`, `DB`        |

### 7.3. Autorización (`API`, `Logic`, `ORM`)

- Tres roles (superadministrador ⊇ administrador ⊇ operativo) con **permisos definidos como datos**, no como código. Si se aclara quién configura las series (véase 10.1), basta con cambiar la matriz.
- **Ámbito:** un operativo ve los clientes asignados, y los delegados durante el intervalo vigente. El filtro se aplica en las consultas de todo módulo: listados, búsqueda, informes, exportación, descargas y API.
- Las reglas de separación de funciones (no aprobar lo propio) son políticas, no pantallas.
- Los accesos por URL directa se comprueban siempre en el servidor. «Recientes» y «favoritos» son comodidades, no autorización.

### 7.4. Dinero, tiempo y datos personales

- **Dinero:** objeto de valor con cantidad decimal y moneda; importe original separado del convertido para informes. Regla de redondeo única y documentada (propuesta: por línea, medio hacia arriba, y el total es la suma de líneas).
- **Tiempo:** instantes en UTC; fechas de negocio (fecha de factura, vencimiento, día 1) como fechas en la zona del emisor. El calendario laboral se resuelve en un servicio de `Logic`. Un reloj inyectable permite probar prorrateo y vencimientos.
- **RGPD:** los datos personales de contacto viven separados del documento contable y el PDF emitido se conserva. La anonimización conserva la trazabilidad. Para las copias (Q-09, pendiente) se propone un registro de supresiones que se reaplica tras cada restauración.
- **Seguridad:** TLS, contraseñas con hash adaptativo, secretos fuera del repositorio, adjuntos validados por tipo y tamaño y nunca ejecutados.
- **Errores:** cada error técnico genera un identificador y una traza solo en servidor; el usuario ve un mensaje comprensible con ese código (NFR-28).

### 7.5. Copias de seguridad

Una tarea programada copia base de datos y documentos a un destino distinto del servidor. La restauración se prueba en un entorno aparte y queda documentada. La fecha del último backup correcto alimenta la pantalla de salud (NFR-18, FR-112).

## 8. Trazabilidad

### 8.1. Requisitos no funcionales → componentes

| NFR                            | Componentes                                 | Táctica                                                                         |
| ------------------------------ | ------------------------------------------- | ------------------------------------------------------------------------------- |
| NFR-01                         | `Logic` (Documentos), `Storage`             | Plantilla PDF corporativa; PDF original conservado                              |
| NFR-02, NFR-24                 | `UI`, `API`                                 | SPA responsive; móvil prioriza consulta, aprobación, impagos e incidencias      |
| NFR-03, NFR-15, NFR-19, NFR-29 | `API`, `Logic` (Identidad y acceso)         | Sesión en servidor revocable, TLS, límite de intentos                           |
| NFR-04, NFR-05, NFR-06         | `UI`, `API`, `ORM`, `DB`                    | Paginación en servidor, índices, valores por defecto                            |
| NFR-07                         | `Logic`, `DB`, `Storage`                    | Datos personales separados del documento contable; anonimización                |
| NFR-08, NFR-30                 | `Logic`                                     | Módulos con fronteras; Configuración única; pruebas del dominio                 |
| NFR-09, NFR-21                 | `Cron`, `Workers`, `Logic`, `DB`            | Recuperación del planificador; mensualidad única por restricción; fallo aislado |
| NFR-10, NFR-17                 | `UI`, `Logic`, `ORM`, `DB`, `NotifManager`  | Idempotencia, outbox, concurrencia optimista, numeración bloqueada              |
| NFR-11                         | `Workers`, `UI`                             | Cola de trabajos con estado visible                                             |
| NFR-12                         | `Logic`, `DB`                               | Tipo decimal exacto y objeto «dinero»                                           |
| NFR-13, NFR-14, NFR-32         | `UI`                                        | Confirmación de acciones masivas, teclado, contraste, consistencia              |
| NFR-16, NFR-25                 | `Storage`, `API`                            | Descargas autenticadas; validación de adjuntos                                  |
| NFR-18                         | `DB`, `Storage`                             | Tarea de copias y restauración probada                                          |
| NFR-20                         | `API`, `Workers`, `DB`                      | Procesos sin estado, escalables por réplica                                     |
| NFR-22, NFR-23                 | `UI`, `NotifManager`, `Logic`               | Textos externalizados; plantillas por idioma; UTC más fecha de negocio          |
| NFR-26, NFR-27                 | `API`, `StripeAdapter`, `MsgAdapter`        | OpenAPI; adaptadores sustituibles                                               |
| NFR-28, NFR-33                 | `API`, `Logic` (Auditoría y observabilidad) | Identificador de error; métricas agregadas sin ranking de personas              |
| NFR-31                         | `Logic`, `DB`                               | Carga de datos ficticios en entornos no productivos                             |

### 8.2. Casos de uso → componentes

Todos los casos de uso atraviesan `UI` → `API` → `Logic` → `ORM` → `DB`; la tabla muestra los componentes específicos de cada uno.

| UC    | Caso de uso                                 | Componentes específicos                                          |
| ----- | ------------------------------------------- | ---------------------------------------------------------------- |
| UC-01 | Autenticarse                                | Logic (Identidad y acceso); API                                  |
| UC-02 | Gestionar usuarios y roles                  | Logic (Identidad y acceso)                                       |
| UC-03 | Gestionar asignación de clientes            | Logic (Identidad y acceso, Clientes y contactos)                 |
| UC-04 | Configurar el sistema                       | Logic (Configuración)                                            |
| UC-05 | Gestionar plantillas de comunicación        | NotifManager                                                     |
| UC-06 | Gestionar clientes                          | Logic (Clientes y contactos)                                     |
| UC-07 | Importar clientes desde CSV                 | Logic (Clientes y contactos); Workers                            |
| UC-08 | Gestionar suscripción                       | Logic (Suscripciones y tarifas)                                  |
| UC-09 | Revisar rango de tarifa anual               | Logic (Suscripciones y tarifas)                                  |
| UC-10 | Gestionar baja y reactivación de cliente    | Logic (Suscripciones y tarifas)                                  |
| UC-11 | Gestionar facturas puntuales y presupuestos | Logic (Facturación)                                              |
| UC-12 | Emitir factura                              | Logic (Facturación, Documentos); Storage; NotifManager           |
| UC-13 | Enviar factura y gestionar entregas         | NotifManager; MsgAdapter                                         |
| UC-14 | Generar facturas recurrentes                | Logic (Facturación); Cron; Workers                               |
| UC-15 | Rectificar factura                          | Logic (Facturación)                                              |
| UC-16 | Gestionar aprobaciones                      | Logic (Aprobaciones)                                             |
| UC-17 | Conciliar cobros                            | Logic (Cobros y conciliación); StripeAdapter                     |
| UC-18 | Gestionar recordatorios e impagos           | Logic (Recobro); Cron; Workers; NotifManager; MsgAdapter         |
| UC-19 | Cancelar deuda incobrable                   | Logic (Cobros y conciliación, Aprobaciones)                      |
| UC-20 | Gestionar incidencias de facturación        | Logic (Incidencias, comentarios y adjuntos)                      |
| UC-21 | Gestionar comentarios y adjuntos            | Logic (Incidencias, comentarios y adjuntos, Documentos); Storage |
| UC-22 | Gestionar notificaciones y alertas          | NotifManager                                                     |
| UC-23 | Consultar pantalla de trabajo diario        | Logic (Informes, dashboard y trabajo diario)                     |
| UC-24 | Buscar y filtrar                            | Logic (Clientes y contactos)                                     |
| UC-25 | Generar informes y exportar datos           | Logic (Informes, dashboard y trabajo diario); Workers            |
| UC-26 | Consultar dashboard financiero              | Logic (Informes, dashboard y trabajo diario)                     |
| UC-27 | Cerrar periodos                             | Logic (Facturación)                                              |
| UC-28 | Consultar auditoría                         | Logic (Auditoría y observabilidad)                               |
| UC-29 | Monitorizar el sistema                      | Logic (Auditoría y observabilidad)                               |
| UC-30 | Consultar y operar vía API                  | API                                                              |

Las dos relaciones «include» de la especificación se reflejan así: `Cron` y `Workers` llaman a Facturación (UC-14 incluye UC-12), y la emisión genera un evento que `NotifManager` envía mediante `MsgAdapter` (UC-12 incluye UC-13).

### 8.3. Requisitos funcionales → casos de uso → componentes

Generada a partir de la línea «Requisitos» de cada caso de uso en `casos-uso-especificación.md`. El texto del requisito está abreviado; el completo está en `requisitos.md`.

| FR     | Requisito (abreviado)                                                                                      | Casos de uso               | Componentes                                                                                                          |
| ------ | ---------------------------------------------------------------------------------------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| FR-001 | Permitir iniciar sesión mediante correo electrónico y contraseña                                           | UC-01                      | Logic (Identidad y acceso); API                                                                                      |
| FR-002 | Exigir que el correo electrónico de acceso pertenezca al dominio corporativo de Turbine y rechazar los de… | UC-01, UC-04               | Logic (Identidad y acceso, Configuración); API                                                                       |
| FR-003 | Permitir al superadministrador crear usuarios e invitarlos                                                 | UC-02                      | Logic (Identidad y acceso)                                                                                           |
| FR-004 | Permitir al superadministrador desactivar cuentas de usuario                                               | UC-02                      | Logic (Identidad y acceso)                                                                                           |
| FR-005 | Permitir recuperar la contraseña mediante enlaces temporales                                               | UC-01                      | Logic (Identidad y acceso); API                                                                                      |
| FR-006 | Distinguir tres roles de usuario: superadministrador, con prácticamente acceso total a la aplicación; adm… | UC-02                      | Logic (Identidad y acceso)                                                                                           |
| FR-007 | Permitir asignar roles y permisos a los usuarios                                                           | UC-02                      | Logic (Identidad y acceso)                                                                                           |
| FR-008 | Permitir que un usuario operativo tenga asignados clientes concretos                                       | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-009 | Permitir al usuario operativo ver y modificar la información operativa de los clientes que tiene asignado… | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-010 | Permitir al usuario operativo crear facturas de sus clientes asignados cuando su permiso lo permita        | UC-11                      | Logic (Facturación)                                                                                                  |
| FR-011 | Permitir al usuario operativo seguir los cobros de sus clientes asignados                                  | UC-17, UC-23 (\*)          | Logic (Cobros y conciliación, Informes, dashboard y trabajo diario); StripeAdapter                                   |
| FR-012 | Permitir al usuario operativo descargar los documentos de sus clientes asignados                           | UC-21                      | Logic (Incidencias, comentarios y adjuntos, Documentos); Storage                                                     |
| FR-013 | No debe permitir al usuario operativo darse a sí mismo acceso a otro cliente, cambiar tarifas globales, c… | UC-02, UC-03               | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-014 | Permitir al administrador asignar y desasignar clientes a los usuarios                                     | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-015 | Cuando cambie la asignación de un cliente, el sistema deberá transferir las tareas pendientes al nuevo re… | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-016 | Cuando cambie la asignación de un cliente, el histórico deberá seguir indicando quién realizó cada acción… | UC-03, UC-28               | Logic (Identidad y acceso, Clientes y contactos, Auditoría y observabilidad)                                         |
| FR-017 | Permitir registrar los datos de un cliente necesarios para la facturación (nombre, NIF, correo, teléfono,… | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-018 | Al crear un cliente, el sistema deberá comprobar que su creación es posible para evitar duplicados y conf… | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-019 | Detectar posibles clientes duplicados por identificador fiscal y avisar al usuario, también durante la im… | UC-06, UC-07               | Logic (Clientes y contactos); Workers                                                                                |
| FR-020 | No debe fusionar empresas automáticamente                                                                  | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-021 | Al registrar el NIF, el sistema deberá permitir especificar el país y la moneda del cliente                | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-022 | El impuesto establecido según el país deberá poder configurarse o seleccionarse                            | UC-04, UC-06               | Logic (Configuración, Clientes y contactos)                                                                          |
| FR-023 | Registrar, por cada cliente, los canales de comunicación permitidos                                        | UC-06, UC-13               | Logic (Clientes y contactos); NotifManager; MsgAdapter                                                               |
| FR-024 | Representar el estado de cada cliente en relación con la facturación                                       | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-025 | Mantener un historial de eventos relevantes asociados y relacionados con cada cliente                      | UC-06, UC-28               | Logic (Clientes y contactos, Auditoría y observabilidad)                                                             |
| FR-026 | Asociar a cada cliente un responsable interno encargado de su facturación (véase A-04)                     | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-027 | Permitir importar clientes desde un fichero CSV, mostrando una previsualización antes de que el usuario c… | UC-07                      | Logic (Clientes y contactos); Workers                                                                                |
| FR-028 | Permitir al usuario mapear las columnas del fichero CSV                                                    | UC-07                      | Logic (Clientes y contactos); Workers                                                                                |
| FR-029 | Mostrar al usuario los errores detectados en el fichero CSV y permitirle decidir si corrige el fichero o … | UC-07                      | Logic (Clientes y contactos); Workers                                                                                |
| FR-030 | Registrar la importación completa como una operación, con un resumen de clientes creados, actualizados, r… | UC-07                      | Logic (Clientes y contactos); Workers                                                                                |
| FR-031 | Permitir configurar el rango de cobro del cliente (de anual a mensual, de mensual a anual, etc.; véase Q-… | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-032 | Avisar de la revisión necesaria cuando se produzca un cambio de rango                                      | UC-09                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-033 | Permitir registrar excepciones al precio, como descuentos, modalidades gratuitas o precio pactado          | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-034 | Cuando el responsable apruebe el cambio de rango, el sistema deberá registrar quién lo aprobó, cuándo y c… | UC-09                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-035 | Poder listar los clientes cuya facturación anual debe revisarse                                            | UC-09                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-036 | Ante un cambio de rango, un responsable deberá revisarlo y aprobarlo                                       | UC-09, UC-16               | Logic (Suscripciones y tarifas, Aprobaciones)                                                                        |
| FR-037 | Permitir pausar temporalmente una suscripción, registrando el motivo, la fecha de inicio y la fecha previ… | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-038 | Durante la pausa, el sistema no deberá generar las mensualidades que correspondan, pero el cliente deberá… | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-039 | Cuando llegue la fecha prevista de reactivación, el sistema deberá avisar al responsable y no asumir sile… | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-040 | Tener en cuenta las mensualidades desde el día 1 de cada mes                                               | UC-08, UC-14               | Logic (Suscripciones y tarifas, Facturación); Cron; Workers                                                          |
| FR-041 | Si un cliente entra un día distinto del 1, el sistema deberá generar la parte proporcional hasta final de… | UC-08, UC-14               | Logic (Suscripciones y tarifas, Facturación); Cron; Workers                                                          |
| FR-042 | El cálculo del prorrateo deberá ser visible para el usuario                                                | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-043 | De forma excepcional y debido a acuerdos, el sistema deberá permitir no cobrar el prorrateo, expresando e… | UC-08                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-044 | Permitir gestionar facturas recurrentes (mensualidades) y facturas puntuales, y generar automáticamente l… | UC-11, UC-14               | Logic (Facturación); Cron; Workers                                                                                   |
| FR-045 | Cada factura puntual deberá poder incluir líneas con concepto, cantidad, precio unitario, descuento (si a… | UC-11                      | Logic (Facturación)                                                                                                  |
| FR-046 | Permitir buscar conceptos frecuentes al contratar un servicio, con un precio orientativo y un impuesto po… | UC-11                      | Logic (Facturación)                                                                                                  |
| FR-047 | Permitir corregir una factura antes de enviarla                                                            | UC-11                      | Logic (Facturación)                                                                                                  |
| FR-048 | Una vez enviada una factura, el sistema deberá registrar de forma trazable cualquier cambio relevante      | UC-15                      | Logic (Facturación)                                                                                                  |
| FR-049 | Cuando una factura no pueda modificarse, el sistema deberá permitir realizar una factura rectificativa (n… | UC-15                      | Logic (Facturación)                                                                                                  |
| FR-050 | La generación de un documento rectificativo requerirá permiso administrativo y aprobación dependiendo del… | UC-15, UC-16               | Logic (Facturación, Aprobaciones)                                                                                    |
| FR-051 | Asignar a cada factura un identificador único compuesto por número y serie, una vez que ésta se ha emitid… | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-052 | El administrador deberá poder configurar la serie de las facturas (véase Q-06)                             | UC-04, UC-12               | Logic (Configuración, Facturación, Documentos); Storage; NotifManager                                                |
| FR-053 | Generar una vista previa de la factura antes de emitirla                                                   | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-054 | Bloquear la emisión de una factura cuando falten datos imprescindibles o se apliquen importes absurdos (v… | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-055 | Avisar al usuario de toda situación de bloqueo o incidencia detectada antes de emitir una factura          | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-056 | Poder representar las facturas mediante un documento PDF                                                   | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-057 | El PDF de la factura deberá incluir los datos fiscales, la numeración, las fechas, las líneas, las bases,… | UC-12                      | Logic (Facturación, Documentos); Storage; NotifManager                                                               |
| FR-058 | Permitir crear presupuestos antes de facturas puntuales grandes y convertirlos en factura una vez aceptad… | UC-11                      | Logic (Facturación)                                                                                                  |
| FR-059 | Permitir a los administradores marcar periodos administrativos como cerrados después de revisar un mes o … | UC-27                      | Logic (Facturación)                                                                                                  |
| FR-060 | Bloquear la modificación de facturas cuando el trimestre o periodo administrativo esté cerrado, impidiend… | UC-15, UC-27               | Logic (Facturación)                                                                                                  |
| FR-061 | Enviar automáticamente las facturas, por defecto por correo electrónico                                    | UC-13, UC-14               | Logic (Facturación); NotifManager; MsgAdapter; Cron; Workers                                                         |
| FR-062 | Poder enviar las facturas por WhatsApp como canal complementario, cuando sea posible y se tenga consentim… | UC-13                      | NotifManager; MsgAdapter                                                                                             |
| FR-063 | Cuando surjan problemas al enviar la factura por correo, el sistema deberá notificar la incidencia         | UC-13                      | NotifManager; MsgAdapter                                                                                             |
| FR-064 | Cuando surjan problemas al enviar la factura por correo, el sistema deberá intentar el reenvío y registra… | UC-13                      | NotifManager; MsgAdapter                                                                                             |
| FR-065 | Mostrar el estado de los mensajes de correo y WhatsApp: en cola, enviados, entregados (si el proveedor lo… | UC-13                      | NotifManager; MsgAdapter                                                                                             |
| FR-066 | Registrar los cobros y el estado de cada factura (pendiente, pagada, impagada)                             | UC-17                      | Logic (Cobros y conciliación); StripeAdapter                                                                         |
| FR-067 | Ofrecer una integración simulada con Stripe que se comporte como si recibiera eventos de pago, que permit… | UC-17                      | Logic (Cobros y conciliación); StripeAdapter                                                                         |
| FR-068 | Al recibir el importe de una factura, el sistema deberá comprobar que se ha recibido el total y, en caso … | UC-17                      | Logic (Cobros y conciliación); StripeAdapter                                                                         |
| FR-069 | Registrar la conclusión de la revisión de una diferencia de importe                                        | UC-17                      | Logic (Cobros y conciliación); StripeAdapter                                                                         |
| FR-070 | Permitir una decisión administrativa de cancelar o considerar incobrable una deuda, como acción muy contr… | UC-16, UC-19               | Logic (Aprobaciones, Cobros y conciliación)                                                                          |
| FR-071 | Enviar correos informando del incumplimiento del pago, con un primer aviso a los 3 días y un segundo avis… | UC-18                      | Logic (Recobro); Cron; Workers; NotifManager; MsgAdapter                                                             |
| FR-072 | El superadministrador deberá poder configurar los tiempos de los avisos de pago                            | UC-04, UC-18               | Logic (Configuración, Recobro); Cron; Workers; NotifManager; MsgAdapter                                              |
| FR-073 | Si un cliente tiene condiciones pactadas a treinta días, su vencimiento deberá prevalecer sobre el valor … | UC-18                      | Logic (Recobro); Cron; Workers; NotifManager; MsgAdapter                                                             |
| FR-074 | Una vez pasado el vencimiento, el sistema deberá continuar el flujo de impago enviando recordatorios peri… | UC-18                      | Logic (Recobro); Cron; Workers; NotifManager; MsgAdapter                                                             |
| FR-075 | Permitir suspender temporalmente los recordatorios de una factura                                          | UC-18                      | Logic (Recobro); Cron; Workers; NotifManager; MsgAdapter                                                             |
| FR-076 | Permitir abrir incidencias de facturación asociadas a un cliente y, opcionalmente, a facturas, cada una c… | UC-20                      | Logic (Incidencias, comentarios y adjuntos)                                                                          |
| FR-077 | Mientras una incidencia afecte al cobro, el sistema deberá permitir que pause los recordatorios si así se… | UC-18, UC-20               | Logic (Recobro, Incidencias, comentarios y adjuntos); Cron; Workers; NotifManager; MsgAdapter                        |
| FR-078 | Al cerrar una incidencia, el sistema deberá dejar registrada la resolución                                 | UC-20                      | Logic (Incidencias, comentarios y adjuntos)                                                                          |
| FR-079 | Exigir que las excepciones (como cambios no habituales o descuentos) sean revisadas antes de enviar la fa… | UC-08, UC-12, UC-16, UC-19 | Logic (Suscripciones y tarifas, Facturación, Documentos, Aprobaciones, Cobros y conciliación); Storage; NotifManager |
| FR-080 | Ofrecer una bandeja de aprobaciones ante cambios, cada una con su estado (pendiente, aprobada, rechazada)… | UC-16, UC-19               | Logic (Aprobaciones, Cobros y conciliación)                                                                          |
| FR-081 | Disponer de una bandeja de notificaciones y hacer llegar al responsable encargado de la facturación de un… | UC-22                      | NotifManager                                                                                                         |
| FR-082 | Ofrecer al administrador una bandeja global y permitirle elegir qué notificaciones recibe según su prefer… | UC-22                      | NotifManager                                                                                                         |
| FR-083 | Permitir marcar como leídas las notificaciones únicamente informativas; las que requieran atención deberá… | UC-22                      | NotifManager                                                                                                         |
| FR-084 | Al entrar en la aplicación, el sistema deberá mostrar una pantalla de trabajo diario con las facturas que… | UC-23                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-085 | Permitir configurar alertas según eventos concretos o preferencias, siempre respetando los permisos de us… | UC-22                      | NotifManager                                                                                                         |
| FR-086 | Incluir un calendario laboral para la correcta organización de los diferentes eventos próximos             | UC-04                      | Logic (Configuración)                                                                                                |
| FR-087 | Permitir buscar y filtrar facturas a partir del nombre comercial, la razón social, el NIF, la persona de … | UC-24                      | Logic (Clientes y contactos)                                                                                         |
| FR-088 | Permitir asignar etiquetas relacionadas con el sector, la ubicación o el cliente para conseguir una organ… | UC-06, UC-24               | Logic (Clientes y contactos)                                                                                         |
| FR-089 | Listar los clientes favoritos o recientes para un acceso más cómodo                                        | UC-24                      | Logic (Clientes y contactos)                                                                                         |
| FR-090 | Generar informes semanales, mensuales y trimestrales de los cobros                                         | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-091 | Permitir filtrar los informes por periodo, cliente, responsable interno, país, sector, estado de factura,… | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-092 | Permitir exportar los resultados de los informes, al menos, a CSV, y obtener un informe legible en PDF     | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-093 | La exportación deberá respetar los permisos del usuario                                                    | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-094 | Ofrecer a los administradores un dashboard financiero que muestre lo facturado en la semana, el mes, el t… | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-095 | El dashboard financiero deberá mostrar lo cobrado, lo pendiente y lo vencido                               | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-096 | El dashboard financiero deberá mostrar el tiempo medio de cobro y la evolución                             | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-097 | El dashboard financiero deberá mostrar los clientes con mayor deuda                                        | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-098 | El dashboard financiero deberá mostrar los ingresos recurrentes esperados del mes, la facturación puntual… | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-099 | Mostrar en el dashboard de los administradores los objetivos de facturación mensuales o anuales y su prog… | UC-26                      | Logic (Informes, dashboard y trabajo diario)                                                                         |
| FR-100 | Los archivos exportados deberán llevar fecha y hora, filtros usados y usuario que los generó, en metadato… | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-101 | Las exportaciones masivas de datos sensibles deberán quedar auditadas                                      | UC-25, UC-28               | Logic (Informes, dashboard y trabajo diario, Auditoría y observabilidad); Workers                                    |
| FR-102 | Los archivos exportados podrán conservarse durante un tiempo configurable y regenerarse si los datos y lo… | UC-04, UC-25               | Logic (Configuración, Informes, dashboard y trabajo diario); Workers                                                 |
| FR-103 | Permitir configurar el plazo de pago por defecto, la cadencia de recordatorios, el dominio de correo perm… | UC-04                      | Logic (Configuración)                                                                                                |
| FR-104 | Validar y auditar los cambios de configuración, que no deberán afectar a las facturas anteriores           | UC-04, UC-28               | Logic (Configuración, Auditoría y observabilidad)                                                                    |
| FR-105 | Ante cambios de configuración delicados, el sistema deberá permitir simularlos antes de aplicarlos defini… | UC-04                      | Logic (Configuración)                                                                                                |
| FR-106 | Disponer de plantillas para los correos de factura, recordatorio previo, vencimiento, impago, disputa res… | UC-05                      | NotifManager                                                                                                         |
| FR-107 | Las plantillas deberán admitir variables como nombre de empresa, número de factura, importe, vencimiento,… | UC-05                      | NotifManager                                                                                                         |
| FR-108 | Mantener un historial de versiones de las plantillas a medida que se actualicen                            | UC-05                      | NotifManager                                                                                                         |
| FR-109 | Mantener, a modo de auditoría, un registro de todos los cambios realizados en la aplicación junto con la … | UC-28                      | Logic (Auditoría y observabilidad)                                                                                   |
| FR-110 | Mantener un registro de ejecución de los procesos automáticos, recordatorios, conciliaciones e informes p… | UC-14, UC-29               | Logic (Facturación, Auditoría y observabilidad); Cron; Workers                                                       |
| FR-111 | El administrador deberá poder configurar un resumen semanal o mensual que llegue a determinadas personas   | UC-25                      | Logic (Informes, dashboard y trabajo diario); Workers                                                                |
| FR-112 | Ofrecer una pantalla de salud que muestre la última ejecución de generación recurrente, la cola de correo… | UC-29                      | Logic (Auditoría y observabilidad)                                                                                   |
| FR-113 | Permitir registrar solicitudes de baja de clientes                                                         | UC-10, UC-30               | Logic (Suscripciones y tarifas); API                                                                                 |
| FR-114 | Permitir consultar las deudas pendientes de un cliente y comprobar, a través de la API, si tiene facturas… | UC-10, UC-30               | Logic (Suscripciones y tarifas); API                                                                                 |
| FR-115 | Si el cliente tiene facturas pendientes, el sistema deberá dejar de generar nuevas mensualidades desde la… | UC-10                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-116 | Si el cliente no tiene facturas pendientes, la parte de facturación deberá poder cerrar la recurrencia en… | UC-10                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-117 | Permitir la baja de clientes solo cuando no tengan facturas pendientes, conservando sus datos según la le… | UC-10                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-118 | Si un cliente vuelve a contratar la suscripción, el sistema deberá permitir reactivar su ficha histórica … | UC-10                      | Logic (Suscripciones y tarifas)                                                                                      |
| FR-119 | Permitir guardar un borrador antes de completar un formulario para su revisión                             | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-120 | Notificar los campos erróneos de un formulario                                                             | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-121 | Mostrar valores por defecto en cada campo (país habitual, moneda habitual, vencimiento por defecto y tari… | UC-06                      | Logic (Clientes y contactos)                                                                                         |
| FR-122 | Permitir realizar acciones masivas (por ejemplo, reasignar veinte clientes de un empleado que se marcha, … | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-123 | Antes de ejecutar una acción masiva, el sistema deberá mostrar a cuántos elementos afecta y pedir confirm… | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-124 | Tras una acción masiva, el sistema deberá notificar el resultado con los éxitos y los errores              | UC-03                      | Logic (Identidad y acceso, Clientes y contactos)                                                                     |
| FR-125 | Permitir adjuntar documentos (por ejemplo, el pedido del cliente, el presupuesto aceptado, el justificant… | UC-21                      | Logic (Incidencias, comentarios y adjuntos, Documentos); Storage                                                     |
| FR-126 | El acceso a un adjunto deberá seguir los permisos del cliente o de la factura a los que esté asociado      | UC-21                      | Logic (Incidencias, comentarios y adjuntos, Documentos); Storage                                                     |
| FR-127 | Permitir comentar en una factura o en una incidencia, mencionar a otro usuario y generarle una notificaci… | UC-21, UC-22               | Logic (Incidencias, comentarios y adjuntos, Documentos); Storage; NotifManager                                       |

(\*) FR-011 no aparece en la línea «Requisitos» de ningún caso de uso de la especificación. Se asigna aquí de forma provisional a UC-17 y UC-23 (véase 10.1).
