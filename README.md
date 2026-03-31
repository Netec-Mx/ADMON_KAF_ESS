# Administración de Kafka

Este curso tiene como propósito entregar a los participantes los conocimientos teóricos y prácticos necesarios para administrar de forma profesional plataformas de mensajería construidas sobre Apache Kafka 4.0.0 y Confluent Platform 8.0, cubriendo tanto despliegues locales como escenarios en la nube.

Durante el desarrollo del programa, se abordarán los fundamentos de Kafka moderno, basado en el modo KRaft (Kafka Raft Metadata mode), que elimina la necesidad de Zookeeper y simplifica la arquitectura del clúster. Se estudiarán sus principales componentes —tópicos, particiones, productores, consumidores, brokers y controladores—, así como la administración del ciclo de vida de los datos, configuración de retención, políticas de replicación y balanceo de carga.

Además, se explorarán herramientas clave del ecosistema Confluent, como Control Center, Kafka Connect, ksqlDB, Schema Registry y Confluent CLI, permitiendo a los alumnos interactuar con flujos de datos en tiempo real, definir integraciones y gestionar esquemas de datos estructurados. También se presentará la opción de trabajar con Confluent Cloud, facilitando la transición desde entornos locales hacia arquitecturas escalables y productivas en la nube.

El enfoque del curso es altamente práctico. Cada módulo incluye laboratorios guiados en los que los participantes desplegarán servicios con Docker Compose, realizarán operaciones reales sobre clústeres Kafka, observarán el comportamiento del sistema frente a fallos y simularán escenarios propios de un entorno empresarial. Al finalizar, los alumnos serán capaces de diseñar, desplegar, administrar y monitorear una solución de mensajería robusta, segura y alineada con los estándares actuales de arquitectura orientada a eventos.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Laboratorio 1: Respecto al contenido del capitulo 1](Capitulo01/README.md#laboratorio-1-respecto-al-contenido-del-capitulo-1)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 168 min

### Capítulo 2

- [Laboratorio 2: Respecto al contenido del capitulo 2](Capitulo02/README.md#laboratorio-2-respecto-al-contenido-del-capitulo-2)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 168 min

### Capítulo 3

- [Laboratorio 3: Respecto al contenido del capitulo 3](Capitulo03/README.md#laboratorio-3-respecto-al-contenido-del-capitulo-3)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 168 min

### Capítulo 4

- [Laboratorio 4: Conector JDBC Source (Base de datos → Kafka)](Capitulo04/README.md#laboratorio-4-conector-jdbc-source-base-de-datos-kafka)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 38 min
- [Laboratorio 5: Conector JDBC Sink (Kafka → Base de datos)](Capitulo04/README.md#laboratorio-5-conector-jdbc-sink-kafka-base-de-datos)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 38 min
- [Laboratorio 6: Consultas en tiempo real con ksqlDB](Capitulo04/README.md#laboratorio-6-consultas-en-tiempo-real-con-ksqldb)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 42 min
- [Laboratorio 7: Evaluación final y cierre del curso](Capitulo04/README.md#laboratorio-7-evaluación-final-y-cierre-del-curso)
  - Descripción: Actividad práctica guiada basada en el contenido del módulo.
  - Duración estimada: 50 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
