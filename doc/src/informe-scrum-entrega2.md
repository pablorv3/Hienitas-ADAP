# Informe Scrum: Entrega 2

**Proyecto:** Sistema interno de facturación y cobros de TurbineH  
**Asignatura:** Análisis y Diseño de Aplicaciones (curso 2026-27)  
**Equipo:** Hienitas  
**Entrega:** Diseño arquitectónico y detallado (27/11/2026)

## 1. Identificación del equipo

| Nombre y apellidos                   | Rol                |
| :----------------------------------- | :----------------- |
| Pablo Ruiz Valdés                    | Scrum Master       |
| Adrián Pérez Pacheco                 | Miembro del equipo |
| Pedro Javier Ortega Fernández        | Miembro del equipo |
| Sergio Cañas Vegas                   | Miembro del equipo |
| Jose Fernández Ocon                  | Miembro del equipo |
| Sergio Maximiliano Rodríguez Donaire | Miembro del equipo |

## 2. Organización del trabajo

Siguiendo la dinámica establecida en la primera entrega, el equipo ha continuado trabajando con una adaptación de Scrum estructurada en tres sprints (Sprints 4, 5 y 6), alineados con las sesiones prácticas del periodo. Cada sprint comenzó con una reunión de planificación (_Sprint Planning_) para asignar tareas y finalizó con una revisión y retrospectiva (_Sprint Review_ y _Sprint Retrospective_).

- **Gestión de tareas:** cumpliendo con la acción de mejora acordada en la Entrega 1, se migró la gestión de tareas a un tablero Kanban formal en **Trello**, organizado mediante cuatro listas (_Product Backlog_, _Sprint Backlog_, _En progreso_ y _Hecho_), con asignación de responsables por tarjeta y etiquetas de tipo y prioridad.
- **Control de versiones:** repositorio público en GitHub ([`Hienitas-ADAP`](https://github.com/pablorv3/Hienitas-ADAP/)), manteniendo la estructura de carpetas (`doc/`, `doc/src/`, `README.md`, `.gitignore`). El equipo ha continuado trabajando únicamente sobre la rama `main`, mejorando la redacción de los mensajes de commit y coordinándose antes de subir cambios para evitar conflictos de fusión.
- **Reparto de responsabilidades:** para evitar que todo el grupo trabajase sobre el mismo tema a la vez, se repartieron en paralelo los cuatro artefactos principales (diagrama físico Entidad-Relación, diagrama UML de clases, diagramas UML de secuencia y diagrama de arquitectura/componentes) en parejas, estableciendo revisiones cruzadas obligatorias para garantizar la coherencia global. El Scrum Master coordinó las reuniones, el tablero de Trello y la integración final.
- **Reuniones:** 4 reuniones presenciales del equipo durante el periodo: tres durante las sesiones prácticas (28/10/2026, 04/11/2026 y 11/11/2026) y una sesión adicional de cierre el 24/11/2026, con margen previo a la entrega del 27/11/2026.
- **Trazabilidad y resolución de dudas:** antes de consolidar el diseño, se revisaron las dudas pendientes de la Entrega 1 con el cliente, actualizando los requisitos y casos de uso afectados y asegurando la trazabilidad directa hacia las tablas, clases, interacciones y componentes diseñados.

## 3. Sprints realizados

### Sprint 4

- **Fecha de inicio:** 28/10/2026
- **Fecha de finalización:** 04/11/2026
- **Objetivo del Sprint:** Implantar el tablero en Trello, cerrar las dudas pendientes de la Entrega 1 y completar el modelo físico Entidad-Relación.
- **Tareas planificadas:**
  - Configurar el tablero Kanban en Trello y migrar las tareas.
  - Revisar las dudas pendientes con el cliente y actualizar requisitos.
  - Actualizar los documentos de la Entrega 1 con las correcciones acordadas.
  - Diseñar el borrador del modelo Entidad-Relación.
  - Elaborar el **diagrama físico Entidad-Relación** completo.
- **Tareas completadas:** Todas las planificadas. El tablero de Trello quedó operativo desde la primera sesión práctica, se cerraron las actualizaciones de la Entrega 1 y se completó el diagrama físico Entidad-Relación.
- **Reuniones realizadas:** Reunión 1 (28/10/2026, 1 h 45 min), de _Sprint Planning_ y resolución de dudas, y Reunión 2 (04/11/2026, 1 h 45 min), de revisión y cierre; véase el apartado 3.1.
- **Problemas encontrados:** Adaptación inicial al seguimiento mediante Trello y necesidad de coordinar las subidas al trabajar todos los miembros directamente sobre la rama `main` de GitHub, además de dudas sobre cómo estructurar ciertas relaciones del modelo físico (como la relación entre clientes, series de facturación, facturas, líneas y cobros).
- **Soluciones aplicadas:** Se acordó actualizar el repositorio local (`git pull`) antes de cada subida y avisar al grupo cuando se modificasen archivos compartidos. Para el modelo físico se contrastaron las reglas de negocio del documento de requisitos, definiendo claves primarias, foráneas y restricciones de integridad referencial.

#### Estado del tablero en el Sprint Planning

![][image1]

#### Estado del tablero en la Sprint Retrospective

![][image2]

### Sprint 5

- **Fecha de inicio:** 04/11/2026
- **Fecha de finalización:** 11/11/2026
- **Objetivo del Sprint:** Desarrollar el diagrama UML de clases e iniciar los diagramas UML de secuencia de los casos de uso principales.
- **Tareas planificadas:**
  - Definir clases, atributos, métodos y relaciones.
  - Elaborar el **diagrama UML de clases** completo.
  - Seleccionar los casos de uso principales para los diagramas de secuencia.
  - Elaborar los **diagramas UML de secuencia (Bloque 1)** correspondientes a los flujos críticos del sistema.
- **Tareas completadas:** Todas las planificadas. Se cerró el diagrama UML de clases y se completó el primer bloque de diagramas de secuencia, verificando su correspondencia con el diagrama físico E-R.
- **Reuniones realizadas:** Reunión 2 (04/11/2026, 1 h 45 min), en la que se planificó el sprint, y Reunión 3 (11/11/2026, 1 h 45 min), de revisión de avances y cierre; véase el apartado 3.1.
- **Problemas encontrados:** Aparecieron discrepancias iniciales entre los nombres de atributos y relaciones del diagrama físico E-R y los del diagrama UML de clases, así como diferencias en el nivel de detalle al repartir los diagramas de secuencia entre varios miembros.
- **Soluciones aplicadas:** Se realizó una sesión de alineación cruzada entre los responsables del E-R y de clases para unificar nombres y tipos. Para los diagramas de secuencia se fijó una plantilla común que especifica qué capas/componentes participan y cómo representar alternativas (`alt`), bucles (`loop`) y excepciones.

#### Estado del tablero en el Sprint Planning

![][image3]

#### Estado del tablero en la Sprint Retrospective

![][image4]

### Sprint 6

- **Fecha de inicio:** 11/11/2026
- **Fecha de finalización:** 27/11/2026 (cierre técnico y reunión final el 24/11/2026)
- **Objetivo del Sprint:** Completar los diagramas UML de secuencia restantes, elaborar el diagrama de arquitectura/componentes, validar la trazabilidad completa y preparar la Entrega 2.
- **Tareas planificadas:**
  - Elaborar los **diagramas UML de secuencia (Bloque 2)**.
  - Elaborar el **diagrama de arquitectura/componentes** reflejando interfaces, dependencias y despliegue lógico de los módulos del sistema.
  - Verificar la trazabilidad y coherencia entre requisitos, casos de uso y los cuatro diagramas de diseño.
  - Unificar formatos y subir la entrega final a la rama `main` en GitHub.
  - Redactar el informe Scrum de la Entrega 2 y la evaluación interna.
- **Tareas completadas:** Todas las planificadas. Los diagramas, fuentes y documentos quedaron integrados en la rama `main` del repositorio tras la reunión de cierre del 24/11/2026, listos para su entrega el 27/11/2026.
- **Reuniones realizadas:** Reunión 3 (11/11/2026, 1 h 45 min), de planificación, y Reunión 4 (24/11/2026, 2 h), de integración y cierre; véase el apartado 3.1.
- **Problemas encontrados:** Algunos métodos invocados en los diagramas de secuencia no estaban recogidos todavía en el diagrama UML de clases, y varios diagramas requerían ajustes visuales para mejorar su legibilidad al exportarlos.
- **Soluciones aplicadas:** Se actualizó el diagrama de clases para mantener coherencia estricta con las firmas usadas en los diagramas de secuencia y se dedicó la reunión de cierre del 24/11/2026 a pulir la legibilidad, las leyendas y la documentación final con margen antes de la entrega del 27/11/2026.

#### Estado del tablero en el Sprint Planning

![][image5]

#### Estado del tablero en la Sprint Retrospective

![][image6]

### 3.1. Reuniones realizadas

Todas las reuniones fueron presenciales y contaron con la participación de los seis miembros del equipo.

| N.º | Fecha      | Duración   | Participantes    | Temas abordados                                                                                                                                                                                   |
| :-- | :--------- | :--------- | :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | 28/10/2026 | 1 h 45 min | Equipo y cliente | Revisión de dudas pendientes de la Entrega 1; configuración del tablero en Trello; _Sprint Planning_ del Sprint 4 y reparto inicial.                                                              |
| 2   | 04/11/2026 | 1 h 45 min | Equipo           | Revisión de las actualizaciones de la Entrega 1 y del diagrama físico E-R; _Sprint Retrospective_ del Sprint 4 y _Sprint Planning_ del Sprint 5.                                                  |
| 3   | 11/11/2026 | 1 h 45 min | Equipo           | Alineación entre diagrama físico E-R y UML de clases; revisión del Bloque 1 de secuencias; _Sprint Retrospective_ del Sprint 5 y _Sprint Planning_ del Sprint 6.                                  |
| 4   | 24/11/2026 | 2 h        | Equipo           | Revisión cruzada de diagramas de secuencia (Bloque 2), clases y arquitectura/componentes; comprobación final en `main`, elaboración del informe Scrum y preparación de la Entrega 2 (27/11/2026). |

## 4. Problemas y riesgos

| Sprint   | Problema o riesgo                                                                                                                  | Impacto                                                         | Solución aplicada                                                                                                                           | Estado                                       |
| :------- | :--------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------- |
| Sprint 4 | Riesgo de conflictos de edición al continuar trabajando todos los miembros directamente sobre la rama `main` de GitHub.            | Posibles sobrescrituras o conflictos al subir cambios a la vez. | Se repartieron archivos independientes por parejas y se acordó realizar `git pull` previo y avisar antes de modificar ficheros compartidos. | Mitigado (pendiente adoptar ramas en GitHub) |
| Sprint 5 | Incoherencias iniciales entre el diagrama físico E-R (tablas/claves) y el diagrama UML de clases (atributos/relaciones).           | Riesgo de entregar un diseño inconsistente entre capas.         | Revisión cruzada entre responsables y unificación de nomenclatura, tipos de datos y cardinalidades.                                         | Resuelto                                     |
| Sprint 5 | Heterogeneidad en el nivel de detalle de los diagramas UML de secuencia elaborados por distintos miembros.                         | Falta de uniformidad y dificultad para seguir los flujos.       | Se definió una plantilla y convenciones comunes en PlantUML (participantes, bloques `alt`/`opt`/`loop` y mensajes de retorno).              | Resuelto                                     |
| Sprint 6 | Desajuste entre algunos métodos invocados en los diagramas de secuencia y las operaciones declaradas en el diagrama UML de clases. | Inconsistencia en la trazabilidad del diseño detallado.         | Auditoría final de firmas entre todos los diagramas de secuencia y el diagrama de clases antes de exportar las versiones definitivas.       | Resuelto                                     |

## 5. Valoración final

**Qué se ha conseguido.** El equipo ha completado y entregado todos los objetivos técnicos de la Entrega 2: el **diagrama físico Entidad-Relación**, el **diagrama UML de clases**, los **diagramas UML de secuencia** y el **diagrama de arquitectura/componentes**, además de la actualización de la primera entrega. Todos los artefactos son coherentes entre sí y trazables respecto a los requisitos y casos de uso. Asimismo, se ha consolidado el uso efectivo de Trello como tablero Kanban y un historial de commits más descriptivo en GitHub.

**Qué ha funcionado bien.**

- La migración a **Trello**, que ha dado mayor visibilidad al estado real de cada tarea y a los responsables durante los tres sprints.
- El reparto de trabajo en paralelo por parejas y artefactos de diseño, evitando cuellos de botella y mejorando el aprovechamiento del tiempo.
- El acuerdo temprano de convenciones y plantillas comunes, que redujo considerablemente el esfuerzo de unificación al final.
- Las revisiones cruzadas entre subgrupos antes de mover las tarjetas a _Hecho_ en Trello, detectando incoherencias a tiempo.
- La reunión presencial de cierre del 24/11/2026, celebrada con margen previo a la entrega del 27/11/2026, para pulir detalles de legibilidad y documentación sin prisas de última hora.

**Qué se puede mejorar.**

- Adoptar finalmente el trabajo con ramas independientes y _Pull Requests_ en GitHub, ya que en esta entrega se ha seguido trabajando únicamente en la rama `main`.
- Realizar validaciones de coherencia entre diagramas (especialmente clases y secuencia) de forma más frecuente y no concentrar parte de ellas en el último sprint.
- Estimar con mayor precisión el esfuerzo de los diagramas más extensos (como los de secuencia con múltiples flujos alternativos) para equilibrar aún mejor la carga.
- Mantener la documentación explicativa al día a medida que se cierra cada diagrama, en lugar de redactar parte de ella al final del último sprint.

**Acciones para la siguiente entrega.**

- Mantener el tablero Kanban en Trello e implantar definitivamente el flujo de trabajo con ramas por tarea y _Pull Requests_ revisadas antes de fusionar en `main`.
- Tomar el diagrama físico E-R, el diagrama de clases y la arquitectura aprobados como base directa para la siguiente fase del proyecto, registrando cualquier cambio de diseño en el repositorio.
- Incorporar _checklists_ de revisión en las tarjetas de Trello para validar trazabilidad y calidad antes de dar una tarea por terminada.

[image1]: ./img/informe-scrum-entrega2-img1.png
[image2]: ./img/informe-scrum-entrega2-img2.png
[image3]: ./img/informe-scrum-entrega2-img3.png
[image4]: ./img/informe-scrum-entrega2-img4.png
[image5]: ./img/informe-scrum-entrega2-img5.png
[image6]: ./img/informe-scrum-entrega2-img6.png
