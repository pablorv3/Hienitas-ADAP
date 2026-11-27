# Cambios respecto a la Entrega 1

Se compara la versión del repositorio al cierre de la Entrega 1 (23/10/2026, commit `bba2265`) con las fuentes actuales de `entrega1-actualizada/fuentes/`.

**Resumen:** requisitos funcionales de 127 a 141 (14 nuevos, con sufijo); una regla de negocio nueva (RB-18); dos suposiciones nuevas (A-15 y A-16). No se elimina ningún identificador BR, UR, FR, NFR ni RB. Las dudas (Q) sí se han renumerado.

## Cambios acordados o incorporados

La columna de impacto indica las áreas de diseño relacionadas. «Sin pendientes identificados» no equivale a una validación completa: la cobertura en el diseño se ha comprobado buscando referencias en sus fuentes.

| Elemento | Cambio | Motivo | Impacto en Entrega 2 | Validación / pendiente |
|---|---|---|---|---|
| UR-03, FR-008, FR-009, FR-014, FR-015, FR-026, FR-081, RB-12, A-04 | Cada cliente tiene obligatoriamente dos usuarios operativos distintos: titular y suplente. | Evitar que la facturación se paralice por ausencia del titular (A-04). | ER, clases, arquitectura. | **Incoherencia:** ER (`responsableSuplenteId`, `NULL`, `ON DELETE SET NULL`) y clases (`0..1`) lo modelan como opcional. |
| UC-03, UC-06 (y precondiciones de UC-11 y UC-20) | UC-03 sustituye desasignar y delegar temporalmente por reasignar y activar al suplente. UC-06 exige titular y suplente en el alta. Se actualiza el diagrama fuente de UC-03. | Alinear los casos de uso con la regla anterior. | ER y arquitectura. | Pendiente volver a exportar el diagrama de UC-03 a PDF. |
| FR-014.1 | Delegación o relevo en el suplente, con registro. | A-15. | ER. | `modelo_datos.md` deja el intervalo de delegación y su registro para una versión posterior. |
| FR-017.1 | Datos comerciales y fiscales separados; varios contactos, que pueden quedar inactivos. | A-15. | ER y clases. | Sin pendientes identificados. |
| FR-041, FR-042, FR-042.1, A-16, UC-08 | Factura inicial prorrateada, cálculo visible antes de confirmar y regla única de prorrateo. | A-15 y A-16. | Clases. | **A-16 es una propuesta pendiente de validar con el cliente.** |
| FR-043.1 | Con fecha de inicio futura no se generan facturas ni recordatorios. | A-15. | Suscripciones (UC-08, UC-14). | No se cita expresamente en la documentación de diseño. |
| FR-031 | El rango es el tramo de tarifa según la facturación anual; la facturación es mensual. | Resolución de Q-07. | ER y clases (`TarifaRango`). | Resuelta. |
| FR-036, FR-036.1 | El nuevo precio se aprueba antes, tiene fecha de efecto y no genera facturas retroactivas. | A-15 (FR-036.1). | Tarifas (UC-09). | FR-036.1 no se cita expresamente en el diseño. |
| FR-052, FR-103 | Las series las configura el superadministrador; la configuración incluye la tabla de tarifas. | Resolución de Q-06 (FR-052). | Configuración (UC-04). | Resuelta. |
| FR-054 | Bloquea si faltan datos fiscales; un importe atípico solo genera advertencia. | Resolución de Q-05. | Emisión (UC-12). | Resuelta. |
| FR-059.1 | Reabrir un periodo exige permiso superior, motivo y auditoría. | Resolución de Q-08. | Cierre (UC-27). | FR-059.1 no se cita expresamente en el diseño. |
| FR-063 a FR-065 | Fallo de envío: incidencia, reintento sin duplicar factura y registro de cada envío. | Concretar la gestión de entregas. | Envíos (UC-13). | Sin pendientes identificados. |
| FR-066, FR-068, FR-070.1, RB-18, A-14 | Se separan ciclo del documento y estado de cobro; incobrable, anulada y rectificada nunca equivalen a pagada. | Resolución de Q-10. | ER, clases, secuencias y arquitectura. | Resuelta. |
| FR-067.1, FR-067.2, FR-069.1 | Pagos sin duplicados, conciliación manual y reversión trazable. | A-15. | ER, clases y secuencia de conciliación. | Sin pendientes identificados. |
| FR-071 | Primer recordatorio hacia el día 3 y segundo en el vencimiento (día 5 por defecto). | Concretar el calendario de recordatorios (A-06). | Secuencia de impagos. | Sin pendientes identificados. |
| FR-080.1, FR-080.2 | Separación de funciones y reasignación de aprobaciones. | A-15. | Aprobaciones (UC-16). | No se citan expresamente en el diseño. |
| FR-108.1 | Correo de prueba y registro de plantilla y versión. | A-15. | ER. | Sin pendientes identificados. |
| FR-114.1 | La API exige autenticación, permisos, límites y registro. | A-15. | Clases y secuencia de API. | Sin pendientes identificados. |
| FR-117, RB-09, UC-10 | La baja es lógica y la deuda no la bloquea; el recobro continúa. | Resolución de Q-04. | ER y clases. | Resuelta. |
| FR-127 | Menciones solo a usuarios con acceso y comentarios restringibles. | Concretar permisos. | Incidencias y comentarios (UC-21). | Sin pendientes identificados. |
| FR-002, actores y casos de uso transversales | Se aclara el dominio de correo y se añade el actor abstracto «Usuario interno» (UC-01, UC-22 a UC-24). | Coherencia de actores; Q-01. | Sin cambio estructural identificado. | Resuelta. |

## Dudas: correspondencia y estado

Se añaden las dudas sobre el dominio de correo (Q-01) y el significado del rango de cobro (Q-07). El resto cambia de número:

| Entrega 1 | Q-01 | Q-02 | Q-03 | Q-04 | Q-05 | Q-06 | Q-07 | Q-08 | Q-09 | Q-10 | Q-11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Actual | Q-02 | Q-03 | Q-04 | Q-05 | Q-06 | Q-08 | Q-09 | Q-10 | Q-11 | Q-12 | Q-13 |

**Siguen pendientes:** Q-02 (tiempo de carga), Q-03 (restricción de red), Q-09 (valores de parámetros) y Q-11 (RGPD en copias de seguridad), además de la validación de A-16. Q-02 y Q-03 figuraban como «Solucionada» con una interpretación provisional.
