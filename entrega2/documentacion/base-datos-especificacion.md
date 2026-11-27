# Modelo Físico de Base de Datos

- **Proyecto:** Sistema Interno de Gestión de Facturación y Cobros (**Turbine**)
- **Entrega:** Entrega 2 — Diseño del Sistema
- **Artefacto:** Capa de Persistencia, Especificación del Modelo Físico Entidad-Relación (ERD) y Diccionario de Datos
- **Diagrama Asociado:** `doc/src/base-datos-entidad-relacion-diagrama.pu` _(exportado a `doc/base-datos-entidad-relacion-diagrama.pdf`)_
- **Motor y ORM:** PostgreSQL / Prisma ORM
- **Notación:** _Information Engineering_ (IE / _Crow's Foot_ — Oracle SQL Developer Data Modeler)

---

## 1. Introducción y Fronteras de la Persistencia

### 1.1. Propósito del Documento

Este documento constituye la especificación técnica del **Modelo Físico de Base de Datos (ERD)** de **Turbine**, complementando la representación gráfica contenida en `doc/base-datos-entidad-relacion-diagrama.pdf`. En él se detallan las decisiones arquitectónicas de persistencia, la organización modular de las **23 tablas** del esquema, las reglas de integridad referencial, el diccionario de datos exhaustivo y la matriz de trazabilidad con los requisitos funcionales, no funcionales y diagramas dinámicos del sistema.

### 1.2. Alcance y Responsabilidades de la Capa de Datos

El diseño relacional delimita con precisión qué información reside en PostgreSQL y qué responsabilidades corresponden a otros bloques de la arquitectura:

1. **Separación entre datos estructurados y almacenamiento binario:** PostgreSQL persiste exclusivamente datos transaccionales, contables, de configuración, auditoría y metadatos de ficheros. Los archivos binarios (PDFs autogenerados de facturas y documentos adjuntos de soporte) se almacenan en un repositorio de objetos aislado (`StorageSeguro`, compatible con Amazon S3 / MinIO), referenciados mediante su ruta lógica en la tabla `ADJUNTO`.
2. **Integridad física frente a control de acceso (RBAC):** La base de datos garantiza la integridad referencial, la unicidad de claves de negocio, la obligatoriedad de campos fiscales y la precisión decimal exacta. La validación de permisos por rol (`SUPERADMIN`, `ADMIN`, `OPERATIVO`) y el filtrado de cartera por responsable asignado se ejecutan en la capa lógica del servidor Backend (API REST), ya que la aplicación conecta con el motor mediante una única cuenta técnica de infraestructura.
3. **Ausencia de portal externo:** Los clientes externos de Turbine y sus personas de contacto no disponen de acceso directo a la base de datos ni a la interfaz web; toda la operativa recae sobre usuarios internos de la organización.

---

## 2. Decisiones Arquitectónicas y Justificación del Modelado

Cada decisión estructural del esquema físico responde a una regla de negocio o restricción técnica del proyecto:

- **DA-BD-01 — Elección de Motor Relacional (PostgreSQL) y Transacciones ACID:** Se adopta PostgreSQL por su cumplimiento estricto de las propiedades ACID (_Atomicidad, Consistencia, Aislamiento y Durabilidad_). En procesos críticos como la facturación masiva mensual (`DS-02`) o la conciliación de cobros (`DS-03`), las transacciones garantizan que una factura y todas sus líneas de detalle se persistan de forma atómica o se reviertan íntegramente ante cualquier fallo.
- **DA-BD-02 — Precisión Financiera Estricta (`DECIMAL(12,2)` y `DECIMAL(5,2)`):** Todos los importes monetarios (precios base, bases imponibles, descuentos, cuotas tributarias y totales) emplean tipos de punto fijo exacto `DECIMAL(12,2)`, mientras que los porcentajes de IVA emplean `DECIMAL(5,2)`. Queda prohibido el uso de tipos de coma flotante (`FLOAT` o `DOUBLE PRECISION`) para evitar desviaciones por redondeo en cálculos contables (`NFR-12`).
- **DA-BD-03 — Estrategia de Claves (UUID, Claves Únicas de Negocio y Claves Compuestas):**
  - **Identificadores internos (`UUID`):** Las entidades principales utilizan claves primarias subrogadas de 36 caracteres (`VARCHAR(36)`) para impedir la enumeración predecible de recursos en las rutas de la API.
  - **Claves Alternativas (`<<UK>>`) e Idempotencia:** Se establecen restricciones `UNIQUE` sobre identificadores naturales críticos (`USUARIO.email`, `CLIENTE.nif`, `ETIQUETA.nombre` y `EJECUCION_PROCESO.clave`) y restricciones `UNIQUE` compuestas sobre `FACTURA (serie, numero)`, `FACTURA (suscripcionId, periodoFacturacion)` y `COBRO (fuente, eventoExternoId)`. Esto garantiza la idempotencia en procesos asíncronos y reintentos de _workers_, evitando duplicidades legales de clientes, facturas, mensualidades o cobros.
  - **Claves Primarias Compuestas:** Las tablas débiles de detalle y las tablas asociativas que resuelven relaciones muchos a muchos (`N:M`) emplean claves primarias compuestas que reflejan su dependencia existencial: `(facturaId, numeroLinea)` en `LINEA_FACTURA`, `(clienteId, etiquetaId)` en `CLIENTE_ETIQUETA`, `(clienteId, contactoId, rol)` en `CLIENTE_CONTACTO` y `(notificacionId, usuarioId)` en `RECIBE_NOTIFICACION`.
- **DA-BD-04 — Retención Legal y Borrado Lógico (_Soft Delete_):** Para cumplir con la prescripción fiscal española (mínimo de 4 a 6 años) y el Reglamento General de Protección de Datos (RGPD / `NFR-07`), el sistema no ejecuta sentencias `DELETE` físicas sobre usuarios, clientes ni facturas emitidas. Se emplean marcas de borrado lógico (`estado = 'INACTIVO'` en `CLIENTE`; `activo = FALSE` en `USUARIO` y `CONTACTO`) que bloquean la generación de nuevas mensualidades manteniendo vivo el histórico contable y el circuito de recobro de deudas pendientes (`RB-09`).
- **DA-BD-05 — Inmutabilidad Contable y Trazabilidad de Facturas Rectificativas (`RB-06`, `RB-08`):** La tabla `FACTURA` incorpora el atributo `trimestreCerrado`. Cuando un periodo fiscal se cierra, la factura queda bloqueada frente a ediciones directas. Cualquier corrección posterior exige emitir una nueva factura rectificativa vinculada a la original mediante la clave foránea autoreferenciada `facturaOriginalId`.
- **DA-BD-06 — Auditoría Inmutable (_Append-Only_) a Nivel de Aplicación:** Dado que los disparadores internos de base de datos desconocen qué usuario web realizó una acción, la tabla `AUDITORIA_LOG` se diseña bajo un patrón de solo inserción (_Append-Only_). En cada mutación sensible se persiste el `usuarioId`, la entidad afectada, el identificador del registro y las instantáneas `valoresAnteriores` y `valoresNuevos` en formato `JSON`, revocando permisos de `UPDATE` y `DELETE` sobre dicha tabla.
- **DA-BD-07 — Continuidad Operativa en Cartera de Clientes:** La tabla `CLIENTE` separa dos relaciones explícitas hacia `USUARIO`: un `responsablePrincipalId` obligatorio (`NOT NULL`) y un `responsableSuplenteId` opcional (`NULL`), garantizando la cobertura operativa ante ausencias del gestor titular.
- **DA-BD-08 — Estado de factura derivado y concurrencia optimista:** `FACTURA` solo persiste el ciclo del documento (`estadoDocumento`) y la marca `incobrable`; el estado de cobro (pendiente, vencida, pagada) se calcula a partir de `COBRO`, `total` y `fechaVencimiento` (`A-14`, `RB-10`). `CLIENTE`, `SUSCRIPCION` y `FACTURA` incorporan la columna `version` para la concurrencia optimista (`NFR-17`), y `EJECUCION_PROCESO` registra cada ejecución de los procesos automáticos con una clave única (`FR-110`, `NFR-21`).

---

## 3. Estructura Modular y Reglas de Integridad Referencial

La representación gráfica completa del esquema físico se encuentra exportada en **`doc/base-datos-entidad-relacion-diagrama.pdf`** bajo la notación _Information Engineering_ (_Crow's Foot_ — Oracle SQL Developer Data Modeler). Estructuralmente, las **23 tablas** se organizan en cinco bloques funcionales:

| Módulo Funcional                           | Tablas Incluidas                                                                                                                              | Propósito Arquitectónico                                                                                                         |
| :----------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------- |
| **1. Identidad y Configuración**           | `USUARIO`, `CONFIGURACION`, `SERIE_FACTURACION`, `TARIFA_RANGO`                                                                               | Autenticación interna, roles RBAC, parametrización global dinámica, series de numeración y tabla de tramos de tarifa versionada. |
| **2. Clientes, Etiquetas y Suscripciones** | `CLIENTE`, `ETIQUETA`, `CLIENTE_ETIQUETA`, `SUSCRIPCION`, `CONTACTO`, `CLIENTE_CONTACTO`                                                      | Cartera comercial, asignación titular/suplente, personas de contacto con rol, segmentación `N:M` y acuerdos recurrentes.         |
| **3. Facturación y Conciliación**          | `FACTURA`, `LINEA_FACTURA`, `COBRO`                                                                                                           | Emisión recurrente/puntual, cierre trimestral, rectificativas vinculadas y cruce financiero con Stripe.                          |
| **4. Gobierno, Auditoría y Alertas**       | `SOLICITUD_APROBACION`, `AUDITORIA_LOG`, `PLANTILLA_COMUNICACION`, `NOTIFICACION`, `RECIBE_NOTIFICACION`, `ENVIO`, `EJECUCION_PROCESO`        | Autorizaciones sensibles, trazabilidad inmutable (_Append-Only_), envíos de facturas, mensajería interna y registro de procesos.  |
| **5. Soporte, Incidencias y Adjuntos**     | `INCIDENCIA`, `COMENTARIO`, `ADJUNTO`                                                                                                         | Gestión de disputas, notas colaborativas con control de visibilidad y metadatos de documentos en `StorageSeguro`.                |

### Políticas de Integridad Referencial (`ON DELETE`)

- **`ON DELETE RESTRICT`:** Aplicado en relaciones contables, legales y de auditoría (`FACTURA.clienteId`, `FACTURA.suscripcionId`, `FACTURA.serie`, `COBRO.facturaId`, `SUSCRIPCION.tarifaRangoId`, `ENVIO.facturaId`, `ENVIO.plantillaId`, `AUDITORIA_LOG.usuarioId`, `SOLICITUD_APROBACION.solicitanteId`, `CLIENTE.responsablePrincipalId`, `INCIDENCIA.clienteId`, `INCIDENCIA.creadaPorId`). Impide eliminar registros maestros mientras existan documentos fiscales o trazas históricas asociadas.
- **`ON DELETE CASCADE`:** Aplicado exclusivamente en entidades débiles o tablas asociativas dependientes del ciclo de vida de su padre (`LINEA_FACTURA.facturaId`, `CLIENTE_ETIQUETA`, `CLIENTE_CONTACTO`, `RECIBE_NOTIFICACION`, `SUSCRIPCION.clienteId`, `COMENTARIO.incidenciaId`, `ADJUNTO.facturaId`).
- **`ON DELETE SET NULL`:** Aplicado en referencias opcionales para preservar la integridad histórica si la entidad referenciada deja de estar disponible (`CLIENTE.responsableSuplenteId`, `FACTURA.facturaOriginalId`, `COBRO.conciliadoPorId`, `ENVIO.envioOriginalId`, `ENVIO.contactoId`, `SOLICITUD_APROBACION.aprobadorId`, `INCIDENCIA.facturaId`). En `ENVIO`, el destinatario se conserva siempre como instantánea en `destinatario`, aunque el contacto deje de existir.

### Alcance y exclusiones del modelo físico

Las siguientes capacidades aparecen en los requisitos o en la arquitectura, pero quedan **fuera del alcance** de este esquema en la Entrega 2:

- **Presupuestos (`FR-058`, `UC-11` A1):** se pospone su tabla propia; en esta entrega solo se modela la factura puntual resultante.
- **Delegaciones temporales (`FR-014.1`):** la continuidad se cubre con `responsableSuplenteId`; el intervalo de delegación y su registro se dejan para una versión posterior.
- **Catálogo de conceptos frecuentes (`FR-046`):** las líneas guardan el concepto como texto (instantánea), por lo que el catálogo no condiciona las facturas emitidas y se pospone.
- **Centros de coste:** no figuran en `requisitos.md` ni en los casos de uso, por lo que no se modelan.

---

## 4. Diccionario de Datos Detallado

### 4.1. Módulo 1: Identidad y Configuración

#### Tabla: `USUARIO`

Almacena las cuentas de los empleados internos de Turbine con acceso autenticado al sistema.

| Columna    | Tipo de Dato   | Nulabilidad | Restricción     | Descripción y Regla de Negocio                                                               |
| :--------- | :------------- | :---------- | :-------------- | :------------------------------------------------------------------------------------------- |
| `id`       | `VARCHAR(36)`  | `NOT NULL`  | `PK`            | Identificador único universal (UUID) del usuario interno.                                    |
| `email`    | `VARCHAR(255)` | `NOT NULL`  | `UK`            | Correo corporativo único; validado por negocio bajo el dominio corporativo (`@turbine.com`). |
| `password` | `VARCHAR(255)` | `NOT NULL`  | —               | Hash criptográfico de la contraseña del usuario.                                             |
| `rol`      | `VARCHAR(50)`  | `NOT NULL`  | `CHECK`         | Nivel de privilegio RBAC: `'SUPERADMIN'`, `'ADMIN'` u `'OPERATIVO'`.                         |
| `activo`   | `BOOLEAN`      | `NOT NULL`  | `DEFAULT TRUE`  | Indicador de borrado lógico para revocar acceso sin perder el historial de auditoría.        |
| `creadoEn` | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()` | Marca de tiempo de alta del usuario en el sistema.                                           |

#### Tabla: `CONFIGURACION`

Persiste los parámetros globales dinámicos del sistema (plazos de pago por defecto, cadencia de avisos a 3 y 5 días, umbrales de aprobación) sin modificar el código fuente.

| Columna              | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                  |
| :------------------- | :------------- | :---------- | :------------------ | :---------------------------------------------------------------------------------------------- |
| `clave`              | `VARCHAR(100)` | `NOT NULL`  | `PK`                | Nombre identificador único del parámetro de configuración.                                      |
| `usuarioId`          | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Identificador del Superadministrador que realizó la última modificación (`ON DELETE RESTRICT`). |
| `valor`              | `TEXT`         | `NOT NULL`  | —                   | Valor asignado al parámetro en formato texto o serializado.                                     |
| `fechaActualizacion` | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`     | Fecha y hora exacta de la última modificación del parámetro.                                    |

#### Tabla: `SERIE_FACTURACION`

Catálogo de series de numeración fiscal configurables por el superadministrador (`FR-052`, `Q-06`). La fila de la serie se bloquea al emitir para asignar el número correlativo sin huecos (`AD-03`).

| Columna        | Tipo de Dato   | Nulabilidad | Restricción    | Descripción y Regla de Negocio                                                     |
| :------------- | :------------- | :---------- | :------------- | :--------------------------------------------------------------------------------- |
| `codigo`       | `VARCHAR(50)`  | `NOT NULL`  | `PK`           | Código de la serie (ej. `'TUR-2026'`, `'RECT-2026'`).                               |
| `descripcion`  | `VARCHAR(255)` | `NOT NULL`  | —              | Uso de la serie (ordinaria, rectificativa, etc.).                                  |
| `ultimoNumero` | `INTEGER`      | `NOT NULL`  | `DEFAULT 0`    | Último número asignado; se incrementa en la misma transacción que la emisión.       |
| `activa`       | `BOOLEAN`      | `NOT NULL`  | `DEFAULT TRUE` | Indica si la serie admite nuevas emisiones.                                        |

#### Tabla: `TARIFA_RANGO`

Tabla versionada de tramos de tarifa según la facturación anual del cliente (`FR-031`, `FR-103`, `Q-07`).

| Columna            | Tipo de Dato    | Nulabilidad | Restricción | Descripción y Regla de Negocio                                                      |
| :----------------- | :-------------- | :---------- | :---------- | :---------------------------------------------------------------------------------- |
| `id`               | `VARCHAR(36)`   | `NOT NULL`  | `PK`        | Identificador único del tramo.                                                      |
| `nombre`           | `VARCHAR(100)`  | `NOT NULL`  | —           | Nombre legible del tramo.                                                           |
| `facturacionDesde` | `DECIMAL(14,2)` | `NOT NULL`  | —           | Límite inferior de facturación anual del cliente para aplicar el tramo.             |
| `facturacionHasta` | `DECIMAL(14,2)` | `NULL`      | —           | Límite superior; `NULL` en el último tramo (sin tope).                              |
| `precioMensual`    | `DECIMAL(12,2)` | `NOT NULL`  | —           | Mensualidad asociada al tramo, antes de impuestos.                                  |
| `vigenteDesde`     | `DATE`          | `NOT NULL`  | —           | Fecha de efecto del tramo; un cambio de precio crea un registro nuevo (`FR-036`).   |
| `vigenteHasta`     | `DATE`          | `NULL`      | —           | Fin de vigencia; `NULL` mientras el tramo esté en vigor.                            |

---

### 4.2. Módulo 2: Clientes, Etiquetas y Suscripciones

#### Tabla: `CLIENTE`

Representa a las empresas clientes de Turbine a las que se prestan y facturan servicios.

| Columna                  | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                  |
| :----------------------- | :------------- | :---------- | :------------------ | :---------------------------------------------------------------------------------------------- |
| `id`                     | `VARCHAR(36)`  | `NOT NULL`  | `PK`                | Identificador único universal (UUID) del cliente.                                               |
| `nif`                    | `VARCHAR(50)`  | `NOT NULL`  | `UK`                | Número de Identificación Fiscal; evita duplicados tanto en alta manual como en importación CSV. |
| `nombreComercial`        | `VARCHAR(255)` | `NOT NULL`  | —                   | Denominación comercial de la empresa cliente.                                                   |
| `razonSocial`            | `VARCHAR(255)` | `NOT NULL`  | —                   | Denominación legal oficial obligatoria para emitir facturas válidas.                            |
| `domicilioFiscal`        | `VARCHAR(500)` | `NOT NULL`  | —                   | Domicilio fiscal del cliente, obligatorio en la factura (`UC-06`, `FR-017.1`).                  |
| `pais`                   | `VARCHAR(100)` | `NOT NULL`  | —                   | País fiscal del cliente; condiciona el tratamiento impositivo aplicable.                        |
| `moneda`                 | `VARCHAR(10)`  | `NOT NULL`  | `DEFAULT 'EUR'`     | Divisa de facturación asociada a la cuenta.                                                     |
| `impuestoRate`           | `DECIMAL(5,2)` | `NOT NULL`  | `DEFAULT 21.00`     | Porcentaje impositivo (IVA) aplicable por defecto al cliente.                                   |
| `estado`                 | `VARCHAR(30)`  | `NOT NULL`  | `CHECK`             | Situación frente a la facturación (`A-14`): `'PENDIENTE_ACTIVACION'`, `'ACTIVO'`, `'PAUSADO'`, `'BAJA_SOLICITADA'`, `'INACTIVO'`. `INACTIVO` es el borrado lógico: mantiene facturas y recobro vivos. |
| `fechaBajaEfectiva`      | `DATE`          | `NULL`      | —                   | Fecha desde la que no se generan nuevas mensualidades (`FR-115`, `FR-116`).                     |
| `responsablePrincipalId` | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Usuario operativo titular asignado al seguimiento del cliente (`ON DELETE RESTRICT`).           |
| `responsableSuplenteId`  | `VARCHAR(36)`  | `NULL`      | `FK -> USUARIO(id)` | Usuario operativo de respaldo ante ausencia del titular (`ON DELETE SET NULL`).                 |
| `version`                | `INTEGER`      | `NOT NULL`  | `DEFAULT 0`         | Contador de concurrencia optimista (`AD-10`, `NFR-17`).                                         |
| `creadoEn`               | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`     | Marca de tiempo de alta del cliente.                                                            |

> El correo electrónico y el teléfono ya no se guardan en `CLIENTE`: residen en `CONTACTO`, vinculado mediante `CLIENTE_CONTACTO` con su rol y prioridad (`FR-017.1`). Así los datos personales quedan separados del documento contable (`NFR-07`).

#### Tabla: `CONTACTO`

Personas de contacto de los clientes, destinatarias de facturas y comunicaciones (`UC-06`, `UC-13`, `FR-017.1`, `FR-023`).

| Columna          | Tipo de Dato   | Nulabilidad | Restricción     | Descripción y Regla de Negocio                                                           |
| :--------------- | :------------- | :---------- | :-------------- | :--------------------------------------------------------------------------------------- |
| `id`             | `VARCHAR(36)`  | `NOT NULL`  | `PK`            | Identificador único del contacto.                                                        |
| `nombre`         | `VARCHAR(255)` | `NOT NULL`  | —               | Nombre de la persona de contacto.                                                        |
| `email`          | `VARCHAR(255)` | `NOT NULL`  | —               | Correo electrónico de envío.                                                             |
| `telefono`       | `VARCHAR(50)`  | `NULL`      | —               | Teléfono con prefijo internacional, necesario para WhatsApp.                             |
| `idioma`         | `VARCHAR(5)`   | `NOT NULL`  | `DEFAULT 'es'`  | Idioma de las plantillas para este contacto (`NFR-22`).                                  |
| `admiteWhatsApp` | `BOOLEAN`      | `NOT NULL`  | `DEFAULT FALSE` | Consentimiento registrado para el canal WhatsApp (`FR-023`, `FR-062`).                   |
| `activo`         | `BOOLEAN`      | `NOT NULL`  | `DEFAULT TRUE`  | Permite marcar el contacto como inactivo sin perder el histórico de envíos (`FR-017.1`). |
| `creadoEn`       | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()` | Marca de tiempo de alta del contacto.                                                    |

#### Tabla: `CLIENTE_CONTACTO`

Tabla asociativa `N:M` entre `CLIENTE` y `CONTACTO` con rol y prioridad. Un mismo contacto puede tener varios roles en un cliente.

| Columna      | Tipo de Dato  | Nulabilidad | Restricción              | Descripción y Regla de Negocio                                                                     |
| :----------- | :------------ | :---------- | :----------------------- | :------------------------------------------------------------------------------------------------- |
| `clienteId`  | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> CLIENTE(id)`  | Cliente al que pertenece el contacto (`ON DELETE CASCADE`).                                        |
| `contactoId` | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> CONTACTO(id)` | Persona de contacto (`ON DELETE CASCADE`).                                                         |
| `rol`        | `VARCHAR(30)` | `NOT NULL`  | `PK, CHECK`              | Función del contacto: `'CONTRATACION'`, `'FACTURACION'`, `'IMPAGO'`.                               |
| `prioridad`  | `INTEGER`     | `NOT NULL`  | `DEFAULT 1`              | Orden de uso dentro del rol; el reintento de envío recorre los contactos de facturación por prioridad. |

#### Tabla: `ETIQUETA`

Catálogo de etiquetas normalizadas para segmentar clientes por sector, ubicación o categoría comercial.

| Columna  | Tipo de Dato   | Nulabilidad | Restricción | Descripción y Regla de Negocio                              |
| :------- | :------------- | :---------- | :---------- | :---------------------------------------------------------- |
| `id`     | `VARCHAR(36)`  | `NOT NULL`  | `PK`        | Identificador único de la etiqueta.                         |
| `nombre` | `VARCHAR(100)` | `NOT NULL`  | `UK`        | Nombre único de la etiqueta (impide duplicados semánticos). |

#### Tabla: `CLIENTE_ETIQUETA`

Tabla asociativa que resuelve la relación muchos a muchos (`N:M`) entre `CLIENTE` y `ETIQUETA`.

| Columna      | Tipo de Dato  | Nulabilidad | Restricción              | Descripción y Regla de Negocio                               |
| :----------- | :------------ | :---------- | :----------------------- | :----------------------------------------------------------- |
| `clienteId`  | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> CLIENTE(id)`  | Identificador del cliente etiquetado (`ON DELETE CASCADE`).  |
| `etiquetaId` | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> ETIQUETA(id)` | Identificador de la etiqueta asignada (`ON DELETE CASCADE`). |

#### Tabla: `SUSCRIPCION`

Modela los acuerdos de facturación recurrente asociados a cada cliente, soportando rangos de tarifa, prorrateos iniciales y pausas temporales.

| Columna            | Tipo de Dato    | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                 |
| :----------------- | :-------------- | :---------- | :------------------ | :--------------------------------------------------------------------------------------------- |
| `id`               | `VARCHAR(36)`   | `NOT NULL`  | `PK`                | Identificador único de la suscripción.                                                         |
| `clienteId`        | `VARCHAR(36)`   | `NOT NULL`  | `FK -> CLIENTE(id)` | Cliente titular de la suscripción (`ON DELETE CASCADE`).                                       |
| `tarifaRangoId`    | `VARCHAR(36)`   | `NOT NULL`  | `FK -> TARIFA_RANGO(id)` | Tramo de tarifa según la facturación anual (`Q-07`). La periodicidad es siempre mensual. Los cambios de tarifa crean un nuevo registro con fecha de efecto y no reescriben facturas emitidas (`ON DELETE RESTRICT`). |
| `precioBase`       | `DECIMAL(12,2)` | `NOT NULL`  | —                   | Importe base pactado antes de impuestos y prorrateos.                                          |
| `pausada`          | `BOOLEAN`       | `NOT NULL`  | `DEFAULT FALSE`     | Si es `TRUE`, el proceso _batch_ (`DS-02`) excluye esta suscripción de la facturación mensual. |
| `motivoPausa`      | `VARCHAR(255)`  | `NULL`      | —                   | Justificación registrada al suspender temporalmente el servicio.                               |
| `fechaInicioPausa` | `TIMESTAMP`     | `NULL`      | —                   | Fecha de inicio de la suspensión temporal.                                                     |
| `fechaFinPausa`    | `TIMESTAMP`     | `NULL`      | —                   | Fecha prevista de reactivación.                                                                |
| `version`          | `INTEGER`       | `NOT NULL`  | `DEFAULT 0`         | Contador de concurrencia optimista (`AD-10`, `NFR-17`).                                        |
| `creadoEn`         | `TIMESTAMP`     | `NOT NULL`  | `DEFAULT NOW()`     | Marca de tiempo de creación de la suscripción.                                                 |

---

### 4.3. Módulo 3: Facturación y Conciliación de Cobros

#### Tabla: `FACTURA`

Representa el documento contable emitido a un cliente, soportando borradores, facturas recurrentes, puntuales, rectificativas y bloqueos por cierre trimestral.

| Columna             | Tipo de Dato    | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                       |
| :------------------ | :-------------- | :---------- | :------------------ | :--------------------------------------------------------------------------------------------------- |
| `id`                | `VARCHAR(36)`   | `NOT NULL`  | `PK`                | Identificador interno único de la factura.                                                           |
| `clienteId`         | `VARCHAR(36)`   | `NOT NULL`  | `FK -> CLIENTE(id)` | Cliente receptor de la factura (`ON DELETE RESTRICT`).                                               |
| `suscripcionId`     | `VARCHAR(36)`   | `NULL`      | `FK -> SUSCRIPCION(id)` | Suscripción de origen cuando `tipo = 'RECURRENTE'`; `NULL` en puntuales y rectificativas (`ON DELETE RESTRICT`). Forma parte de la `UK` compuesta `(suscripcionId, periodoFacturacion)`. |
| `facturaOriginalId` | `VARCHAR(36)`   | `NULL`      | `FK -> FACTURA(id)` | Clave autoreferenciada; referencia la factura original corregida cuando `tipo = 'RECTIFICATIVA'`.    |
| `serie`             | `VARCHAR(50)`   | `NULL`      | `FK -> SERIE_FACTURACION(codigo)` | Código de serie fiscal (ej. `'TUR-2026'`, `'RECT-2026'`); `NULL` en borrador, se asigna al emitir (`ON DELETE RESTRICT`). |
| `numero`            | `VARCHAR(100)`  | `NULL`      | `UK compuesta (serie, numero)` | Numeración oficial correlativa dentro de la serie, asignada únicamente en el momento de la emisión (`NULL` en borrador). |
| `tipo`              | `VARCHAR(50)`   | `NOT NULL`  | `CHECK`             | Clasificación contable: `'RECURRENTE'`, `'PUNTUAL'` o `'RECTIFICATIVA'`.                             |
| `estadoDocumento`   | `VARCHAR(30)`   | `NOT NULL`  | `CHECK`             | Ciclo del documento: `'BORRADOR'`, `'PENDIENTE_APROBACION'`, `'LISTA_EMITIR'`, `'EMITIDA'`, `'ANULADA'`, `'RECTIFICADA'`. El estado de cobro no se almacena: se deriva de `COBRO` y `fechaVencimiento`. |
| `incobrable`        | `BOOLEAN`       | `NOT NULL`  | `DEFAULT FALSE`     | `TRUE` solo tras aprobar una `SOLICITUD_APROBACION` de tipo `'INCOBRABLE'`; no equivale a pagada.     |
| `periodoFacturacion`| `DATE`          | `NULL`      | `UK compuesta (suscripcionId, periodoFacturacion)` | Primer día del mes facturado en las recurrentes; impide generar dos veces la misma mensualidad (`NFR-21`, `DS-02`). |
| `fechaEmision`      | `DATE`          | `NULL`      | —                   | Fecha de negocio oficial de emisión de la factura (`NFR-23`).                                        |
| `fechaVencimiento`  | `DATE`          | `NOT NULL`  | —                   | Fecha límite de pago; determina el disparo de avisos de impago a los 3 y 5 días (`RB-01`).           |
| `moneda`            | `VARCHAR(10)`   | `NOT NULL`  | `DEFAULT 'EUR'`     | Divisa del documento, copiada del cliente al crear la factura (instantánea, `AD-07`).                |
| `baseImponible`     | `DECIMAL(12,2)` | `NOT NULL`  | —                   | Suma de subtotales de las líneas antes de impuestos.                                                 |
| `impuestoTotal`     | `DECIMAL(12,2)` | `NOT NULL`  | —                   | Cuota tributaria total calculada.                                                                    |
| `total`             | `DECIMAL(12,2)` | `NOT NULL`  | —                   | Importe líquido total a abonar (`baseImponible + impuestoTotal`).                                    |
| `trimestreCerrado`  | `BOOLEAN`       | `NOT NULL`  | `DEFAULT FALSE`     | Si es `TRUE`, inmutabiliza el registro prohibiendo ediciones directas (`RB-08`).                     |
| `version`           | `INTEGER`       | `NOT NULL`  | `DEFAULT 0`         | Contador de concurrencia optimista (`AD-10`, `NFR-17`).                                              |

#### Tabla: `LINEA_FACTURA`

Entidad débil que almacena el desglose inmutable de conceptos e importes de cada factura mediante una clave primaria compuesta.

| Columna          | Tipo de Dato    | Nulabilidad | Restricción             | Descripción y Regla de Negocio                                   |
| :--------------- | :-------------- | :---------- | :---------------------- | :--------------------------------------------------------------- |
| `facturaId`      | `VARCHAR(36)`   | `NOT NULL`  | `PK, FK -> FACTURA(id)` | Identificador de la factura padre (`ON DELETE CASCADE`).         |
| `numeroLinea`    | `INTEGER`       | `NOT NULL`  | `PK`                    | Número correlativo de línea dentro de la factura (`1, 2, 3...`). |
| `concepto`       | `VARCHAR(255)`  | `NOT NULL`  | —                       | Descripción del servicio o mensualidad facturada.                |
| `cantidad`       | `INTEGER`       | `NOT NULL`  | `DEFAULT 1`             | Unidades facturadas.                                             |
| `precioUnitario` | `DECIMAL(12,2)` | `NOT NULL`  | —                       | Precio unitario sin impuestos.                                   |
| `descuento`      | `DECIMAL(12,2)` | `NOT NULL`  | `DEFAULT 0.00`          | Importe de descuento aplicado sobre la línea.                    |
| `impuestoRate`   | `DECIMAL(5,2)`  | `NOT NULL`  | —                       | Porcentaje impositivo aplicable a la línea.                      |

#### Tabla: `COBRO`

Registra los movimientos de pago confirmados por la pasarela externa (Stripe simulado) para su conciliación con las facturas (`DS-03`). La restricción `UNIQUE (fuente, eventoExternoId)` garantiza que un evento de pago duplicado no se contabilice dos veces (`FR-067.1`).

| Columna              | Tipo de Dato    | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                           |
| :------------------- | :-------------- | :---------- | :------------------ | :------------------------------------------------------------------------------------------------------- |
| `id`                 | `VARCHAR(36)`   | `NOT NULL`  | `PK`                | Identificador único del evento de cobro.                                                                 |
| `facturaId`          | `VARCHAR(36)`   | `NULL`      | `FK -> FACTURA(id)` | Factura contra la que se concilia el pago; `NULL` mientras el cobro esté pendiente de conciliación manual (`FR-067.2`, `ON DELETE RESTRICT`). |
| `fuente`             | `VARCHAR(30)`   | `NOT NULL`  | `UK compuesta (fuente, eventoExternoId)` | Origen del cobro (ej. `'STRIPE'`); permite añadir otras fuentes sin cambiar el modelo.     |
| `eventoExternoId`    | `VARCHAR(255)`  | `NOT NULL`  | `UK compuesta (fuente, eventoExternoId)` | Identificador del evento en la fuente externa; base de la idempotencia.                    |
| `referenciaExterna`  | `VARCHAR(255)`  | `NULL`      | —                   | Referencia de pago aportada por el pagador (concepto o identificador de factura), si existe.            |
| `fechaPago`          | `TIMESTAMP`     | `NOT NULL`  | —                   | Fecha y hora efectiva del abono.                                                                         |
| `importe`            | `DECIMAL(12,2)` | `NOT NULL`  | —                   | Cantidad recibida; se contrasta con `FACTURA.total` para detectar pagos parciales o sobrantes (`RB-10`). |
| `moneda`             | `VARCHAR(10)`   | `NOT NULL`  | —                   | Divisa del cobro recibido.                                                                               |
| `concepto`           | `VARCHAR(255)`  | `NULL`      | —                   | Concepto o descripción recibida de la fuente.                                                            |
| `estadoConciliacion` | `VARCHAR(30)`   | `NOT NULL`  | `CHECK`             | `'PENDIENTE'`, `'AUTOMATICA'`, `'MANUAL'`, `'REVERTIDA'`. `PENDIENTE` exige revisión manual (`FR-067.2`, `FR-068`). |
| `conciliadoPorId`    | `VARCHAR(36)`   | `NULL`      | `FK -> USUARIO(id)` | Usuario que concilió o revirtió manualmente; `NULL` si lo hizo una regla automática (`ON DELETE SET NULL`). |
| `fechaConciliacion`  | `TIMESTAMP`     | `NULL`      | —                   | Momento de la conciliación o de su reversión.                                                            |
| `motivoReversion`    | `TEXT`          | `NULL`      | —                   | Motivo registrado al revertir una conciliación errónea sin borrar el evento (`FR-069.1`).                |

---

### 4.4. Módulo 4: Gobierno, Auditoría y Notificaciones

#### Tabla: `SOLICITUD_APROBACION`

Gestiona el flujo de autorización administrativa para excepciones tarifarias, descuentos, bajas externas, rectificativas o deudas incobrables (`DS-05`, `DS-08`).

| Columna           | Tipo de Dato  | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                    |
| :---------------- | :------------ | :---------- | :------------------ | :------------------------------------------------------------------------------------------------ |
| `id`              | `VARCHAR(36)` | `NOT NULL`  | `PK`                | Identificador único de la solicitud.                                                              |
| `tipo`            | `VARCHAR(50)` | `NOT NULL`  | —                   | Categoría (`'CAMBIO_RANGO'`, `'DESCUENTO'`, `'INCOBRABLE'`, `'RECTIFICATIVA'`, `'BAJA_CLIENTE'`). |
| `estado`          | `VARCHAR(50)` | `NOT NULL`  | —                   | Estado de resolución (`'PENDIENTE'`, `'APROBADA'`, `'RECHAZADA'`).                                |
| `solicitanteId`   | `VARCHAR(36)` | `NOT NULL`  | `FK -> USUARIO(id)` | Usuario interno que origina la solicitud (`ON DELETE RESTRICT`).                                  |
| `aprobadorId`     | `VARCHAR(36)` | `NULL`      | `FK -> USUARIO(id)` | Administrador que resuelve la petición (`ON DELETE SET NULL`).                                    |
| `datosCambio`     | `JSON`        | `NOT NULL`  | —                   | Carga estructurada con los valores propuestos, entidad destino y justificación.                   |
| `fechaSolicitud`  | `TIMESTAMP`   | `NOT NULL`  | `DEFAULT NOW()`     | Marca de tiempo de registro de la solicitud.                                                      |
| `fechaResolucion` | `TIMESTAMP`   | `NULL`      | —                   | Marca de tiempo en que el administrador aprueba o rechaza.                                        |

#### Tabla: `AUDITORIA_LOG`

Registro inmutable (_Append-Only_) de todas las mutaciones sensibles realizadas en el sistema (`UC-28`, `FR-109`).

| Columna             | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                                |
| :------------------ | :------------- | :---------- | :------------------ | :-------------------------------------------------------------------------------------------- |
| `id`                | `VARCHAR(36)`  | `NOT NULL`  | `PK`                | Identificador único del asiento de auditoría.                                                 |
| `usuarioId`         | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Usuario responsable de la operación ejecutada (`ON DELETE RESTRICT`).                         |
| `entidadAfectada`   | `VARCHAR(100)` | `NOT NULL`  | —                   | Nombre de la entidad modificada (ej. `'Cliente'`, `'Factura'`, `'Suscripcion'`).              |
| `entidadId`         | `VARCHAR(36)`  | `NOT NULL`  | —                   | Identificador UUID del registro afectado.                                                     |
| `accion`            | `VARCHAR(50)`  | `NOT NULL`  | —                   | Tipo de operación (`'CREAR'`, `'ACTUALIZAR'`, `'EMITIR'`, `'APROBAR'`, `'RECTIFICAR'`, etc.). |
| `valoresAnteriores` | `JSON`         | `NULL`      | —                   | Instantánea del estado previo del registro antes de la modificación.                          |
| `valoresNuevos`     | `JSON`         | `NULL`      | —                   | Instantánea del estado resultante tras la operación.                                          |
| `fecha`             | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`     | Marca de tiempo exacta de la transacción.                                                     |

#### Tabla: `PLANTILLA_COMUNICACION`

Almacena y versiona las plantillas de correo electrónico y WhatsApp con variables dinámicas editables por los administradores (`UC-05`).

| Columna         | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                     |
| :-------------- | :------------- | :---------- | :------------------ | :--------------------------------------------------------------------------------- |
| `id`            | `VARCHAR(36)`  | `NOT NULL`  | `PK`                | Identificador único de la plantilla.                                               |
| `ultimoAutorId` | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Último administrador que editó la plantilla (`ON DELETE RESTRICT`).                |
| `tipo`          | `VARCHAR(50)`  | `NOT NULL`  | —                   | Evento disparador (`'FACTURA_EMITIDA'`, `'RECORDATORIO_3DIAS'`, `'IMPAGO_5DIAS'`). |
| `canal`         | `VARCHAR(50)`  | `NOT NULL`  | —                   | Canal de envío (`'EMAIL'`, `'WHATSAPP'`).                                          |
| `asunto`        | `VARCHAR(255)` | `NOT NULL`  | —                   | Asunto del mensaje (con soporte para placeholders como `{{numero_factura}}`).      |
| `cuerpo`        | `TEXT`         | `NOT NULL`  | —                   | Contenido completo de la comunicación con variables dinámicas.                     |
| `version`       | `INTEGER`      | `NOT NULL`  | `DEFAULT 1`         | Número incremental de versión de la plantilla.                                     |
| `activa`        | `BOOLEAN`      | `NOT NULL`  | `DEFAULT TRUE`      | Indica si es la versión vigente empleada por los adaptadores de mensajería.        |

#### Tabla: `NOTIFICACION`

Almacena el contenido centralizado de las alertas internas generadas por los eventos del dominio (`DS-01`, `DS-03`, `DS-04`, `DS-06`, `DS-08`).

| Columna          | Tipo de Dato   | Nulabilidad | Restricción     | Descripción y Regla de Negocio                                                                      |
| :--------------- | :------------- | :---------- | :-------------- | :-------------------------------------------------------------------------------------------------- |
| `id`             | `VARCHAR(36)`  | `NOT NULL`  | `PK`            | Identificador único de la notificación.                                                             |
| `titulo`         | `VARCHAR(255)` | `NOT NULL`  | —               | Resumen breve de la alerta.                                                                         |
| `mensaje`        | `TEXT`         | `NOT NULL`  | —               | Descripción detallada del evento o acción requerida.                                                |
| `requiereAccion` | `BOOLEAN`      | `NOT NULL`  | `DEFAULT FALSE` | Indica si la alerta exige intervención manual del usuario (ej. discrepancia de cobro o aprobación). |

#### Tabla: `RECIBE_NOTIFICACION`

Tabla asociativa que gestiona la entrega y el estado individual de lectura y resolución de cada notificación por usuario destinatario.

| Columna          | Tipo de Dato  | Nulabilidad | Restricción                  | Descripción y Regla de Negocio                                            |
| :--------------- | :------------ | :---------- | :--------------------------- | :------------------------------------------------------------------------ |
| `notificacionId` | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> NOTIFICACION(id)` | Identificador de la notificación emitida (`ON DELETE CASCADE`).           |
| `usuarioId`      | `VARCHAR(36)` | `NOT NULL`  | `PK, FK -> USUARIO(id)`      | Identificador del usuario destinatario (`ON DELETE CASCADE`).             |
| `fechaEnvio`     | `TIMESTAMP`   | `NOT NULL`  | `DEFAULT NOW()`              | Marca de tiempo en que la notificación se puso a disposición del usuario. |
| `leida`          | `BOOLEAN`     | `NOT NULL`  | `DEFAULT FALSE`              | Indica si el usuario ha visualizado la alerta en su bandeja.              |
| `resuelta`       | `BOOLEAN`     | `NOT NULL`  | `DEFAULT FALSE`              | Indica si la tarea asociada a la alerta ha sido completada.               |

#### Tabla: `ENVIO`

Registra cada intento de envío de una factura al cliente, con el destinatario como instantánea y la distinción entre envío original y reintentos (`UC-13`, `FR-064`, `FR-065`, `FR-108.1`).

| Columna           | Tipo de Dato   | Nulabilidad | Restricción                        | Descripción y Regla de Negocio                                                                   |
| :---------------- | :------------- | :---------- | :--------------------------------- | :----------------------------------------------------------------------------------------------- |
| `id`              | `VARCHAR(36)`  | `NOT NULL`  | `PK`                               | Identificador único del envío.                                                                   |
| `facturaId`       | `VARCHAR(36)`  | `NOT NULL`  | `FK -> FACTURA(id)`                | Factura enviada; los reintentos reutilizan la misma factura sin duplicarla (`ON DELETE RESTRICT`). |
| `envioOriginalId` | `VARCHAR(36)`  | `NULL`      | `FK -> ENVIO(id)`                  | `NULL` en el envío original; en un reintento apunta al envío original (`ON DELETE SET NULL`).     |
| `contactoId`      | `VARCHAR(36)`  | `NULL`      | `FK -> CONTACTO(id)`               | Contacto al que se dirigió el envío, si procede de la ficha (`ON DELETE SET NULL`).              |
| `destinatario`    | `VARCHAR(255)` | `NOT NULL`  | —                                  | Dirección o teléfono usado, guardado como instantánea aunque el contacto cambie después.         |
| `canal`           | `VARCHAR(30)`  | `NOT NULL`  | `CHECK`                            | `'EMAIL'` o `'WHATSAPP'`.                                                                        |
| `estado`          | `VARCHAR(30)`  | `NOT NULL`  | `CHECK`                            | `'EN_COLA'`, `'ENVIADO'`, `'ENTREGADO'`, `'FALLIDO'`, `'REBOTADO'`.                              |
| `plantillaId`     | `VARCHAR(36)`  | `NOT NULL`  | `FK -> PLANTILLA_COMUNICACION(id)` | Plantilla y versión utilizadas en el envío (`FR-108.1`, `ON DELETE RESTRICT`).                   |
| `fecha`           | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`                    | Momento del intento de envío.                                                                    |
| `detalleError`    | `TEXT`         | `NULL`      | —                                  | Motivo devuelto por el proveedor en caso de fallo o rebote.                                      |

#### Tabla: `EJECUCION_PROCESO`

Registro de ejecución de los procesos automáticos y de la cola de trabajos que vive en la base de datos (`FR-110`, `FR-112`, `NFR-21`, `AD-05`). La clave única impide encolar o ejecutar dos veces el mismo trabajo.

| Columna          | Tipo de Dato   | Nulabilidad | Restricción | Descripción y Regla de Negocio                                                                 |
| :--------------- | :------------- | :---------- | :---------- | :--------------------------------------------------------------------------------------------- |
| `id`             | `VARCHAR(36)`  | `NOT NULL`  | `PK`        | Identificador único de la ejecución.                                                           |
| `clave`          | `VARCHAR(150)` | `NOT NULL`  | `UK`        | Clave de ejecución (ej. `'GENERACION_RECURRENTE:2026-12'`).                                    |
| `tipo`           | `VARCHAR(40)`  | `NOT NULL`  | —           | Proceso ejecutado (generación recurrente, recordatorios, conciliación, informe, importación).   |
| `inicio`         | `TIMESTAMP`    | `NOT NULL`  | —           | Marca de tiempo de inicio.                                                                     |
| `fin`            | `TIMESTAMP`    | `NULL`      | —           | Marca de tiempo de finalización; `NULL` mientras está en curso.                                 |
| `estado`         | `VARCHAR(30)`  | `NOT NULL`  | `CHECK`     | `'EN_CURSO'`, `'COMPLETADO'`, `'COMPLETADO_CON_ERRORES'`, `'FALLIDO'`.                         |
| `procesados`     | `INTEGER`      | `NOT NULL`  | `DEFAULT 0` | Elementos procesados.                                                                          |
| `creados`        | `INTEGER`      | `NOT NULL`  | `DEFAULT 0` | Elementos creados (por ejemplo, facturas generadas).                                           |
| `fallidos`       | `INTEGER`      | `NOT NULL`  | `DEFAULT 0` | Elementos que fallaron.                                                                        |
| `detalleErrores` | `JSON`         | `NULL`      | —           | Elemento afectado y motivo de cada fallo.                                                      |

---

### 4.5. Módulo 5: Soporte, Incidencias y Gestión Documental

#### Tabla: `INCIDENCIA`

Registra disputas o incidencias operativas vinculadas a un cliente y, opcionalmente, a una factura concreta (`UC-20`).

| Columna           | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                 |
| :---------------- | :------------- | :---------- | :------------------ | :----------------------------------------------------------------------------- |
| `id`              | `VARCHAR(36)`  | `NOT NULL`  | `PK`                | Identificador único de la incidencia.                                          |
| `clienteId`       | `VARCHAR(36)`  | `NOT NULL`  | `FK -> CLIENTE(id)` | Cliente afectado por la incidencia (`ON DELETE RESTRICT`).                     |
| `facturaId`       | `VARCHAR(36)`  | `NULL`      | `FK -> FACTURA(id)` | Factura en disputa, si aplica (`ON DELETE SET NULL`).                          |
| `titulo`          | `VARCHAR(255)` | `NOT NULL`  | —                   | Título descriptivo de la incidencia.                                           |
| `descripcion`     | `TEXT`         | `NOT NULL`  | —                   | Detalle completo del problema reportado.                                       |
| `estado`          | `VARCHAR(50)`  | `NOT NULL`  | —                   | Estado del _ticket_ (`'ABIERTA'`, `'EN_REVISION'`, `'RESUELTA'`, `'CERRADA'`). |
| `creadaPorId`     | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Usuario interno que registró la incidencia (`ON DELETE RESTRICT`).             |
| `fechaCreacion`   | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`     | Fecha y hora de apertura.                                                      |
| `fechaResolucion` | `TIMESTAMP`    | `NULL`      | —                   | Fecha y hora de cierre o resolución.                                           |

#### Tabla: `COMENTARIO`

Anotaciones colaborativas sobre incidencias o facturas con control de privacidad y soporte para menciones `@usuario` (`UC-21`).

| Columna            | Tipo de Dato  | Nulabilidad | Restricción            | Descripción y Regla de Negocio                                                        |
| :----------------- | :------------ | :---------- | :--------------------- | :------------------------------------------------------------------------------------ |
| `id`               | `VARCHAR(36)` | `NOT NULL`  | `PK`                   | Identificador único del comentario.                                                   |
| `incidenciaId`     | `VARCHAR(36)` | `NULL`      | `FK -> INCIDENCIA(id)` | Incidencia a la que pertenece el comentario (`ON DELETE CASCADE`).                    |
| `facturaId`        | `VARCHAR(36)` | `NULL`      | `FK -> FACTURA(id)`    | Factura comentada directamente, si no deriva de una incidencia (`ON DELETE CASCADE`). |
| `usuarioId`        | `VARCHAR(36)` | `NOT NULL`  | `FK -> USUARIO(id)`    | Autor del comentario (`ON DELETE RESTRICT`).                                          |
| `texto`            | `TEXT`        | `NOT NULL`  | —                      | Contenido textual de la nota (admite menciones `@usuario`).                           |
| `restringidoAdmin` | `BOOLEAN`     | `NOT NULL`  | `DEFAULT FALSE`        | Si es `TRUE`, oculta el comentario a usuarios con rol `'OPERATIVO'`.                  |
| `fechaCreacion`    | `TIMESTAMP`   | `NOT NULL`  | `DEFAULT NOW()`        | Marca de tiempo de publicación.                                                       |

#### Tabla: `ADJUNTO`

Metadatos de seguridad y ubicación de los documentos binarios almacenados en `StorageSeguro` (`UC-12`, `UC-21`, `NFR-25`).

| Columna         | Tipo de Dato   | Nulabilidad | Restricción         | Descripción y Regla de Negocio                                                         |
| :-------------- | :------------- | :---------- | :------------------ | :------------------------------------------------------------------------------------- |
| `id`            | `VARCHAR(36)`  | `NOT NULL`  | `PK`                | Identificador único del metadato del fichero.                                          |
| `clienteId`     | `VARCHAR(36)`  | `NULL`      | `FK -> CLIENTE(id)` | Cliente al que pertenece el documento (`ON DELETE CASCADE`).                           |
| `facturaId`     | `VARCHAR(36)`  | `NULL`      | `FK -> FACTURA(id)` | Factura asociada al fichero, como su PDF oficial o justificante (`ON DELETE CASCADE`). |
| `nombreArchivo` | `VARCHAR(255)` | `NOT NULL`  | —                   | Nombre original legible del fichero.                                                   |
| `rutaStorage`   | `VARCHAR(500)` | `NOT NULL`  | —                   | Ruta interna o clave de objeto en `StorageSeguro` (sin exposición pública directa).    |
| `tipoMime`      | `VARCHAR(100)` | `NOT NULL`  | —                   | Tipo MIME validado (`application/pdf`, `image/png`, `image/jpeg`, etc.).               |
| `tamanoBytes`   | `BIGINT`       | `NOT NULL`  | —                   | Tamaño exacto en bytes validado frente al límite configurado (`NFR-25`).               |
| `subidoPorId`   | `VARCHAR(36)`  | `NOT NULL`  | `FK -> USUARIO(id)` | Usuario interno que subió o generó el documento (`ON DELETE RESTRICT`).                |
| `fechaSubida`   | `TIMESTAMP`    | `NOT NULL`  | `DEFAULT NOW()`     | Marca de tiempo de carga en el repositorio seguro.                                     |

---

## 5. Matriz de Trazabilidad Funcional y Dinámica

| Capacidad / Caso de Uso                                             | Diagramas de Secuencia    | Tablas Implicadas                                               | Mecanismo Físico de Soporte                                                                                                   |
| :------------------------------------------------------------------ | :------------------------ | :-------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------- |
| **Autenticación y Roles** (`UC-01`, `UC-02`)                        | `DS-05`, `DS-08`          | `USUARIO`                                                       | Unicidad de `email`, restricción de dominio corporativo y enumeración de roles (`SUPERADMIN`, `ADMIN`, `OPERATIVO`).          |
| **Asignación de Cartera y Continuidad** (`UC-03`, `UC-06`)          | `DS-01`                   | `CLIENTE`, `USUARIO`                                            | Doble clave foránea hacia `USUARIO` (`responsablePrincipalId` obligatorio y `responsableSuplenteId` opcional).                |
| **Contactos del Cliente** (`UC-06`, `UC-13`)                        | `DS-01`                   | `CONTACTO`, `CLIENTE_CONTACTO`, `CLIENTE`                       | Relación `N:M` con `rol` (`CONTRATACION`, `FACTURACION`, `IMPAGO`) y `prioridad`; `domicilioFiscal` obligatorio en `CLIENTE`. |
| **Clasificación y Filtrado Avanzado** (`UC-24`)                     | —                         | `CLIENTE`, `ETIQUETA`, `CLIENTE_ETIQUETA`                       | Resolución `N:M` mediante clave primaria compuesta `(clienteId, etiquetaId)`.                                                 |
| **Importación Masiva CSV Auditada** (`UC-07`)                       | `DS-07`                   | `CLIENTE`, `AUDITORIA_LOG`                                      | Restricción `UNIQUE` en `CLIENTE.nif` para detección de duplicados                                                            |
| **Suscripciones, Prorrateos y Pausas** (`UC-08`, `UC-09`, `UC-10`)  | `DS-02`, `DS-05`          | `SUSCRIPCION`, `CLIENTE`, `TARIFA_RANGO`                        | Separación entre `CLIENTE.estado` y `SUSCRIPCION.pausada` con motivos y fechas de suspensión; tramo de tarifa versionado en `TARIFA_RANGO`. |
| **Facturación Dual, Cierre y Rectificativas** (`UC-11` a `UC-15`)   | `DS-01`, `DS-02`, `DS-05` | `FACTURA`, `LINEA_FACTURA`, `SERIE_FACTURACION`                 | `DECIMAL(12,2)`, `estadoDocumento`, bloqueo `trimestreCerrado`, `UK (serie, numero)`, clave compuesta `(facturaId, numeroLinea)` y autoreferencia `facturaOriginalId`. |
| **Generación Recurrente sin Duplicados** (`UC-14`, `UC-29`)         | `DS-02`                   | `FACTURA`, `SUSCRIPCION`, `EJECUCION_PROCESO`                   | `UK (suscripcionId, periodoFacturacion)` en `FACTURA` y `UK` en `EJECUCION_PROCESO.clave`, con contadores de procesados, creados y fallidos (`FR-110`). |
| **Envío de Facturas y Reintentos** (`UC-13`)                        | `DS-01`                   | `ENVIO`, `CONTACTO`, `PLANTILLA_COMUNICACION`                   | Destinatario como instantánea, autoreferencia `envioOriginalId` para distinguir original y reintentos, y estado de entrega.   |
| **Conciliación de Pagos y Discrepancias** (`UC-17`, `UC-18`)        | `DS-03`, `DS-04`          | `FACTURA`, `COBRO`                                              | Registro de eventos de cobro con `estadoConciliacion` (`PENDIENTE` exige revisión manual y las diferencias de importe quedan señaladas) para contrastar `COBRO.importe` contra `FACTURA.total`; `UK (fuente, eventoExternoId)`. |
| **Flujo de Aprobaciones Sensibles** (`UC-16`, `UC-19`, `UC-30`)     | `DS-05`, `DS-08`          | `SOLICITUD_APROBACION`, `USUARIO`                               | Almacenamiento en `JSON` de los cambios propuestos y doble relación `solicitanteId` / `aprobadorId`.                          |
| **Incidencias, Comentarios y Adjuntos** (`UC-20`, `UC-21`, `UC-22`) | `DS-06`                   | `INCIDENCIA`, `COMENTARIO`, `ADJUNTO`                           | Soporte para privacidad (`restringidoAdmin`) y referencia segura a almacenamiento externo (`rutaStorage`).                    |
| **Notificaciones y Plantillas** (`UC-05`, `UC-18`, `UC-22`)         | `DS-01`, `DS-04`, `DS-06` | `PLANTILLA_COMUNICACION`, `NOTIFICACION`, `RECIBE_NOTIFICACION` | Versionado de plantillas y control individualizado de lectura y resolución por usuario.                                       |
| **Auditoría Inmutable del Sistema** (`UC-28`)                       | `DS-01` a `DS-08`         | `AUDITORIA_LOG`                                                 | Tabla _Append-Only_ con instantáneas `valoresAnteriores` y `valoresNuevos` en `JSON`.                                         |

---
