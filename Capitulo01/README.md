# Fundamentos de Mensajería Basada en Eventos y Arquitectura Kafka

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 168 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |
| **Módulo** | Capítulo 1 — Introducción a la Mensajería Basada en Eventos |

## Descripción General

Este laboratorio es de naturaleza exploratoria y conceptual, diseñado para consolidar la comprensión teórica del Capítulo 1 mediante actividades prácticas de análisis, reflexión y diseño. Los participantes explorarán los componentes fundamentales de la arquitectura de Apache Kafka 4.0, trazarán el flujo completo de un mensaje desde su producción hasta su consumo, y analizarán las diferencias entre el modo KRaft y la arquitectura tradicional con ZooKeeper.

El laboratorio tiene relevancia práctica inmediata: antes de escribir una sola línea de código o ejecutar un comando de Kafka, es fundamental que el arquitecto o desarrollador comprenda *por qué* existe cada componente y *qué problema resuelve*. Las decisiones de diseño que se toman en esta etapa conceptual tienen impacto directo en la escalabilidad, resiliencia y mantenibilidad de los sistemas en producción.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Identificar y describir los componentes principales de la arquitectura de Apache Kafka 4.0 y el rol específico de cada uno en un sistema de mensajería basado en eventos.
- [ ] Distinguir las diferencias arquitectónicas entre el modo KRaft (sin ZooKeeper) y la arquitectura tradicional con ZooKeeper, argumentando las ventajas del nuevo modelo.
- [ ] Relacionar el modelo de publicación/suscripción de Kafka con casos de uso reales de sistemas distribuidos modernos como los de LinkedIn, Netflix y Uber.
- [ ] Reconocer los componentes adicionales de la Confluent Platform 8.0 y su valor agregado sobre el Apache Kafka base.
- [ ] Diseñar una arquitectura Kafka básica para un escenario de negocio dado, justificando las decisiones de diseño con base en los conceptos del capítulo.

## Prerrequisitos

### Conocimiento Requerido

- Comprensión general de sistemas de mensajería asíncrona y colas de mensajes.
- Familiaridad con conceptos de sistemas distribuidos como tolerancia a fallos y escalabilidad horizontal.
- Conocimiento básico de redes TCP/IP y comunicación cliente-servidor.
- Haber leído o revisado el material teórico del Capítulo 1 del curso, específicamente la Lección 1.1 sobre mensajería basada en eventos.

### Acceso Requerido

- Acceso a un editor de texto o procesador de documentos (puede ser cualquier editor: VS Code, Notepad++, LibreOffice Writer, o incluso papel y lápiz para los diagramas).
- Acceso a internet para consultar documentación de referencia (opcional pero recomendado).
- Directorio de trabajo local con permisos de escritura para guardar los archivos del laboratorio.

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| Procesador | Cualquier procesador moderno de 64 bits |
| Memoria RAM | Mínimo 2 GB disponibles (laboratorio conceptual, sin contenedores) |
| Espacio en disco | 50 MB para archivos de trabajo del laboratorio |
| Conexión a internet | Opcional (para consultar documentación de referencia) |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Editor de texto | Cualquier versión reciente | Crear y editar los archivos de análisis y diseño del laboratorio |
| Navegador web | Cualquier versión moderna | Consultar documentación de referencia de Confluent y Apache Kafka |
| Terminal / Consola | Bash, Zsh, PowerShell o CMD | Crear la estructura de directorios del laboratorio |

### Configuración Inicial

Crea la estructura de directorios de trabajo para este laboratorio ejecutando los siguientes comandos en tu terminal:

```bash
# Crear directorio principal del laboratorio
mkdir -p ~/kafka-labs/lab-01-00-01
cd ~/kafka-labs/lab-01-00-01

# Crear subdirectorios para cada actividad
mkdir -p actividad-1-componentes
mkdir -p actividad-2-flujo-mensajes
mkdir -p actividad-3-kraft-vs-zookeeper
mkdir -p actividad-4-casos-de-uso
mkdir -p actividad-5-diseno-arquitectura
mkdir -p actividad-6-confluent-platform

# Verificar la estructura creada
find ~/kafka-labs/lab-01-00-01 -type d
```

**Salida esperada:**

```
/home/usuario/kafka-labs/lab-01-00-01
/home/usuario/kafka-labs/lab-01-00-01/actividad-1-componentes
/home/usuario/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes
/home/usuario/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper
/home/usuario/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso
/home/usuario/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura
/home/usuario/kafka-labs/lab-01-00-01/actividad-6-confluent-platform
```

## Instrucciones Paso a Paso

### Paso 1: Mapeo de Componentes Fundamentales de Kafka

**Objetivo:** Identificar y documentar los componentes principales de la arquitectura de Apache Kafka 4.0, describiendo el rol específico de cada uno y las relaciones entre ellos.

**Instrucciones:**

1. Navega al directorio de la primera actividad:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-1-componentes
   ```

2. Crea el archivo de análisis de componentes con el siguiente comando. Este archivo será tu documento de trabajo para esta actividad:

   ```bash
   cat > componentes-kafka.md << 'EOF'
   # Análisis de Componentes de Apache Kafka 4.0
   
   ## Instrucciones
   Para cada componente listado a continuación, completa:
   - Descripción con tus propias palabras (2-3 oraciones)
   - Responsabilidad principal en el sistema
   - Analogía del mundo real que lo represente
   - Con qué otros componentes interactúa directamente
   
   ---
   
   ## Componente 1: Producer (Productor)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 2: Broker (Servidor Kafka)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 3: Consumer (Consumidor)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 4: Topic (Tópico)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 5: Partition (Partición)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 6: Consumer Group (Grupo de Consumidores)
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Componente 7: Offset
   **Descripción:**
   [COMPLETAR]
   
   **Responsabilidad principal:**
   [COMPLETAR]
   
   **Analogía del mundo real:**
   [COMPLETAR]
   
   **Interactúa con:**
   [COMPLETAR]
   
   ---
   
   ## Reflexión Final
   ¿Cuál componente consideras más crítico para la confiabilidad del sistema y por qué?
   [COMPLETAR - mínimo 3 oraciones]
   
   EOF
   ```

3. Abre el archivo con tu editor de texto preferido y completa cada sección. Usa como referencia el material del Capítulo 1 y, si lo deseas, la documentación oficial:

   ```bash
   # En Linux/macOS con VS Code
   code componentes-kafka.md
   
   # Alternativa con nano (disponible en casi todos los sistemas Linux)
   nano componentes-kafka.md
   
   # En Windows con Notepad (desde PowerShell)
   # notepad componentes-kafka.md
   ```

4. Una vez completado, crea un archivo de respuestas de referencia para autoevaluación. Este archivo contiene las descripciones correctas que deberás comparar con las tuyas:

   ```bash
   cat > respuestas-referencia-componentes.md << 'EOF'
   # Respuestas de Referencia — Componentes Kafka 4.0
   
   ## Producer (Productor)
   Aplicación cliente que genera y envía mensajes (eventos) a un tópico de Kafka.
   El productor decide en qué partición publicar cada mensaje (usando una clave,
   round-robin, o una función de particionamiento personalizada). No necesita
   conocer a los consumidores de sus mensajes.
   Analogía: El periodista que escribe y envía artículos a la editorial.
   
   ## Broker
   Servidor Kafka que recibe mensajes de los productores, los almacena en disco
   de forma durable y los sirve a los consumidores. Un clúster Kafka típico tiene
   3 o más brokers para tolerancia a fallos. En Kafka 4.0 con KRaft, uno de los
   brokers también actúa como controlador del clúster.
   Analogía: La editorial que recibe, almacena y distribuye los periódicos.
   
   ## Consumer (Consumidor)
   Aplicación cliente que lee mensajes de uno o más tópicos de Kafka. Los
   consumidores llevan el registro de qué mensajes ya procesaron mediante el
   offset. Pueden releer mensajes anteriores si es necesario, lo cual es una
   característica única de Kafka frente a colas tradicionales.
   Analogía: El lector que se suscribe al periódico y lo lee a su propio ritmo.
   
   ## Topic (Tópico)
   Canal lógico con nombre al que los productores publican mensajes y del que
   los consumidores leen. Es análogo a una categoría o feed de noticias. Un
   tópico se divide en particiones para permitir paralelismo y escalabilidad.
   Analogía: La sección de un periódico (Deportes, Economía, Internacional).
   
   ## Partition (Partición)
   Subdivisión ordenada e inmutable de un tópico. Cada partición es una secuencia
   de mensajes ordenada por offset. Las particiones permiten distribuir la carga
   entre múltiples brokers y habilitan el procesamiento paralelo por múltiples
   consumidores dentro de un mismo grupo.
   Analogía: Las diferentes imprentas que producen ejemplares del mismo periódico.
   
   ## Consumer Group (Grupo de Consumidores)
   Conjunto de consumidores que colaboran para leer un tópico. Cada partición
   es asignada a exactamente un consumidor dentro del grupo, garantizando que
   cada mensaje sea procesado una sola vez por el grupo. Múltiples grupos pueden
   leer el mismo tópico de forma independiente (modelo pub/sub).
   Analogía: El equipo de distribución donde cada repartidor cubre una zona.
   
   ## Offset
   Número entero secuencial que identifica de forma única cada mensaje dentro
   de una partición. Los consumidores usan el offset para saber hasta dónde han
   leído. Kafka almacena los offsets de los grupos de consumidores en el tópico
   interno __consumer_offsets.
   Analogía: El número de página donde el lector dejó de leer el periódico.
   
   EOF
   ```

5. Compara tu archivo completado con las respuestas de referencia y anota las diferencias:

   ```bash
   # Ver ambos archivos lado a lado (si tienes diff disponible)
   diff componentes-kafka.md respuestas-referencia-componentes.md
   
   # O simplemente abre ambos archivos en tu editor
   echo "Archivos disponibles para revisión:"
   ls -la ~/kafka-labs/lab-01-00-01/actividad-1-componentes/
   ```

**Salida Esperada:**

```
Archivos disponibles para revisión:
total 24
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 2847 [fecha] componentes-kafka.md
-rw-r--r-- 1 usuario usuario 3156 [fecha] respuestas-referencia-componentes.md
```

**Verificación:**

- [ ] El archivo `componentes-kafka.md` existe y tiene todos los campos completados (sin ningún `[COMPLETAR]` restante).
- [ ] Puedes describir con tus propias palabras la diferencia entre un tópico y una partición.
- [ ] Puedes explicar por qué el offset es fundamental para la confiabilidad del sistema.

---

### Paso 2: Trazado del Flujo Completo de un Mensaje

**Objetivo:** Trazar paso a paso el recorrido de un mensaje desde que es producido hasta que es consumido, identificando cada componente involucrado y las decisiones que ocurren en cada etapa.

**Instrucciones:**

1. Navega al directorio de la segunda actividad:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes
   ```

2. Crea el archivo de escenario de negocio que usarás como contexto para trazar el flujo:

   ```bash
   cat > escenario-ecommerce.md << 'EOF'
   # Escenario de Negocio: Plataforma de E-Commerce
   
   ## Contexto
   Una plataforma de comercio electrónico procesa órdenes de compra.
   Cuando un usuario confirma una compra, los siguientes sistemas deben reaccionar:
   
   1. Sistema de Pagos: Procesar el cobro de la tarjeta
   2. Sistema de Inventario: Reducir el stock del producto
   3. Sistema de Notificaciones: Enviar email de confirmación al usuario
   4. Sistema de Logística: Crear la orden de envío
   5. Sistema de Analytics: Registrar la venta para reportes
   
   ## Configuración Kafka para este escenario
   - Tópico: "ordenes-confirmadas"
   - Número de particiones: 6
   - Factor de replicación: 3
   - Clúster: 3 brokers (broker-1, broker-2, broker-3)
   - Consumer Groups:
     * grupo-pagos (1 consumidor)
     * grupo-inventario (2 consumidores)
     * grupo-notificaciones (1 consumidor)
     * grupo-logistica (3 consumidores)
     * grupo-analytics (1 consumidor)
   
   ## Mensaje de ejemplo
   {
     "evento": "OrdenConfirmada",
     "orden_id": "ORD-2024-78542",
     "usuario_id": "USR-1042",
     "producto_id": "PROD-987",
     "cantidad": 2,
     "monto_total": 150.00,
     "timestamp": "2024-01-15T14:32:00Z"
   }
   
   EOF
   ```

3. Crea el archivo de trazado del flujo donde documentarás cada etapa:

   ```bash
   cat > flujo-mensaje-kafka.md << 'EOF'
   # Trazado del Flujo de un Mensaje en Kafka
   
   ## Escenario: Orden Confirmada en E-Commerce
   
   Traza el recorrido completo del mensaje de ejemplo desde que el sistema
   de órdenes lo produce hasta que todos los consumidores lo procesan.
   Para cada etapa, responde las preguntas indicadas.
   
   ---
   
   ## ETAPA 1: El Productor Genera el Evento
   
   **¿Qué sistema actúa como productor en este escenario?**
   [COMPLETAR]
   
   **¿Qué información incluye el mensaje que se va a publicar?**
   [COMPLETAR]
   
   **¿A qué tópico publica el mensaje?**
   [COMPLETAR]
   
   **¿Cómo decide el productor en qué partición publicar el mensaje?
   (Pista: el mensaje tiene orden_id como clave posible)**
   [COMPLETAR]
   
   ---
   
   ## ETAPA 2: El Broker Recibe y Almacena el Mensaje
   
   **¿Cuál de los 3 brokers recibe el mensaje inicialmente?
   (Pista: investiga el concepto de "broker líder" para una partición)**
   [COMPLETAR]
   
   **¿Qué hace el broker con el mensaje antes de confirmarlo al productor?**
   [COMPLETAR]
   
   **¿Qué es la replicación y cómo protege este mensaje de una falla?**
   [COMPLETAR]
   
   **¿Qué offset se asigna al mensaje y dónde se almacena esa información?**
   [COMPLETAR]
   
   ---
   
   ## ETAPA 3: Los Consumidores Leen el Mensaje
   
   **¿Cuántos Consumer Groups leerán este mismo mensaje?**
   [COMPLETAR]
   
   **¿El mensaje se "elimina" del tópico después de que el primer grupo lo lee?
   Explica por qué sí o por qué no.**
   [COMPLETAR]
   
   **Si el grupo-inventario tiene 2 consumidores y el tópico tiene 6 particiones,
   ¿cuántas particiones le corresponden a cada consumidor del grupo?**
   [COMPLETAR]
   
   **¿Qué pasa si el consumidor del grupo-pagos falla a mitad del procesamiento?
   ¿Cómo sabe Kafka que debe reenviarle el mensaje?**
   [COMPLETAR]
   
   ---
   
   ## ETAPA 4: Comparación con Comunicación Síncrona
   
   **Si este mismo escenario se implementara con llamadas REST síncronas
   (el sistema de órdenes llama a cada sistema uno por uno), ¿qué problemas
   podrían ocurrir? Lista al menos 3 problemas concretos.**
   
   Problema 1: [COMPLETAR]
   Problema 2: [COMPLETAR]
   Problema 3: [COMPLETAR]
   
   **¿Cuál de los 5 sistemas consumidores se beneficia MÁS del modelo asíncrono
   y por qué? (Piensa en cuál es el más lento o menos crítico para el usuario)**
   [COMPLETAR]
   
   ---
   
   ## ETAPA 5: Diagrama ASCII del Flujo
   
   Dibuja un diagrama ASCII simple que muestre el flujo completo.
   Ejemplo de formato sugerido (modifícalo según tu comprensión):
   
   [Sistema Órdenes]
        |
        | publica evento "OrdenConfirmada"
        v
   [Tópico: ordenes-confirmadas]
        |
        +---> [COMPLETAR: grupo y sistema]
        |
        +---> [COMPLETAR: grupo y sistema]
        |
        +---> [COMPLETAR: grupo y sistema]
        |
        +---> [COMPLETAR: grupo y sistema]
        |
        +---> [COMPLETAR: grupo y sistema]
   
   EOF
   ```

4. Completa el archivo con tus respuestas usando tu editor:

   ```bash
   nano flujo-mensaje-kafka.md
   ```

5. Crea un archivo de validación con las respuestas correctas para autoevaluación:

   ```bash
   cat > validacion-flujo.md << 'EOF'
   # Validación — Flujo de Mensaje en Kafka
   
   ## Respuestas Clave
   
   ### Etapa 1
   - Productor: El servicio de gestión de órdenes (Order Service)
   - Partición: Con orden_id como clave, Kafka aplica hash(clave) % num_particiones
     para determinar la partición. Esto garantiza que todas las órdenes del mismo
     ID siempre vayan a la misma partición (orden garantizado por orden).
   
   ### Etapa 2
   - El broker LÍDER de la partición recibe el mensaje primero.
   - Antes de confirmar (ACK) al productor, el broker escribe en su log local.
     Con acks=all, espera que todos los brokers réplica también confirmen.
   - Replicación: Los otros 2 brokers mantienen copias del mensaje.
     Si el líder falla, uno de los seguidores se convierte en nuevo líder.
   - El offset es un número secuencial (ej: offset 1547) asignado por el broker
     líder. Se almacena en el log de la partición en disco.
   
   ### Etapa 3
   - 5 Consumer Groups leen el mismo mensaje de forma independiente.
   - NO se elimina. Kafka retiene los mensajes según la política de retención
     configurada (por tiempo o tamaño), independientemente de si fueron leídos.
   - 6 particiones / 2 consumidores = 3 particiones por consumidor.
   - Kafka lleva el registro del último offset confirmado (committed offset).
     Si el consumidor falla sin confirmar, al reiniciarse retoma desde el último
     offset confirmado y reprocesa el mensaje.
   
   ### Etapa 4
   - Problema 1: Si el sistema de analytics está lento, bloquea la respuesta
     al usuario aunque no sea crítico para completar la orden.
   - Problema 2: Si cualquier sistema falla, la orden completa puede fallar
     o quedar en estado inconsistente.
   - Problema 3: Escalar requiere escalar todos los sistemas simultáneamente.
   - Sistema que más se beneficia: Analytics, porque no es crítico para el
     usuario y puede procesar en batch sin afectar la experiencia de compra.
   
   EOF
   ```

**Salida Esperada:**

```
# Al listar los archivos del directorio:
ls -la ~/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes/

total 20
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 1024 [fecha] escenario-ecommerce.md
-rw-r--r-- 1 usuario usuario 3072 [fecha] flujo-mensaje-kafka.md
-rw-r--r-- 1 usuario usuario 2048 [fecha] validacion-flujo.md
```

**Verificación:**

- [ ] Puedes explicar por qué el mensaje NO se elimina después de ser leído por el primer consumidor.
- [ ] Puedes calcular correctamente la distribución de particiones entre consumidores de un grupo.
- [ ] Puedes enumerar al menos 3 problemas concretos que surgen con la comunicación síncrona en este escenario.

---

### Paso 3: Análisis Comparativo KRaft vs ZooKeeper

**Objetivo:** Comprender las diferencias arquitectónicas entre el modo KRaft (Kafka Raft Metadata mode) introducido en Kafka 4.0 y la arquitectura tradicional que dependía de ZooKeeper, identificando las ventajas del nuevo modelo.

**Instrucciones:**

1. Navega al directorio de la tercera actividad:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper
   ```

2. Crea el archivo de análisis comparativo:

   ```bash
   cat > comparativa-kraft-zookeeper.md << 'EOF'
   # Análisis Comparativo: KRaft vs ZooKeeper en Apache Kafka
   
   ## Contexto Histórico
   
   Apache Kafka dependió de ZooKeeper para la gestión de metadatos del clúster
   desde su creación en LinkedIn (2011) hasta Kafka 3.x. Con Kafka 4.0 (incluido
   en Confluent Platform 8.0), ZooKeeper fue completamente eliminado y reemplazado
   por KRaft (Kafka Raft Metadata mode).
   
   ---
   
   ## Parte 1: Arquitectura con ZooKeeper (Modo Legado)
   
   Investiga y responde las siguientes preguntas sobre la arquitectura anterior:
   
   **¿Qué es ZooKeeper y cuál era su rol en un clúster Kafka?**
   [COMPLETAR - describe ZooKeeper en 2-3 oraciones]
   
   **Lista las 4 responsabilidades principales que ZooKeeper tenía en Kafka:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   4. [COMPLETAR]
   
   **¿Por qué tener ZooKeeper como componente separado era problemático?
   Lista al menos 3 desventajas operacionales:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   
   ---
   
   ## Parte 2: Arquitectura con KRaft (Kafka 4.0)
   
   **¿Qué es el algoritmo Raft y para qué se usa en KRaft?**
   [COMPLETAR - 2-3 oraciones]
   
   **¿Dónde se almacenan ahora los metadatos del clúster en modo KRaft?**
   [COMPLETAR]
   
   **¿Qué es el "Controller Quorum" en KRaft y cómo funciona?**
   [COMPLETAR - 2-3 oraciones]
   
   **Lista las 4 ventajas principales de KRaft sobre ZooKeeper:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   4. [COMPLETAR]
   
   ---
   
   ## Parte 3: Tabla Comparativa
   
   Completa la siguiente tabla comparativa:
   
   | Característica | ZooKeeper (Legado) | KRaft (Kafka 4.0) |
   |----------------|-------------------|-------------------|
   | Componentes requeridos | Kafka + ZooKeeper | [COMPLETAR] |
   | Almacenamiento de metadatos | ZooKeeper znodes | [COMPLETAR] |
   | Límite práctico de particiones | ~200,000 | [COMPLETAR] |
   | Tiempo de recuperación ante fallo | Minutos | [COMPLETAR] |
   | Complejidad operacional | Alta (2 sistemas) | [COMPLETAR] |
   | Protocolo de consenso | ZAB (ZooKeeper Atomic Broadcast) | [COMPLETAR] |
   | Disponible desde | Kafka 0.x | [COMPLETAR] |
   | Estado en Kafka 4.0 | [COMPLETAR] | Predeterminado |
   
   ---
   
   ## Parte 4: Escenario de Decisión
   
   Un equipo de ingeniería está migrando su clúster Kafka de la versión 2.8
   (con ZooKeeper) a Kafka 4.0 (KRaft). El clúster tiene:
   - 5 brokers
   - 50,000 particiones
   - 200 tópicos
   - Tiempo de inactividad máximo permitido: 30 minutos
   
   **¿Cuáles son los beneficios más importantes que obtendrán con la migración
   a KRaft en este escenario específico? Justifica tu respuesta.**
   [COMPLETAR - mínimo 4 oraciones]
   
   **¿Qué consideraciones de compatibilidad deben tener en cuenta durante
   la migración? (Pista: piensa en los clientes y aplicaciones existentes)**
   [COMPLETAR - mínimo 2 oraciones]
   
   EOF
   ```

3. Completa el análisis usando tu editor y el material del curso:

   ```bash
   nano comparativa-kraft-zookeeper.md
   ```

4. Crea el archivo de respuestas de referencia para esta actividad:

   ```bash
   cat > respuestas-kraft-zookeeper.md << 'EOF'
   # Respuestas de Referencia — KRaft vs ZooKeeper
   
   ## Arquitectura con ZooKeeper
   ZooKeeper es un servicio coordinador distribuido de Apache usado para
   mantener configuración, sincronización y servicios de nombres en sistemas
   distribuidos. En Kafka, actuaba como el "cerebro" del clúster, almacenando
   todos los metadatos y coordinando la elección de líderes.
   
   Responsabilidades de ZooKeeper en Kafka:
   1. Registro y descubrimiento de brokers (qué brokers están activos)
   2. Elección del broker Controlador del clúster
   3. Almacenamiento de configuración de tópicos y particiones
   4. Seguimiento de los offsets de consumidores (en versiones antiguas)
   
   Desventajas de ZooKeeper:
   1. Requería operar y mantener un sistema distribuido separado con su
      propio conjunto de conocimientos, monitoreo y procedimientos.
   2. El protocolo ZAB tenía limitaciones de escalabilidad: con más de
      200,000 particiones, el tiempo de inicio y recuperación se volvía
      inaceptablemente largo (minutos u horas).
   3. La elección del controlador dependía de ZooKeeper, creando un punto
      de fallo externo y aumentando la latencia de recuperación.
   
   ## Arquitectura con KRaft
   Raft es un algoritmo de consenso distribuido diseñado para ser más
   comprensible que Paxos. En KRaft, un quórum de brokers ejecuta el
   protocolo Raft para acordar el estado del clúster sin depender de
   un sistema externo.
   
   Los metadatos se almacenan en un tópico interno especial llamado
   __cluster_metadata, replicado entre los nodos del Controller Quorum.
   
   El Controller Quorum es un conjunto de 3 o 5 brokers designados como
   controladores que ejecutan el protocolo Raft. Uno es el líder activo
   y los demás son seguidores. Si el líder falla, el quórum elige un
   nuevo líder automáticamente en segundos.
   
   Ventajas de KRaft:
   1. Arquitectura simplificada: un solo sistema que instalar, configurar
      y monitorear en lugar de dos.
   2. Escalabilidad masiva: soporta millones de particiones (vs 200K).
   3. Recuperación más rápida: segundos en lugar de minutos.
   4. Consistencia mejorada: los metadatos son parte del propio Kafka.
   
   ## Tabla Comparativa (respuestas)
   - Solo Kafka | Tópico __cluster_metadata | Millones | Segundos
   - Baja (1 sistema) | Raft | Kafka 3.3 (preview), 4.0 (GA)
   - Eliminado/Deprecated
   
   EOF
   ```

**Salida Esperada:**

```
ls -la ~/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper/

total 16
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 2560 [fecha] comparativa-kraft-zookeeper.md
-rw-r--r-- 1 usuario usuario 2048 [fecha] respuestas-kraft-zookeeper.md
```

**Verificación:**

- [ ] Puedes explicar en 30 segundos qué problema resuelve KRaft respecto a ZooKeeper.
- [ ] La tabla comparativa está completamente llena sin celdas vacías.
- [ ] Puedes argumentar por qué KRaft es especialmente beneficioso para clústeres con decenas de miles de particiones.

---

### Paso 4: Análisis de Casos de Uso Reales

**Objetivo:** Relacionar el modelo de publicación/suscripción de Kafka con casos de uso reales de empresas como LinkedIn, Netflix y Uber, identificando qué patrones de mensajería aplica cada una.

**Instrucciones:**

1. Navega al directorio de la cuarta actividad:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso
   ```

2. Crea el archivo de análisis de casos de uso reales:

   ```bash
   cat > casos-uso-reales.md << 'EOF'
   # Análisis de Casos de Uso Reales de Apache Kafka
   
   ## Instrucciones
   Para cada empresa, analiza cómo usa Kafka e identifica:
   - El problema que Kafka resuelve para esa empresa
   - Los productores y consumidores involucrados
   - El patrón de mensajería predominante (Event Notification,
     Event-Carried State Transfer, o Event Sourcing)
   - La escala aproximada del uso
   
   ---
   
   ## Caso 1: LinkedIn — El Origen de Kafka
   
   **Contexto:** LinkedIn creó Kafka en 2011 para resolver un problema específico.
   Con más de 300 millones de usuarios profesionales, necesitaban procesar
   enormes volúmenes de datos de actividad (visitas a perfiles, clics,
   búsquedas, mensajes) para alimentar sistemas de recomendaciones, análisis
   y detección de fraude en tiempo real.
   
   **¿Qué problema específico de LinkedIn resolvió Kafka?**
   [COMPLETAR - describe el problema antes de Kafka y cómo lo resolvió]
   
   **Identifica al menos 3 productores de eventos en el sistema de LinkedIn:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   
   **Identifica al menos 3 consumidores de esos eventos:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   
   **¿Qué patrón de mensajería usa LinkedIn principalmente para datos de
   actividad de usuarios? Justifica tu respuesta.**
   [COMPLETAR]
   
   ---
   
   ## Caso 2: Netflix — Streaming de Entretenimiento y Datos
   
   **Contexto:** Netflix sirve más de 200 millones de suscriptores globales.
   Cada vez que un usuario reproduce un video, pausa, avanza o cambia la
   calidad de reproducción, ese evento debe ser capturado y procesado para
   mejorar recomendaciones, detectar problemas de calidad de servicio y
   optimizar la infraestructura de CDN.
   
   **¿Cómo usa Netflix Kafka para mejorar la experiencia del usuario?**
   [COMPLETAR - 2-3 oraciones]
   
   **¿Por qué es crítico para Netflix que los eventos de reproducción NO
   se pierdan, incluso si un consumidor falla temporalmente?**
   [COMPLETAR - relaciona con la durabilidad de mensajes en Kafka]
   
   **Netflix tiene sistemas de recomendación, detección de anomalías y
   facturación que consumen los mismos eventos de reproducción.
   ¿Qué modelo de distribución de Kafka permite esto?**
   [COMPLETAR - explica el modelo pub/sub y los Consumer Groups]
   
   **¿Qué patrón de mensajería es más apropiado para los eventos de
   reproducción de Netflix? Justifica.**
   [COMPLETAR - Event Notification, Event-Carried State Transfer o Event Sourcing]
   
   ---
   
   ## Caso 3: Uber — Logística en Tiempo Real
   
   **Contexto:** Uber coordina millones de viajes diarios globalmente.
   Cada conductor envía su ubicación GPS cada pocos segundos, los usuarios
   solicitan viajes, los algoritmos de precios dinámicos calculan tarifas,
   y el sistema de emparejamiento conecta conductores con pasajeros, todo
   en tiempo real.
   
   **¿Por qué la mensajería asíncrona es especialmente valiosa para Uber
   frente a llamadas REST síncronas?**
   [COMPLETAR - considera la escala y la naturaleza de los datos de ubicación]
   
   **Estima la escala de eventos por segundo que Uber podría generar
   considerando: 5 millones de conductores activos enviando GPS cada 3 segundos.**
   
   Cálculo:
   [COMPLETAR - realiza el cálculo matemático simple]
   
   **¿Qué pasa con el sistema de emparejamiento si el servicio de precios
   dinámicos tiene un pico de latencia en un modelo síncrono vs asíncrono?**
   [COMPLETAR - compara ambos escenarios]
   
   **¿Qué patrón de mensajería es más apropiado para el historial de viajes
   de Uber (para auditoría y disputas)? Justifica.**
   [COMPLETAR]
   
   ---
   
   ## Síntesis Comparativa
   
   Completa la siguiente tabla comparando los tres casos:
   
   | Empresa | Volumen estimado | Patrón principal | Característica Kafka más crítica |
   |---------|-----------------|-----------------|----------------------------------|
   | LinkedIn | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | Netflix | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | Uber | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   
   **¿Qué tienen en común los tres casos de uso que hace que Kafka sea
   la solución ideal para todos ellos?**
   [COMPLETAR - identifica el patrón común, mínimo 3 oraciones]
   
   EOF
   ```

3. Completa el análisis con tus respuestas:

   ```bash
   nano casos-uso-reales.md
   ```

4. Crea un archivo de contexto adicional con datos reales de referencia:

   ```bash
   cat > datos-escala-referencia.md << 'EOF'
   # Datos de Escala Real — Para Referencia
   
   ## LinkedIn (datos públicos aproximados, 2023)
   - 7 billones de mensajes por día procesados por Kafka
   - Más de 10,000 tópicos activos
   - Pico de 7 millones de mensajes por segundo
   - Fuente: Engineering Blog de LinkedIn
   
   ## Netflix (datos públicos aproximados, 2023)
   - 500 billones de eventos por día
   - Kafka procesa datos de 200+ millones de suscriptores
   - Usado para: recomendaciones, monitoreo de calidad, seguridad, facturación
   - Fuente: Netflix Tech Blog
   
   ## Uber (datos públicos aproximados, 2023)
   - 1 billón de mensajes por día en picos
   - Más de 4,000 tópicos Kafka
   - Casos de uso: geolocalización, emparejamiento, precios dinámicos, fraude
   - Fuente: Uber Engineering Blog
   
   ## Cálculo de escala para Uber (GPS):
   5,000,000 conductores × (1 evento / 3 segundos) = 1,666,667 eventos/segundo
   Solo de datos GPS. El total incluyendo solicitudes, emparejamientos y
   actualizaciones de estado es significativamente mayor.
   
   EOF
   ```

**Salida Esperada:**

```
ls -la ~/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso/

total 16
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 3584 [fecha] casos-uso-reales.md
-rw-r--r-- 1 usuario usuario 1024 [fecha] datos-escala-referencia.md
```

**Verificación:**

- [ ] Puedes explicar por qué LinkedIn creó Kafka y qué problema específico resolvía.
- [ ] Puedes calcular correctamente la escala de eventos por segundo del caso de Uber.
- [ ] La tabla comparativa está completamente llena con argumentos justificados.

---

### Paso 5: Reconocimiento de Componentes de Confluent Platform 8.0

**Objetivo:** Identificar los componentes adicionales que Confluent Platform 8.0 agrega sobre el Apache Kafka base, comprendiendo el valor agregado de cada uno.

**Instrucciones:**

1. Navega al directorio de la quinta actividad:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-6-confluent-platform
   ```

2. Crea el archivo de análisis de Confluent Platform:

   ```bash
   cat > confluent-platform-analisis.md << 'EOF'
   # Análisis de Componentes — Confluent Platform 8.0
   
   ## Introducción
   Apache Kafka es el núcleo open-source. Confluent Platform 8.0 es la
   distribución empresarial que agrega componentes adicionales para facilitar
   el uso en producción. En este laboratorio identificarás qué agrega cada
   componente y por qué es valioso.
   
   ---
   
   ## Componente 1: Schema Registry
   
   **¿Qué problema resuelve Schema Registry?**
   (Pista: piensa en qué pasa si el productor cambia el formato del mensaje
   sin avisar a los consumidores)
   [COMPLETAR]
   
   **¿Qué es un schema (esquema) en el contexto de mensajes Kafka?**
   [COMPLETAR]
   
   **¿Qué formatos de serialización soporta Schema Registry?**
   [COMPLETAR - menciona al menos 3: Avro, Protobuf, JSON Schema]
   
   **¿Qué significa "compatibilidad BACKWARD" en Schema Registry?**
   [COMPLETAR]
   
   **Escenario:** Un productor cambia el campo "monto" de tipo Integer a Float
   en el esquema de "ordenes-confirmadas". ¿Esto sería compatible BACKWARD?
   [COMPLETAR - justifica tu respuesta]
   
   ---
   
   ## Componente 2: Kafka Connect
   
   **¿Qué es Kafka Connect y para qué sirve?**
   [COMPLETAR - 2-3 oraciones]
   
   **¿Cuál es la diferencia entre un conector Source y un conector Sink?**
   [COMPLETAR - da un ejemplo concreto de cada uno]
   
   **¿Por qué es valioso usar Kafka Connect en lugar de escribir código
   personalizado de productor/consumidor para integrar bases de datos?**
   [COMPLETAR - menciona al menos 2 ventajas]
   
   **En el contexto del curso, ¿qué conector específico se usará para
   integrar PostgreSQL con Kafka?**
   [COMPLETAR]
   
   ---
   
   ## Componente 3: ksqlDB
   
   **¿Qué es ksqlDB y qué lo hace diferente de una base de datos SQL tradicional?**
   [COMPLETAR - enfatiza el concepto de "streaming SQL"]
   
   **¿Cuál es la diferencia entre un STREAM y una TABLE en ksqlDB?**
   [COMPLETAR]
   
   **Escribe un ejemplo conceptual de una consulta ksqlDB que calcule
   el total de ventas por categoría de producto en tiempo real:**
   
   ```sql
   -- Tu consulta ksqlDB aquí (no necesita ser 100% sintácticamente correcta,
   -- pero sí conceptualmente coherente)
   [COMPLETAR]
   ```
   
   ---
   
   ## Componente 4: Confluent Control Center
   
   **¿Qué funcionalidades principales ofrece el Control Center?**
   [COMPLETAR - lista al menos 4 funcionalidades]
   
   **¿Por qué es importante tener una interfaz web de administración
   además de las herramientas de línea de comandos?**
   [COMPLETAR]
   
   **¿Qué tipo de métricas esperas poder visualizar en el Control Center?**
   [COMPLETAR - menciona al menos 5 métricas relevantes]
   
   ---
   
   ## Componente 5: Confluent REST Proxy
   
   **¿Qué es el REST Proxy y en qué casos es útil?**
   [COMPLETAR]
   
   **¿Cuándo preferirías usar el REST Proxy en lugar del cliente nativo de Kafka?**
   [COMPLETAR - piensa en lenguajes de programación sin cliente Kafka nativo]
   
   ---
   
   ## Mapa de Valor Agregado
   
   Para cada componente, indica qué problema del mundo real resuelve
   en una frase concisa:
   
   | Componente | Sin el componente... | Con el componente... |
   |------------|---------------------|---------------------|
   | Schema Registry | Los mensajes pueden tener formatos incompatibles sin detección | [COMPLETAR] |
   | Kafka Connect | Debes escribir código personalizado para cada integración | [COMPLETAR] |
   | ksqlDB | Necesitas un sistema de procesamiento stream separado (Flink, Spark) | [COMPLETAR] |
   | Control Center | Solo tienes herramientas CLI para administrar el clúster | [COMPLETAR] |
   | REST Proxy | Solo aplicaciones con cliente Kafka nativo pueden interactuar | [COMPLETAR] |
   
   ---
   
   ## Reflexión sobre Licenciamiento
   
   **Confluent Platform tiene componentes con licencia comercial.
   ¿Cuáles de los componentes analizados son open-source y cuáles
   son comerciales? ¿Cómo afecta esto a la decisión de adopción?**
   [COMPLETAR - investiga o razona sobre esto, mínimo 3 oraciones]
   
   EOF
   ```

3. Completa el análisis con tu editor:

   ```bash
   nano confluent-platform-analisis.md
   ```

**Salida Esperada:**

```
ls -la ~/kafka-labs/lab-01-00-01/actividad-6-confluent-platform/

total 8
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 3840 [fecha] confluent-platform-analisis.md
```

**Verificación:**

- [ ] Puedes explicar la diferencia entre un conector Source y un Sink con ejemplos concretos.
- [ ] Puedes describir qué problema resuelve Schema Registry con un ejemplo de escenario de fallo.
- [ ] La tabla de "Mapa de Valor Agregado" está completamente llena.

---

### Paso 6: Diseño de Arquitectura Kafka para un Escenario de Negocio

**Objetivo:** Aplicar los conceptos aprendidos diseñando una arquitectura Kafka básica para un escenario de negocio real, justificando cada decisión de diseño con base en los fundamentos del capítulo.

**Instrucciones:**

1. Navega al directorio de la actividad de diseño:

   ```bash
   cd ~/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura
   ```

2. Lee el escenario de negocio asignado y crea el documento de diseño:

   ```bash
   cat > escenario-diseno.md << 'EOF'
   # Escenario de Negocio para Diseño de Arquitectura
   
   ## Sistema: Plataforma de Monitoreo de Salud IoT
   
   ### Descripción del Negocio
   Una empresa de salud digital despliega 50,000 dispositivos IoT (relojes
   inteligentes y monitores de glucosa) que envían métricas de salud de
   pacientes cada 30 segundos. Los datos deben ser procesados por:
   
   1. **Sistema de Alertas**: Detecta valores críticos (ej: frecuencia cardíaca
      > 150 bpm) y notifica al médico en menos de 5 segundos.
   
   2. **Sistema de Análisis Histórico**: Almacena todos los datos en un data
      warehouse para análisis de tendencias y reportes médicos.
   
   3. **Sistema de Machine Learning**: Consume datos en tiempo real para
      actualizar modelos predictivos de condiciones médicas.
   
   4. **Sistema de Facturación**: Registra el uso del servicio por paciente
      para generar facturas mensuales.
   
   5. **Dashboard de Médicos**: Muestra el estado actual de sus pacientes
      en tiempo real.
   
   ### Restricciones del Sistema
   - Los datos de salud son críticos: NO se puede perder ningún evento.
   - El sistema de alertas tiene SLA de latencia < 5 segundos end-to-end.
   - Los datos deben retenerse por 7 años (regulación médica).
   - El sistema debe escalar a 500,000 dispositivos en 2 años.
   - Presupuesto de infraestructura: moderado (no se puede tener 20 brokers).
   
   EOF
   ```

3. Crea el documento de diseño de arquitectura que deberás completar:

   ```bash
   cat > diseno-arquitectura-kafka.md << 'EOF'
   # Diseño de Arquitectura Kafka — Plataforma IoT de Salud
   
   ## Autor: [Tu nombre]
   ## Fecha: [Fecha actual]
   
   ---
   
   ## Sección 1: Identificación de Productores y Consumidores
   
   ### Productores
   Lista todos los productores de eventos en este sistema:
   
   | Productor | Eventos que genera | Frecuencia estimada |
   |-----------|-------------------|---------------------|
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   
   ### Consumidores
   Lista todos los consumidores y su agrupación en Consumer Groups:
   
   | Sistema Consumidor | Consumer Group | Número de Instancias |
   |-------------------|---------------|---------------------|
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   | [COMPLETAR] | [COMPLETAR] | [COMPLETAR] |
   
   ---
   
   ## Sección 2: Diseño de Tópicos
   
   Para cada tópico que necesites, define:
   
   ### Tópico 1: [Nombre del tópico]
   **Propósito:** [COMPLETAR]
   **Número de particiones:** [COMPLETAR]
   **Justificación de particiones:** [COMPLETAR - considera el paralelismo necesario]
   **Factor de replicación:** [COMPLETAR]
   **Justificación de replicación:** [COMPLETAR - considera la criticidad de los datos]
   **Política de retención:** [COMPLETAR - tiempo o tamaño, considera la regulación de 7 años]
   **Clave del mensaje:** [COMPLETAR - ¿qué campo usarías como clave y por qué?]
   
   ### Tópico 2: [Nombre del tópico]
   **Propósito:** [COMPLETAR]
   **Número de particiones:** [COMPLETAR]
   **Justificación de particiones:** [COMPLETAR]
   **Factor de replicación:** [COMPLETAR]
   **Justificación de replicación:** [COMPLETAR]
   **Política de retención:** [COMPLETAR]
   **Clave del mensaje:** [COMPLETAR]
   
   (Agrega más tópicos si tu diseño lo requiere)
   
   ---
   
   ## Sección 3: Diseño del Clúster
   
   **¿Cuántos brokers propones para este sistema? Justifica.**
   [COMPLETAR - considera: factor de replicación, tolerancia a fallos, escala futura]
   
   **¿Usarías KRaft o ZooKeeper? ¿Por qué?**
   [COMPLETAR - en el contexto de Kafka 4.0 esto debería ser obvio, pero justifica]
   
   **¿Qué componentes de Confluent Platform incluirías y por qué?**
   [COMPLETAR - para cada componente que menciones, justifica por qué es necesario
   en este escenario específico de salud IoT]
   
   ---
   
   ## Sección 4: Patrones de Mensajería
   
   **¿Qué patrón de mensajería usarías para los datos de métricas de salud?**
   (Event Notification, Event-Carried State Transfer, o Event Sourcing)
   [COMPLETAR - justifica considerando que los datos deben retenerse 7 años
   y el sistema de ML necesita datos completos]
   
   **¿Cómo garantizas que el sistema de alertas reciba los datos críticos
   en menos de 5 segundos?**
   [COMPLETAR - piensa en la configuración del productor: acks, linger.ms, batch.size]
   
   **¿Cómo garantizas que NO se pierda ningún evento de salud?**
   [COMPLETAR - piensa en: replicación, acks, retención, Consumer Group offsets]
   
   ---
   
   ## Sección 5: Diagrama ASCII de la Arquitectura
   
   Dibuja un diagrama ASCII que muestre los componentes principales
   y sus relaciones. Incluye: dispositivos IoT, productores, brokers,
   tópicos, consumer groups y sistemas consumidores.
   
   ```
   [Tu diagrama ASCII aquí]
   
   Ejemplo de estructura sugerida (modifícala completamente):
   
   [Dispositivos IoT] --> [Productor: Gateway IoT]
                                    |
                                    v
                         [Kafka Cluster: 3 Brokers]
                         [Tópico: metricas-salud]
                         [Particiones: ?]
                                    |
              +---------------------+---------------------+
              |                     |                     |
              v                     v                     v
   [CG: alertas]          [CG: analytics]      [CG: ml-model]
   [Sistema Alertas]      [Data Warehouse]     [ML Pipeline]
   ```
   
   ---
   
   ## Sección 6: Consideraciones de Escala Futura
   
   **El sistema debe escalar de 50,000 a 500,000 dispositivos.
   ¿Qué aspectos de tu diseño actual facilitan esa escala?**
   [COMPLETAR]
   
   **¿Qué cambiarías en tu diseño cuando llegue a 500,000 dispositivos?**
   [COMPLETAR]
   
   ---
   
   ## Sección 7: Reflexión sobre Comunicación Síncrona vs Asíncrona
   
   **¿Por qué sería inviable usar comunicación REST síncrona para este
   sistema IoT de salud con 50,000 dispositivos? Lista al menos 4 razones
   específicas para este escenario.**
   
   1. [COMPLETAR]
   2. [COMPLETAR]
   3. [COMPLETAR]
   4. [COMPLETAR]
   
   EOF
   ```

4. Completa el diseño de arquitectura. Esta es la actividad más importante del laboratorio, tómate el tiempo necesario:

   ```bash
   nano diseno-arquitectura-kafka.md
   ```

5. Una vez completado, genera un resumen ejecutivo de tu diseño:

   ```bash
   cat > resumen-ejecutivo.md << 'EOF'
   # Resumen Ejecutivo — Diseño de Arquitectura Kafka IoT Salud
   
   ## Decisiones Clave de Diseño
   (Completa este resumen después de terminar el diseño completo)
   
   **Número de tópicos diseñados:** [COMPLETAR]
   **Número de brokers propuestos:** [COMPLETAR]
   **Componentes Confluent Platform incluidos:** [COMPLETAR]
   **Patrón de mensajería principal:** [COMPLETAR]
   
   ## La decisión de diseño más importante que tomé fue:
   [COMPLETAR - en 2-3 oraciones, describe la decisión más crítica y por qué]
   
   ## El mayor riesgo de mi diseño es:
   [COMPLETAR - identifica honestamente una debilidad o área de incertidumbre]
   
   ## Cómo Kafka resuelve el problema mejor que REST síncrono:
   [COMPLETAR - síntesis en 3-4 oraciones]
   
   EOF
   
   nano resumen-ejecutivo.md
   ```

**Salida Esperada:**

```
ls -la ~/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/

total 20
drwxr-xr-x 2 usuario usuario 4096 [fecha] .
drwxr-xr-x 8 usuario usuario 4096 [fecha] ..
-rw-r--r-- 1 usuario usuario 1536 [fecha] escenario-diseno.md
-rw-r--r-- 1 usuario usuario 4096 [fecha] diseno-arquitectura-kafka.md
-rw-r--r-- 1 usuario usuario 512  [fecha] resumen-ejecutivo.md
```

**Verificación:**

- [ ] El diseño incluye al menos 2 tópicos con justificación de particiones y replicación.
- [ ] Se propone un número de brokers con justificación coherente.
- [ ] El diagrama ASCII muestra claramente los flujos de datos entre componentes.
- [ ] La sección de escala futura demuestra comprensión de cómo Kafka escala horizontalmente.

---

### Paso 7: Consolidación y Autoevaluación Final

**Objetivo:** Consolidar todo el aprendizaje del laboratorio mediante una autoevaluación estructurada que verifique la comprensión de los conceptos clave del Capítulo 1.

**Instrucciones:**

1. Regresa al directorio raíz del laboratorio:

   ```bash
   cd ~/kafka-labs/lab-01-00-01
   ```

2. Crea el archivo de autoevaluación final:

   ```bash
   cat > autoevaluacion-final.md << 'EOF'
   # Autoevaluación Final — Laboratorio 01-00-01
   
   ## Instrucciones
   Responde cada pregunta sin consultar tus notas. Luego verifica tus
   respuestas con el material del curso. Puntúate honestamente del 1 al 5
   en cada área (1=No lo entiendo, 5=Puedo explicárselo a otro).
   
   ---
   
   ## Bloque 1: Mensajería Basada en Eventos (Conceptos Fundamentales)
   
   **Pregunta 1:** ¿Cuál es la diferencia fundamental entre un evento y
   una solicitud (request) en el contexto de sistemas distribuidos?
   [Tu respuesta:]
   
   **Pregunta 2:** Describe el patrón "Event-Carried State Transfer" con
   un ejemplo del mundo real diferente a los del material del curso.
   [Tu respuesta:]
   
   **Pregunta 3:** ¿Cuándo es preferible usar comunicación síncrona (REST)
   sobre mensajería asíncrona? Da un ejemplo concreto.
   [Tu respuesta:]
   
   **Autoevaluación Bloque 1:** [1-5]
   
   ---
   
   ## Bloque 2: Componentes de Apache Kafka 4.0
   
   **Pregunta 4:** Explica la relación entre un tópico, sus particiones
   y los offsets sin usar diagramas.
   [Tu respuesta:]
   
   **Pregunta 5:** ¿Por qué un Consumer Group con más consumidores que
   particiones no mejora el throughput de procesamiento?
   [Tu respuesta:]
   
   **Pregunta 6:** ¿Qué ventaja tiene Kafka sobre RabbitMQ en términos
   de la capacidad de "reproducir" mensajes pasados?
   [Tu respuesta:]
   
   **Autoevaluación Bloque 2:** [1-5]
   
   ---
   
   ## Bloque 3: KRaft vs ZooKeeper
   
   **Pregunta 7:** Explica en términos simples qué hace el algoritmo Raft
   que antes hacía ZooKeeper en un clúster Kafka.
   [Tu respuesta:]
   
   **Pregunta 8:** ¿Por qué la eliminación de ZooKeeper permite a Kafka 4.0
   soportar millones de particiones en lugar de ~200,000?
   [Tu respuesta:]
   
   **Autoevaluación Bloque 3:** [1-5]
   
   ---
   
   ## Bloque 4: Confluent Platform 8.0
   
   **Pregunta 9:** ¿En qué escenario Schema Registry previene un incidente
   de producción? Describe el escenario concreto.
   [Tu respuesta:]
   
   **Pregunta 10:** ¿Cuál es la diferencia entre Kafka Connect y escribir
   un consumidor personalizado para leer de PostgreSQL?
   [Tu respuesta:]
   
   **Autoevaluación Bloque 4:** [1-5]
   
   ---
   
   ## Bloque 5: Diseño de Arquitectura
   
   **Pregunta 11:** Para el escenario IoT de salud, ¿por qué elegiste el
   número de particiones que elegiste para el tópico principal?
   [Tu respuesta - referencia tu diseño del Paso 6:]
   
   **Pregunta 12:** ¿Cómo garantiza tu diseño que ningún dato de salud
   se pierda en caso de que un broker falle?
   [Tu respuesta:]
   
   **Autoevaluación Bloque 5:** [1-5]
   
   ---
   
   ## Puntuación Total y Plan de Acción
   
   Suma tus puntuaciones: [Bloque1 + Bloque2 + Bloque3 + Bloque4 + Bloque5] = [TOTAL]/25
   
   **Áreas donde me siento más seguro:**
   [COMPLETAR]
   
   **Áreas donde necesito reforzar:**
   [COMPLETAR]
   
   **Acciones concretas que tomaré antes del próximo laboratorio:**
   1. [COMPLETAR]
   2. [COMPLETAR]
   
   EOF
   
   nano autoevaluacion-final.md
   ```

3. Genera un reporte de completitud del laboratorio para verificar que todos los archivos están presentes:

   ```bash
   echo "=== REPORTE DE COMPLETITUD — LABORATORIO 01-00-01 ==="
   echo ""
   echo "Archivos generados:"
   find ~/kafka-labs/lab-01-00-01 -type f -name "*.md" | sort
   echo ""
   echo "Total de archivos creados:"
   find ~/kafka-labs/lab-01-00-01 -type f -name "*.md" | wc -l
   echo ""
   echo "Tamaño total del trabajo realizado:"
   du -sh ~/kafka-labs/lab-01-00-01/
   ```

**Salida Esperada:**

```
=== REPORTE DE COMPLETITUD — LABORATORIO 01-00-01 ===

Archivos generados:
/home/usuario/kafka-labs/lab-01-00-01/actividad-1-componentes/componentes-kafka.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-1-componentes/respuestas-referencia-componentes.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes/escenario-ecommerce.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes/flujo-mensaje-kafka.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes/validacion-flujo.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper/comparativa-kraft-zookeeper.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper/respuestas-kraft-zookeeper.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso/casos-uso-reales.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso/datos-escala-referencia.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/diseno-arquitectura-kafka.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/escenario-diseno.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/resumen-ejecutivo.md
/home/usuario/kafka-labs/lab-01-00-01/actividad-6-confluent-platform/confluent-platform-analisis.md
/home/usuario/kafka-labs/lab-01-00-01/autoevaluacion-final.md

Total de archivos creados:
14

Tamaño total del trabajo realizado:
156K    /home/usuario/kafka-labs/lab-01-00-01/
```

**Verificación:**

- [ ] Existen exactamente 14 archivos Markdown en la estructura del laboratorio.
- [ ] El archivo de autoevaluación está completamente respondido (sin campos `[Tu respuesta:]` vacíos).
- [ ] La puntuación de autoevaluación ha sido calculada y se han identificado áreas de mejora.

## Validación y Pruebas

### Criterios de Éxito

- [ ] Todos los archivos de análisis están completados sin campos `[COMPLETAR]` vacíos.
- [ ] El mapeo de componentes Kafka incluye los 7 componentes con descripción, responsabilidad, analogía e interacciones.
- [ ] El trazado del flujo de mensajes responde correctamente por qué los mensajes no se eliminan tras ser leídos.
- [ ] La tabla comparativa KRaft vs ZooKeeper tiene todas las celdas completadas con información correcta.
- [ ] Los tres casos de uso (LinkedIn, Netflix, Uber) están analizados con el patrón de mensajería identificado.
- [ ] El diseño de arquitectura IoT incluye al menos 2 tópicos con justificación completa de particiones y replicación.
- [ ] La autoevaluación final está completada con puntuaciones y plan de acción.
- [ ] El reporte de completitud muestra los 14 archivos esperados.

### Procedimiento de Pruebas

1. Verifica que no quedan campos sin completar en ningún archivo:

   ```bash
   # Buscar campos incompletos en todos los archivos
   grep -r "\[COMPLETAR\]" ~/kafka-labs/lab-01-00-01/ --include="*.md"
   ```

   **Resultado Esperado:** No debe aparecer ninguna línea con `[COMPLETAR]`. Si aparecen, debes volver a esos archivos y completarlos.

2. Verifica que el archivo de diseño de arquitectura tiene contenido sustancial:

   ```bash
   # Contar líneas del archivo de diseño (debe tener más de 50 líneas con contenido)
   wc -l ~/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/diseno-arquitectura-kafka.md
   ```

   **Resultado Esperado:** El archivo debe tener al menos 80 líneas (incluyendo el contenido que completaste).

3. Verifica la integridad de la estructura de directorios:

   ```bash
   # Verificar que todos los directorios y archivos existen
   find ~/kafka-labs/lab-01-00-01 -type f | sort | wc -l
   ```

   **Resultado Esperado:** Al menos 14 archivos.

4. Realiza una prueba de comprensión oral (o escrita) respondiendo estas preguntas sin consultar notas:

   ```bash
   cat > ~/kafka-labs/lab-01-00-01/prueba-comprension-oral.md << 'EOF'
   # Prueba de Comprensión Final
   
   Responde estas preguntas en voz alta o por escrito en 2 minutos cada una:
   
   1. Explica la diferencia entre una cola (Queue) y pub/sub en 30 segundos.
   
   2. ¿Por qué un Consumer Group con 3 consumidores y un tópico de 2 particiones
      no puede procesar más rápido que con 2 consumidores?
   
   3. ¿Qué componente de Confluent Platform usarías para conectar una base
      de datos Oracle a Kafka sin escribir código?
   
   4. Describe en una oración qué es KRaft y por qué importa.
   
   5. ¿Cuál es la diferencia entre Event Notification y Event-Carried State Transfer?
   
   EOF
   
   echo "Prueba de comprensión creada. Respóndela sin consultar notas."
   ```

   **Resultado Esperado:** Puedes responder las 5 preguntas con fluidez y precisión.

## Solución de Problemas

### Problema 1: El Comando `cat > archivo.md << 'EOF'` No Funciona en Windows PowerShell

**Síntomas:**
- Error: `The term 'cat' is not recognized` en PowerShell
- El archivo se crea vacío o con contenido incorrecto
- El heredoc (`<< 'EOF'`) no es reconocido

**Causa:**
PowerShell no soporta la sintaxis heredoc de Bash. El comando `cat` en PowerShell es un alias de `Get-Content`, no de `echo`.

**Solución:**

```powershell
# Opción 1: Usar WSL2 (recomendado para este curso)
wsl bash -c "cat > ~/kafka-labs/lab-01-00-01/actividad-1-componentes/componentes-kafka.md << 'EOF'
[contenido]
EOF"

# Opción 2: Crear el archivo directamente con el editor
# Abre VS Code y crea los archivos manualmente
code ~/kafka-labs/lab-01-00-01/

# Opción 3: Usar PowerShell con Set-Content
Set-Content -Path "componentes-kafka.md" -Value @"
# Análisis de Componentes de Apache Kafka 4.0
[contenido aquí]
"@
```

---

### Problema 2: El Editor `nano` No Está Disponible en el Sistema

**Síntomas:**
- Error: `bash: nano: command not found`
- El comando `nano archivo.md` falla

**Causa:**
`nano` no está instalado en el sistema operativo o no está en el PATH.

**Solución:**

```bash
# En Ubuntu/Debian: instalar nano
sudo apt-get update && sudo apt-get install -y nano

# Alternativa: usar vim (generalmente disponible)
vim componentes-kafka.md
# Para salir de vim: ESC, luego :wq (guardar y salir) o :q! (salir sin guardar)

# Alternativa: usar el editor de texto gráfico del sistema
# En Ubuntu con GNOME:
gedit componentes-kafka.md &

# En macOS:
open -a TextEdit componentes-kafka.md

# Alternativa universal: VS Code (si está instalado)
code componentes-kafka.md
```

---

### Problema 3: El Comando `find` No Muestra los 14 Archivos Esperados

**Síntomas:**
- `find ~/kafka-labs/lab-01-00-01 -type f | wc -l` devuelve menos de 14
- Faltan archivos en la estructura de directorios

**Causa:**
Alguno de los pasos de creación de archivos no se ejecutó correctamente, o se usó un directorio diferente.

**Solución:**

```bash
# Verificar qué archivos existen actualmente
find ~/kafka-labs/lab-01-00-01 -type f | sort

# Verificar qué directorios existen
find ~/kafka-labs/lab-01-00-01 -type d

# Si falta algún directorio, recrearlo
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-1-componentes
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-2-flujo-mensajes
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-3-kraft-vs-zookeeper
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-4-casos-de-uso
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura
mkdir -p ~/kafka-labs/lab-01-00-01/actividad-6-confluent-platform

# Volver al paso correspondiente y re-ejecutar los comandos de creación
# de archivos que faltan
```

---

### Problema 4: Dificultad para Completar la Actividad de Diseño de Arquitectura (Paso 6)

**Síntomas:**
- No sabes cuántas particiones proponer para el tópico principal
- No puedes justificar el número de brokers
- El diagrama ASCII se ve incompleto o confuso

**Causa:**
Los conceptos de dimensionamiento de Kafka se profundizarán en laboratorios posteriores. Para este laboratorio conceptual, las respuestas no necesitan ser perfectas.

**Solución:**

```bash
# Usa estas guías heurísticas para el diseño inicial:

# Número de particiones:
# Regla general: particiones = max(consumidores esperados, throughput_MB/s / 10)
# Para 50,000 dispositivos enviando cada 30s: ~1,667 eventos/seg
# Con mensajes de 1KB: ~1.6 MB/s → al menos 1 partición, pero para paralelismo
# y crecimiento futuro: 12-24 particiones es razonable

# Factor de replicación:
# Para datos críticos de salud: siempre 3 (tolera la pérdida de 1 broker)
# Nunca menos de 3 en producción para datos críticos

# Número de brokers:
# Mínimo = factor de replicación = 3
# Recomendado para escala futura: 5-7 brokers

# Crea un archivo de guía de referencia
cat > ~/kafka-labs/lab-01-00-01/actividad-5-diseno-arquitectura/guia-dimensionamiento.md << 'EOF'
# Guía Rápida de Dimensionamiento Kafka (Nivel Introductorio)

## Particiones
- Más particiones = más paralelismo = mayor throughput
- Regla práctica inicial: 3x el número de brokers
- Para el escenario IoT: 12-24 particiones para el tópico principal

## Factor de Replicación
- 1: Sin tolerancia a fallos (solo desarrollo)
- 2: Tolera 1 fallo (mínimo para staging)
- 3: Tolera 1 fallo con margen (estándar producción)
- Para datos médicos críticos: SIEMPRE 3

## Número de Brokers
- Mínimo = factor de replicación
- Recomendado = factor de replicación + 1 o 2 (para mantenimiento sin downtime)
- Para este escenario: 3-5 brokers

## Retención de Datos
- Para regulación de 7 años: retention.ms = 220752000000 (7 años en ms)
- Alternativa: usar log compaction para el estado más reciente

EOF
echo "Guía de dimensionamiento creada en actividad-5-diseno-arquitectura/"
```

## Limpieza

Este laboratorio es completamente conceptual y no despliega ningún servicio o contenedor. Los únicos archivos creados son documentos Markdown en tu directorio local.

```bash
# OPCIÓN A: Conservar todos los archivos del laboratorio (RECOMENDADO)
# Los archivos servirán como referencia para los laboratorios siguientes
echo "Los archivos del laboratorio se conservan en: ~/kafka-labs/lab-01-00-01/"
ls -la ~/kafka-labs/lab-01-00-01/

# OPCIÓN B: Archivar el laboratorio en un archivo comprimido
# (solo si necesitas liberar espacio, los archivos son muy pequeños)
cd ~/kafka-labs
tar -czf lab-01-00-01-backup.tar.gz lab-01-00-01/
echo "Laboratorio archivado en: ~/kafka-labs/lab-01-00-01-backup.tar.gz"

# OPCIÓN C: Eliminar completamente (NO RECOMENDADO - perderás tu trabajo)
# Solo ejecutar si estás completamente seguro
# rm -rf ~/kafka-labs/lab-01-00-01/
```

> ⚠️ **Advertencia:** NO elimines los archivos de este laboratorio. Los documentos de análisis y diseño que creaste servirán como referencia conceptual para los laboratorios 2 al 7, especialmente el diseño de arquitectura del Paso 6, que se irá refinando a medida que aprendas a implementar cada componente en la práctica.

> 💡 **Nota sobre Persistencia:** A diferencia de los laboratorios posteriores que usan Docker con volúmenes, este laboratorio almacena únicamente archivos de texto en tu sistema local. No hay contenedores que detener ni volúmenes que eliminar.

## Resumen

### Lo que Lograste

- Completaste un análisis exhaustivo de los 7 componentes fundamentales de Apache Kafka 4.0 (Producer, Broker, Consumer, Topic, Partition, Consumer Group, Offset) con descripciones, responsabilidades y analogías del mundo real.
- Trazaste el flujo completo de un mensaje en un sistema de e-commerce desde su producción hasta el consumo por 5 sistemas independientes, comprendiendo la durabilidad de mensajes y la gestión de offsets.
- Analizaste y documentaste las diferencias arquitectónicas entre KRaft y ZooKeeper, comprendiendo por qué KRaft es el futuro de Kafka y sus ventajas en escalabilidad y simplicidad operacional.
- Contextualizaste el uso de Kafka en empresas reales (LinkedIn, Netflix, Uber) a escala de millones de eventos por segundo, identificando los patrones de mensajería que cada una aplica.
- Reconociste el ecosistema completo de Confluent Platform 8.0 (Schema Registry, Kafka Connect, ksqlDB, Control Center, REST Proxy) y el valor específico que cada componente aporta.
- Diseñaste una arquitectura Kafka completa para un sistema IoT de salud, tomando y justificando decisiones de diseño sobre tópicos, particiones, replicación y componentes necesarios.

### Conceptos Clave Aprendidos

- **La mensajería basada en eventos** desacopla productores y consumidores, permitiendo que los sistemas operen de forma independiente y asíncrona, lo que aumenta la resiliencia y escalabilidad del sistema completo.
- **El modelo pub/sub de Kafka** permite que múltiples Consumer Groups lean el mismo tópico de forma independiente, a diferencia del modelo de cola donde cada mensaje es consumido por un solo receptor.
- **KRaft elimina ZooKeeper** como dependencia externa, simplificando la operación del clúster y permitiendo escalar a millones de particiones con tiempos de recuperación en segundos en lugar de minutos.
- **Los patrones de mensajería** (Event Notification, Event-Carried State Transfer, Event Sourcing) ofrecen diferentes equilibrios entre tamaño del mensaje, acoplamiento y capacidad de auditoría, y Kafka es especialmente adecuado para los dos últimos.
- **Confluent Platform 8.0** agrega una capa de productividad y gestión empresarial sobre el core de Kafka que reduce significativamente el tiempo de desarrollo e integración en proyectos reales.

### Próximos Pasos

- En el **Laboratorio 1.2** explorarás la historia y evolución de Apache Kafka, desde su origen en LinkedIn hasta su adopción masiva, comprendiendo las decisiones de diseño que lo diferencian de RabbitMQ y ActiveMQ.
- En los **Laboratorios 2 y 3** pondrás en práctica los conceptos de este laboratorio instalando y configurando un clúster Kafka real con Docker Compose, creando los tópicos que diseñaste aquí y produciendo/consumiendo mensajes reales.
- Revisa tu diseño de arquitectura del Paso 6 después de completar cada laboratorio subsiguiente, actualizándolo con el conocimiento práctico que vayas adquiriendo.

## Recursos Adicionales

- [Documentación oficial de Apache Kafka 4.0](https://kafka.apache.org/documentation/) — Referencia completa de configuración, APIs y conceptos arquitectónicos. Especialmente útil la sección "Design" para profundizar en los fundamentos.
- [Confluent Developer Portal](https://developer.confluent.io) — Tutoriales, cursos gratuitos, artículos técnicos y videos sobre Kafka y Confluent Platform. El curso "Apache Kafka Fundamentals" es complementario a este laboratorio.
- [Designing Event-Driven Systems — Ben Stopford (O'Reilly)](https://www.confluent.io/designing-event-driven-systems) — Libro gratuito que profundiza en los patrones Event Notification, Event-Carried State Transfer y Event Sourcing con ejemplos prácticos en Kafka.
- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum) — La propuesta técnica original de KRaft para entender las motivaciones y decisiones de diseño.
- [LinkedIn Engineering Blog — Kafka Origins](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) — El artículo clásico "The Log" de Jay Kreps que explica la filosofía detrás de Kafka y por qué el log append-only es tan poderoso.
- [Confluent Platform 8.0 — Notas de Versión](https://docs.confluent.io/platform/current/release-notes/index.html) — Para mantenerse actualizado sobre las últimas características y cambios en la plataforma que se usará en los laboratorios.
