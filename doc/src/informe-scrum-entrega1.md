# Informe Scrum: Entrega 1

**Proyecto:** Sistema interno de facturación y cobros de TurbineH  
**Asignatura:** Análisis y Diseño de Aplicaciones (curso 2026-27)  
**Equipo:** Hienitas  
**Entrega:** Análisis de requisitos y casos de uso (23/10/2026)

## 1\. Identificación del equipo

| Nombre y apellidos                   | Rol                |
| :----------------------------------- | :----------------- |
| Pablo Ruiz Valdés                    | Scrum Master       |
| Adrián Pérez Pacheco                 | Miembro del equipo |
| Pedro Javier Ortega Fernández        | Miembro del equipo |
| Sergio Cañas Vegas                   | Miembro del equipo |
| Jose Fernández Ocon                  | Miembro del equipo |
| Sergio Maximiliano Rodríguez Donaire | Miembro del equipo |

## 2\. Organización del trabajo

El equipo ha trabajado con una adaptación de Scrum en tres sprints, uno por cada sesión de prácticas. Al inicio de cada sprint se realizó una reunión de planificación (Sprint Planning) para seleccionar las tareas y, al final, una revisión y retrospectiva (Sprint Review y Sprint Retrospective).

- **Gestión de tareas:** GitHub Projects, con Product Backlog, Sprint Backlog, responsable, estado, prioridad, milestone y etiquetas de tipo (requisito, diagrama, documentación, revisión). Sin embargo, para esta entrega no se ha implementado por dificultades técnicas del equipo, para los tableros se ha utilizado un documento compartido en el que los integrantes hemos ido apuntando y marcando las tareas completadas.
- **Control de versiones:** repositorio público en GitHub ([`Hienitas-ADAP`](https://github.com/pablorv3/Hienitas-ADAP/)), con la estructura indicada en la guía (doc/, doc/src/, README.md, .gitignore). Cada miembro ha trabajado con commits propios y descriptivos. Además, se cuenta con un Drive usado por cada miembro para almacenar borradores y versiones antiguas, así como para compartir archivos de manera simple.
- **Reparto de responsabilidades:** el documento de requisitos se elaboró de forma conjunta y los casos de uso se repartieron por bloques entre los miembros (especificación y diagrama de cada caso). El Scrum Master coordinó las reuniones, el tablero y las entregas.
- **Reuniones:** 5 reuniones presenciales del equipo durante el periodo. La primera, el día de presentación del problema. Tres de ellas durante las sesiones prácticas y una el día anterior a la entrega 1\.
- **Documentación de las fuentes:** el equipo ha distinguido entre lo dicho por el cliente, las decisiones de análisis, las suposiciones y las dudas pendientes, y las ha recogido en el documento de requisitos (secciones 6 y 7).

## 3\. Sprints realizados

### Sprint 1

- **Fecha de inicio:** 29/09/2026 (presentación del caso por el cliente)
- **Fecha de finalización:** 07/10/2026
- **Objetivo del Sprint:** Comprender el problema del cliente y preparar la base de trabajo del equipo, incluyendo un primer borrador de los requisitos.
- **Tareas planificadas:**
  - Tomar notas de la presentación del cliente y analizar la transcripción.
  - Leer la guía operativa del proyecto.
  - Estudiar y analizar los requisitos funcionales y no funcionales.
  - Crear el repositorio y la estructura inicial de carpetas.
  - Configurar el tablero de GitHub Projects.
  - Elaborar un primer borrador del documento de requisitos.
- **Tareas completadas:** Todas las planificadas. El repositorio con su estructura inicial se creó el 07/10/2026 y el borrador del documento de requisitos se revisó en equipo.
- **Reuniones realizadas:** Reunión 1 (29/09/2026, 1h 45), Reunión 2 (30/09/2026, 1 h 45 min), véase el apartado 3.1.
- **Problemas encontrados:** La información del cliente estaba dispersa en la transcripción y aparecía descrita la misma funcionalidad en varios momentos; además, quedaron dudas sin cerrar.
- **Soluciones aplicadas:** Se ordenaron las ideas por temas y se anotaron las dudas para resolverlas más adelante.

#### Estado del tablero en el Sprint Planning

![][image1]

#### Estado del tablero en la Sprint Retrospective

![][image2]

&nbsp;

### Sprint 2

- **Fecha de inicio:** 07/10/2026
- **Fecha de finalización:** 21/10/2026
- **Objetivo del Sprint:** Elaborar el documento de requisitos conforme a la plantilla de la guía.
- **Tareas planificadas:**
  - Reformular y descomponer los requisitos en diferentes grupos.
  - Añadir los nuevos requisitos funcionales aportados por el cliente durante la segunda entrevista.
  - Definir actores, reglas de negocio, suposiciones y dudas.
  - Elaborar la trazabilidad y el glosario.
  - Subir el documento de requisitos a doc/ y doc/src/.
- **Tareas completadas:** Todas las planificadas. La primera versión del documento de requisitos según la plantilla aportada por los profesores se subió al repositorio el 21/10/2026.
- **Reuniones realizadas:** Reunión 2 (07/10/2026, 1 h 45 min), en la que se planteó el sprint, y reunión 3 (21/10/2026, 1 h 45 min), de cierre; véase el apartado 3.1.
- **Problemas encontrados:** Hubo que adaptar el documento original de requisitos a la plantilla, además el cliente presentó requisitos nuevos y algunas de las dudas del cliente seguían sin cerrar (véase el apartado 4).
- **Soluciones aplicadas:** Se reformularon los antiguos requisitos, se estudiaron los nuevos _y_ se conservó una correspondencia con el documento original y borradores anteriores(Anexo A)_._ Las dudas se registraron en el apartado de cuestiones pendientes con una interpretación provisional.

#### Estado del tablero en el Sprint Planning

![][image3]

#### Estado del tablero en la Sprint Retrospective

![][image4]

### Sprint 3

- **Fecha de inicio:** 21/10/2026
- **Fecha de finalización:** 23/10/2026
- **Objetivo del Sprint:** Especificar y modelar los casos de uso y preparar la entrega.
- **Tareas planificadas:**
  - Definir la lista de casos de uso principales.
  - Redactar la especificación de los casos de uso.
  - Elaborar el diagrama UML de cada caso de uso en PlantUML (doc/src/casos-uso/).
  - Unificar el formato y los nombres de los archivos.
  - Elaborar el informe Scrum y la evaluación interna.
- **Tareas completadas:** Todas las planificadas. Los diagramas de los casos de uso se subieron al repositorio entre el 22/10/2026 y el 23/10/2026.
- **Reuniones realizadas:** Reunión 4 (21/10/2026, 1 h 45 min), de planificación, y reunión 5 (23/10/2026, 2 h), de cierre y fuera del horario de clase; véase el apartado 3.1.
- **Problemas encontrados:** Coordinar el reparto de los casos de uso entre varios miembros, con todo el grupo centrado en el mismo tema, y mantener la coherencia con los requisitos.
- **Soluciones aplicadas:** Se acordó una plantilla común para los diagramas y un criterio de nombres (uc-XX-nombre.pu), y se revisó la trazabilidad con los requisitos.

#### Estado del tablero en el Sprint Planning

![][image5]

#### Estado del tablero en la Sprint Retrospective

![][image6]

### 3.1. Reuniones realizadas

Todas las reuniones fueron presenciales y participaron los seis miembros del equipo.

| N.º | Fecha      | Duración   | Participantes    | Temas abordados                                                                                                                |
| :-- | :--------- | :--------- | :--------------- | :----------------------------------------------------------------------------------------------------------------------------- |
| 1   | 29/09/2026 | 1h 45 min  | Equipo y cliente | Presentación del caso por el cliente. Elaboración de borradores.                                                               |
| 2   | 30/09/2026 | 1 h 45 min | Equipo           | Estudio y análisis de los requisitos funcionales y no funcionales.                                                             |
| 3   | 07/10/2026 | 1 h 45 min | Equipo           | Revisión del documento de requisitos y planteamiento del Sprint 2\. Ampliación del documento de requisitos.                    |
| 4   | 21/10/2026 | 1 h 45 min | Equipo           | Sprint Planning del Sprint 3; desarrollo de las especificaciones de los casos de uso.                                          |
| 5   | 23/10/2026 | 2 h        | Equipo           | Revisión de las especificaciones de los casos de uso, desarrollo retoque de los diagramas UML y elaboración del informe Scrum. |

4\. Problemas y riesgos

| Sprint   | Problema o riesgo                                                                                                                                         | Impacto                                     | Solución aplicada                                                                                                                                               | Estado                                                    |
| :------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------- |
| Sprint 1 | Dudas del cliente sin cerrar (por ejemplo, quién configura las series de facturación o si la baja de un cliente se bloquea con facturas pendientes).      | Riesgo de modelar mal algunos casos de uso. | Se registraron en el apartado de cuestiones pendientes con una interpretación provisional, para preguntarlas al cliente en la siguiente revisión.               | Pendiente (se aplican las interpretaciones provisionales) |
| Sprint 2 | Hubo que reformular el documento original de requisitos y adaptarlo a la plantilla (requisitos atómicos y separación entre funcionales y no funcionales). | Retraso en la elaboración del documento.    | Se reformuló y se descompuso, conservando una correspondencia con el documento original y borradores anteriores (Anexo A) para comprobar que no se omitía nada. | Resuelto                                                  |

5\. Valoración final

**Qué se ha conseguido.** El equipo ha entregado el documento de requisitos conforme a la plantilla, el diagrama UML y la especificación de los diferentes casos de uso. Además, mantiene un repositorio organizado y con un historial de commits progresivo.

**Qué ha funcionado bien.**

- La organización en tres sprints, uno por sesión de prácticas, con reuniones presenciales.
- El uso de GitHub (repositorio y tablero) para repartir el trabajo y seguir el avance.
- El registro explícito de suposiciones y dudas del cliente.
- La reunión final, fuera del horario de clase, para cerrar la primera entrega, corrigiendo los detalles finales.
- Una organización equilibrada y acordada del trabajo entre los miembros del grupo.

**Qué se puede mejorar.**

- Hacer uso de las herramientas mencionadas para los tableros de los Sprint Planning y Backlog en lugar de usar documentos.
- Cerrar antes las dudas con el cliente para evitar interpretaciones provisionales.
- Acordar antes el formato común de los diagramas y plantillas para reducir el trabajo de unificación.
- Repartir mejor las tareas, evitando centrar a todo el grupo en un mismo tema durante el mismo sprint.
- Realizar commits más descriptivos y controlados. Así como el uso de ramas independientes para la modificación y posterior pull request.

**Acciones para la Entrega 2\.**

- Migrar inmediatamente toda la gestión de tareas a un tablero Kanban formal (GitHub Projects) o parecido.
- Revisar las dudas pendientes con el cliente y actualizar los requisitos y casos de uso si cambian.
- Repartir con tiempo el diagrama Entidad-Relación, el diagrama de clases, los diagramas de secuencia y el de arquitectura.
- Mantener la trazabilidad entre requisitos, casos de uso y diseño.

[image1]: ./img/informe-scrum-entrega1-img1.png
[image2]: ./img/informe-scrum-entrega1-img2.png
[image3]: ./img/informe-scrum-entrega1-img3.png
[image4]: ./img/informe-scrum-entrega1-img4.png
[image5]: ./img/informe-scrum-entrega1-img5.png
[image6]: ./img/informe-scrum-entrega1-img6.png
