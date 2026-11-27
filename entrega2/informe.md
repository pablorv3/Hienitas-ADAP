# Entrega 2 · Diseño del sistema

**Asignatura:** Análisis y Diseño de Aplicaciones · Grupo A · 2026–27  
**Equipo:** Hienitas  
**Integrantes:** Pablo Ruiz Valdés, Adrián Pérez Pacheco, Pedro Javier Ortega Fernández, Sergio Cañas Vegas, Jose Fernández Ocon y Sergio Maximiliano Rodríguez Donaire  
**Fecha:** 27/11/2026  
**Repositorio:** https://github.com/pablorv3/Hienitas-ADAP

## 1. Introducción y alcance

Este informe presenta el diseño del sistema interno de facturación y cobros de Turbine, a partir de los requisitos y los 30 casos de uso de la Entrega 1 actualizada. Es una aplicación web interna que gestiona clientes, suscripciones, facturas (recurrentes, puntuales y rectificativas), envíos, cobros, impagos, aprobaciones, informes y auditoría.

No es una pasarela de pago ni un portal para clientes. En esta fase las integraciones con Stripe, correo y WhatsApp son simuladas y no se integra Verifactu.

## 2. Actualizaciones de la Entrega 1

Se añaden 14 requisitos funcionales con sufijo, la regla RB-18 y las suposiciones A-15 y A-16. Los cambios principales son la obligación de titular y suplente por cliente, la separación entre ciclo del documento y estado de cobro, la baja lógica que no extingue la deuda y la resolución de varias dudas. El detalle está en `cambios_entrega1.md`. La Entrega 1 actualizada se entrega en un PDF separado.

## 3. Modelo físico de datos (Entidad–Relación)

Diagrama: `diagramas/modelo_datos.puml`. Explicación: `documentacion/modelo_datos.md`.

## 4. Diagrama UML de clases

Diagrama: `diagramas/diagrama_clases.puml`. Explicación: `documentacion/diagrama_clases.md`.

## 5. Diagramas UML de secuencia

Diagramas: `diagramas/secuencias/` (DS-01 a DS-08). Explicación, casos de uso y escenario de cada uno: `documentacion/diagramas_secuencia.md`.

## 6. Arquitectura y componentes

Diagrama: `diagramas/arquitectura.puml`. Explicación: `documentacion/arquitectura.md`.

## 7. Informe Scrum

Contenido y capturas del tablero: `informe_scrum.md` y `evidencias/`.

## 8. Conclusiones y cuestiones pendientes

El diseño se documenta en cuatro vistas: datos, clases, interacciones y componentes. El modelo de clases emplea los patrones Facade, Adapter, State, Strategy, Observer, Decorator y Factory Method, y las integraciones externas quedan encapsuladas.

Cuestiones pendientes:

- El suplente es obligatorio en los requisitos (FR-014, FR-026, RB-12), pero opcional en el modelo de datos y el diagrama de clases.
- La delegación temporal con registro (FR-014.1) queda aplazada en el modelo de datos.
- La regla de prorrateo (A-16) está pendiente de validar con el cliente.
- Siguen abiertas las dudas Q-02, Q-03, Q-09 y Q-11.
