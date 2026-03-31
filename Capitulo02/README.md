# Despliegue y Operación de Apache Kafka 4.0 con Docker Compose y Confluent Platform 8.0

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 168 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | Capítulo 2 – Instalación y Configuración de Kafka |
| **Tecnologías** | Apache Kafka 4.0, Confluent Platform 8.0, Docker Compose 2.x, KRaft |

---

## Descripción General

En este laboratorio desplegarás un clúster Apache Kafka 4.0 completamente funcional en modo KRaft (sin ZooKeeper) utilizando Docker Compose y las imágenes oficiales de Confluent Platform 8.0. A través de una serie de ejercicios progresivos, construirás el archivo de configuración desde cero, verificarás el estado del clúster, administrarás tópicos y realizarás tus primeras operaciones de producción y consumo de mensajes desde la línea de comandos.

Este laboratorio es la base práctica de todo el curso: el entorno que construirás aquí será reutilizado en los laboratorios posteriores. Comprender cómo funciona cada componente, cómo se comunican entre sí y cómo diagnosticar problemas de configuración te dará la confianza necesaria para operar Kafka en escenarios reales.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Desplegar un clúster Apache Kafka 4.0 en modo KRaft usando Docker Compose con Confluent Platform 8.0, verificando que todos los contenedores estén operativos.
- [ ] Analizar e interpretar la estructura del archivo `docker-compose.yml` identificando los parámetros críticos del broker: listeners, roles KRaft, factores de replicación y puertos.
- [ ] Verificar el estado y la salud del clúster Kafka utilizando comandos CLI (`kafka-metadata-quorum.sh`, `kafka-broker-api-versions.sh`) y análisis de logs de contenedores.
- [ ] Crear, listar, describir y eliminar tópicos Kafka con diferentes configuraciones usando `kafka-topics.sh` desde la línea de comandos.
- [ ] Producir y consumir mensajes básicos desde la terminal usando `kafka-console-producer.sh` y `kafka-console-consumer.sh`, explorando offsets y grupos de consumidores.

---

## Prerrequisitos

### Conocimientos Requeridos

- Haber completado el Laboratorio 1 (conceptos fundamentales de Kafka: tópicos, particiones, productores, consumidores, offsets).
- Comprensión básica del modelo publicación/suscripción de Kafka.
- Familiaridad con la terminal Linux/Unix: navegación de directorios, edición de archivos con `nano` o `vim`, gestión de procesos.
- Conocimiento básico de Docker: qué es un contenedor, qué es una imagen, comandos `docker ps`, `docker logs`.
- Comprensión básica de archivos YAML: indentación, listas, mapas clave-valor.

### Acceso y Permisos Requeridos

- Docker Engine 24.0 o superior instalado y en ejecución (`docker --version` debe funcionar sin errores).
- Docker Compose Plugin 2.20 o superior instalado (`docker compose version` debe funcionar).
- Permisos de administrador (`sudo` en Linux/macOS) para ejecutar contenedores Docker.
- Conexión a internet para descargar imágenes de Confluent Platform (aproximadamente 3-4 GB en total).
- Al menos 8 GB de RAM disponibles para Docker y 10 GB de espacio libre en disco.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación Mínima | Especificación Recomendada |
|------------|----------------------|---------------------------|
| CPU | 4 núcleos físicos, 64 bits, virtualización habilitada | 6+ núcleos |
| RAM | 8 GB disponibles para Docker | 16 GB |
| Disco | 10 GB libres | 20 GB libres |
| Red | 10 Mbps para descarga de imágenes | 50+ Mbps |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Motor de contenedores principal |
| Docker Compose Plugin | 2.20+ | Orquestación de múltiples contenedores |
| Confluent Platform (cp-kafka) | 8.0.0 | Broker Kafka 4.0 en modo KRaft |
| Confluent Schema Registry | 8.0.0 | Gestión centralizada de esquemas |
| Confluent Control Center | 8.0.0 | Interfaz web de administración |
| curl | 7.x+ | Verificación de APIs REST |
| Editor de texto | Cualquiera reciente | Edición de archivos YAML |

### Configuración Inicial del Entorno

Antes de comenzar los pasos del laboratorio, ejecuta los siguientes comandos para verificar que tu entorno está listo:

```bash
# Verificar versión de Docker
docker --version

# Verificar versión de Docker Compose
docker compose version

# Verificar que el daemon de Docker está en ejecución
docker info | grep "Server Version"

# Verificar espacio disponible en disco
df -h /

# Verificar memoria RAM disponible
free -h
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparar el Directorio de Trabajo del Laboratorio

**Objetivo:** Crear la estructura de directorios que contendrá todos los archivos de configuración del laboratorio, garantizando un entorno organizado y reproducible.

**Instrucciones:**

1. Abre una terminal y crea el directorio principal del laboratorio:

   ```bash
   mkdir -p ~/kafka-labs/lab-02
   cd ~/kafka-labs/lab-02
   ```

2. Crea los subdirectorios necesarios para organizar los archivos del laboratorio:

   ```bash
   mkdir -p config scripts data
   ```

3. Verifica la estructura creada:

   ```bash
   ls -la ~/kafka-labs/lab-02/
   ```

4. Establece este directorio como tu directorio de trabajo para todo el laboratorio:

   ```bash
   cd ~/kafka-labs/lab-02
   pwd
   ```

**Salida Esperada:**

```
/home/tu-usuario/kafka-labs/lab-02
```

```
total 0
drwxr-xr-x 5 usuario usuario  60 Jan 01 10:00 .
drwxr-xr-x 3 usuario usuario  20 Jan 01 10:00 ..
drwxr-xr-x 2 usuario usuario   6 Jan 01 10:00 config
drwxr-xr-x 2 usuario usuario   6 Jan 01 10:00 data
drwxr-xr-x 2 usuario usuario   6 Jan 01 10:00 scripts
```

**Verificación:**

- Confirma que el directorio `~/kafka-labs/lab-02` existe y contiene las tres subcarpetas.
- Confirma que `pwd` muestra la ruta correcta al directorio del laboratorio.

---

### Paso 2: Crear el Archivo docker-compose.yml del Clúster Kafka

**Objetivo:** Construir el archivo de configuración Docker Compose que define el entorno completo de Kafka con broker en modo KRaft, Schema Registry y Control Center, comprendiendo el propósito de cada parámetro.

**Instrucciones:**

1. Asegúrate de estar en el directorio del laboratorio:

   ```bash
   cd ~/kafka-labs/lab-02
   ```

2. Crea el archivo `docker-compose.yml` con el siguiente contenido. Puedes usar `nano`, `vim` o cualquier editor de texto:

   ```bash
   cat > docker-compose.yml << 'EOF'
   version: '3.8'

   services:

     # ─────────────────────────────────────────────
     # BROKER KAFKA (modo KRaft, sin ZooKeeper)
     # ─────────────────────────────────────────────
     broker:
       image: confluentinc/cp-kafka:8.0.0
       container_name: broker
       hostname: broker
       ports:
         - "9092:9092"
         - "9101:9101"
       environment:
         # Identificador único del nodo en el clúster
         KAFKA_NODE_ID: 1

         # Roles del nodo: actúa como broker Y controlador KRaft
         KAFKA_PROCESS_ROLES: 'broker,controller'

         # Quórum de controladores KRaft: ID@host:puerto
         KAFKA_CONTROLLER_QUORUM_VOTERS: '1@broker:29093'

         # Definición de todos los listeners disponibles
         KAFKA_LISTENERS: 'PLAINTEXT://broker:29092,CONTROLLER://broker:29093,PLAINTEXT_HOST://0.0.0.0:9092'

         # Mapeo de listeners a protocolos de seguridad
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'

         # Dirección que el broker anuncia a los clientes
         KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092'

         # Listener exclusivo para comunicación entre controladores KRaft
         KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'

         # Listener para comunicación interna entre brokers
         KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'

         # Factor de replicación para tópicos internos (1 para entorno de desarrollo)
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
         KAFKA_DEFAULT_REPLICATION_FACTOR: 1
         KAFKA_MIN_INSYNC_REPLICAS: 1

         # Número de particiones por defecto para nuevos tópicos
         KAFKA_NUM_PARTITIONS: 3

         # Directorio donde Kafka almacena los datos de los logs
         KAFKA_LOG_DIRS: '/var/lib/kafka/data'

         # Período de retención de mensajes: 7 días en milisegundos
         KAFKA_LOG_RETENTION_MS: 604800000

         # Tamaño máximo de retención por partición: 1 GB
         KAFKA_LOG_RETENTION_BYTES: 1073741824

         # Habilitar la eliminación de tópicos desde CLI
         KAFKA_DELETE_TOPIC_ENABLE: 'true'

         # Configuración JMX para métricas (se usará en laboratorios posteriores)
         KAFKA_JMX_PORT: 9101
         KAFKA_JMX_HOSTNAME: localhost

         # ID del clúster KRaft (UUID en Base64)
         CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'

       volumes:
         - kafka-data:/var/lib/kafka/data

       networks:
         - kafka-network

       healthcheck:
         test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
         interval: 30s
         timeout: 10s
         retries: 5
         start_period: 60s

     # ─────────────────────────────────────────────
     # SCHEMA REGISTRY
     # ─────────────────────────────────────────────
     schema-registry:
       image: confluentinc/cp-schema-registry:8.0.0
       container_name: schema-registry
       hostname: schema-registry
       depends_on:
         broker:
           condition: service_healthy
       ports:
         - "8081:8081"
       environment:
         SCHEMA_REGISTRY_HOST_NAME: schema-registry
         SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker:29092'
         SCHEMA_REGISTRY_LISTENERS: 'http://0.0.0.0:8081'
         SCHEMA_REGISTRY_KAFKASTORE_TOPIC_REPLICATION_FACTOR: 1

       networks:
         - kafka-network

       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:8081/subjects"]
         interval: 30s
         timeout: 10s
         retries: 5
         start_period: 45s

     # ─────────────────────────────────────────────
     # CONTROL CENTER (interfaz web de Confluent)
     # ─────────────────────────────────────────────
     control-center:
       image: confluentinc/cp-enterprise-control-center:8.0.0
       container_name: control-center
       hostname: control-center
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_healthy
       ports:
         - "9021:9021"
       environment:
         CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker:29092'
         CONTROL_CENTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
         CONTROL_CENTER_REPLICATION_FACTOR: 1
         CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
         CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
         CONFLUENT_METRICS_TOPIC_REPLICATION: 1
         PORT: 9021

       networks:
         - kafka-network

   # ─────────────────────────────────────────────
   # VOLÚMENES NOMBRADOS (persistencia entre reinicios)
   # ─────────────────────────────────────────────
   volumes:
     kafka-data:
       name: lab02-kafka-data

   # ─────────────────────────────────────────────
   # RED VIRTUAL COMPARTIDA
   # ─────────────────────────────────────────────
   networks:
     kafka-network:
       name: lab02-kafka-network
       driver: bridge
   EOF
   ```

3. Verifica que el archivo se creó correctamente y revisa su contenido:

   ```bash
   cat docker-compose.yml
   ```

4. Valida la sintaxis del archivo Docker Compose:

   ```bash
   docker compose config
   ```

**Salida Esperada de `docker compose config`:**

```yaml
name: lab-02
services:
  broker:
    container_name: broker
    environment:
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      ...
  schema-registry:
    ...
  control-center:
    ...
```

> Si `docker compose config` no muestra errores de sintaxis, el archivo es válido.

**Verificación:**

- El comando `docker compose config` debe ejecutarse sin errores.
- El archivo `docker-compose.yml` debe estar presente en el directorio `~/kafka-labs/lab-02/`.
- Identifica visualmente en el archivo los tres servicios: `broker`, `schema-registry` y `control-center`.

---

### Paso 3: Analizar los Parámetros Críticos del Broker Kafka

**Objetivo:** Comprender en profundidad el propósito de los parámetros de configuración más importantes del broker antes de levantarlo, para poder interpretar su comportamiento y diagnosticar problemas.

**Instrucciones:**

1. Crea un archivo de referencia con las anotaciones de los parámetros clave:

   ```bash
   cat > config/broker-params-reference.md << 'EOF'
   # Referencia de Parámetros Críticos del Broker Kafka

   ## Parámetros de Identidad y Rol KRaft

   | Parámetro | Valor en Lab | Descripción |
   |-----------|-------------|-------------|
   | KAFKA_NODE_ID | 1 | ID único del nodo en el clúster. Cada broker debe tener un ID diferente. |
   | KAFKA_PROCESS_ROLES | broker,controller | Roles del nodo. En producción se separan en nodos distintos. |
   | KAFKA_CONTROLLER_QUORUM_VOTERS | 1@broker:29093 | Lista de controladores para el quórum KRaft. Formato: ID@host:puerto |
   | CLUSTER_ID | MkU3OEVBNTcwNTJENDM2Qk | UUID del clúster en Base64. Debe ser único e inmutable. |

   ## Parámetros de Listeners (Comunicación de Red)

   | Listener | Puerto | Uso |
   |----------|--------|-----|
   | PLAINTEXT | 29092 | Comunicación interna entre contenedores Docker |
   | CONTROLLER | 29093 | Protocolo KRaft entre controladores |
   | PLAINTEXT_HOST | 9092 | Acceso desde clientes en el host (fuera de Docker) |

   ## Parámetros de Replicación y Tolerancia a Fallos

   | Parámetro | Valor | Significado |
   |-----------|-------|-------------|
   | KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR | 1 | RF del tópico __consumer_offsets. Usar >=3 en producción. |
   | KAFKA_DEFAULT_REPLICATION_FACTOR | 1 | RF por defecto para nuevos tópicos. |
   | KAFKA_MIN_INSYNC_REPLICAS | 1 | Mínimo de réplicas sincronizadas para aceptar escrituras. |

   ## Parámetros de Almacenamiento y Retención

   | Parámetro | Valor | Significado |
   |-----------|-------|-------------|
   | KAFKA_LOG_DIRS | /var/lib/kafka/data | Directorio de datos dentro del contenedor. |
   | KAFKA_LOG_RETENTION_MS | 604800000 | Retención de 7 días (7 * 24 * 60 * 60 * 1000 ms). |
   | KAFKA_LOG_RETENTION_BYTES | 1073741824 | Máximo 1 GB por partición. |
   | KAFKA_NUM_PARTITIONS | 3 | Particiones por defecto para nuevos tópicos. |
   EOF
   ```

2. Examina el cálculo de la retención en tiempo:

   ```bash
   # Verificar que 604800000 ms = 7 días
   echo "7 días en ms = $((7 * 24 * 60 * 60 * 1000)) ms"
   echo "1 GB en bytes = $((1024 * 1024 * 1024)) bytes"
   ```

3. Identifica los puertos que estarán expuestos en tu máquina local:

   ```bash
   grep -A 3 "ports:" docker-compose.yml
   ```

**Salida Esperada:**

```
7 días en ms = 604800000 ms
1 GB en bytes = 1073741824 bytes
```

```yaml
    ports:
      - "9092:9092"
      - "9101:9101"
...
    ports:
      - "8081:8081"
...
    ports:
      - "9021:9021"
```

**Verificación:**

- Confirma que el archivo `config/broker-params-reference.md` fue creado.
- Comprende la diferencia entre el puerto `29092` (interno Docker) y `9092` (externo al host).
- Comprende por qué `KAFKA_PROCESS_ROLES: 'broker,controller'` elimina la necesidad de ZooKeeper.

---

### Paso 4: Iniciar el Clúster Kafka con Docker Compose

**Objetivo:** Levantar todos los servicios del entorno Kafka en segundo plano y monitorear el proceso de arranque para confirmar que todos los contenedores inician correctamente.

**Instrucciones:**

1. Asegúrate de estar en el directorio correcto:

   ```bash
   cd ~/kafka-labs/lab-02
   ```

2. Descarga las imágenes Docker antes de levantar los servicios (esto puede tomar varios minutos dependiendo de tu conexión):

   ```bash
   docker compose pull
   ```

3. Levanta todos los servicios en segundo plano:

   ```bash
   docker compose up -d
   ```

4. Monitorea el estado de los contenedores mientras arrancan:

   ```bash
   # Verificar el estado de los contenedores (ejecutar varias veces durante el arranque)
   docker compose ps
   ```

5. Observa los logs del broker en tiempo real para confirmar el arranque exitoso:

   ```bash
   # Seguir los logs del broker (espera el mensaje de inicio exitoso, luego Ctrl+C)
   docker compose logs -f broker
   ```

   Busca en los logs el mensaje que indica que el broker está listo. Cuando veas algo similar a `[KafkaServer id=1] started`, presiona `Ctrl+C`.

6. Verifica que los tres contenedores están en estado `running`:

   ```bash
   docker compose ps
   ```

7. Verifica que los puertos están correctamente mapeados al host:

   ```bash
   docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
   ```

**Salida Esperada de `docker compose ps`:**

```
NAME               IMAGE                                               COMMAND                  SERVICE            CREATED          STATUS                    PORTS
broker             confluentinc/cp-kafka:8.0.0                        "/etc/confluent/dock…"   broker             2 minutes ago    Up 2 minutes (healthy)    0.0.0.0:9092->9092/tcp, 0.0.0.0:9101->9101/tcp
control-center     confluentinc/cp-enterprise-control-center:8.0.0    "/etc/confluent/dock…"   control-center     1 minute ago     Up 1 minute               0.0.0.0:9021->9021/tcp
schema-registry    confluentinc/cp-schema-registry:8.0.0              "/etc/confluent/dock…"   schema-registry    2 minutes ago    Up 2 minutes (healthy)    0.0.0.0:8081->8081/tcp
```

> **Nota:** El Control Center puede tardar entre 2 y 5 minutos en estar completamente operativo. Es normal que inicialmente muestre `starting` o `Up X seconds` sin el indicador `(healthy)`.

**Verificación:**

- Los tres contenedores deben mostrar estado `Up`.
- El broker debe mostrar `(healthy)` después de aproximadamente 60-90 segundos.
- Los puertos `9092`, `8081` y `9021` deben estar mapeados al host.

---

### Paso 5: Verificar el Estado del Clúster y el Quórum KRaft

**Objetivo:** Confirmar que el clúster Kafka está operativo y que el modo KRaft está funcionando correctamente, utilizando herramientas CLI incluidas en la imagen del broker.

**Instrucciones:**

1. Verifica la versión de la API del broker para confirmar que responde correctamente:

   ```bash
   docker exec broker kafka-broker-api-versions --bootstrap-server localhost:9092
   ```

2. Verifica el estado del quórum KRaft (el mecanismo de consenso que reemplaza a ZooKeeper):

   ```bash
   docker exec broker kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status
   ```

3. Consulta los metadatos del clúster para ver la información de los brokers registrados:

   ```bash
   docker exec broker kafka-metadata-quorum --bootstrap-server localhost:9092 describe --replication
   ```

4. Verifica el estado de Schema Registry usando su API REST:

   ```bash
   curl -s http://localhost:8081/subjects | python3 -m json.tool 2>/dev/null || curl -s http://localhost:8081/subjects
   ```

5. Verifica la conectividad básica al broker desde el host:

   ```bash
   # Listar los brokers disponibles
   docker exec broker kafka-broker-api-versions \
     --bootstrap-server localhost:9092 \
     2>&1 | grep "id:"
   ```

6. Revisa los logs de arranque del broker para identificar la secuencia de inicialización KRaft:

   ```bash
   docker compose logs broker | grep -E "(KRaft|KafkaServer|started|LEADER|FOLLOWER|OBSERVER)" | head -20
   ```

**Salida Esperada de `kafka-metadata-quorum --status`:**

```
ClusterId:              MkU3OEVBNTcwNTJENDM2Qk
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          XXX
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [1]
CurrentObservers:       []
```

**Salida Esperada de Schema Registry:**

```json
[]
```

> La respuesta `[]` es correcta: indica que no hay esquemas registrados aún, lo cual es esperado en un entorno recién inicializado.

**Verificación:**

- El comando `kafka-metadata-quorum describe --status` debe mostrar `LeaderId: 1`.
- `CurrentVoters` debe mostrar `[1]` (el único nodo del clúster).
- `MaxFollowerLag` debe ser `0`.
- La API REST de Schema Registry debe responder con `[]`.

---

### Paso 6: Explorar los Logs de Arranque del Broker

**Objetivo:** Desarrollar la habilidad de leer e interpretar los logs de Kafka para entender la secuencia de inicialización y detectar posibles problemas.

**Instrucciones:**

1. Revisa los logs completos del broker desde el inicio:

   ```bash
   docker compose logs broker 2>&1 | head -60
   ```

2. Busca mensajes clave de la secuencia de arranque KRaft:

   ```bash
   # Verificar la inicialización del almacenamiento KRaft
   docker compose logs broker 2>&1 | grep -i "kraft" | head -10
   ```

3. Busca el mensaje que confirma que el broker está listo para aceptar conexiones:

   ```bash
   docker compose logs broker 2>&1 | grep -E "started|KafkaServer|listening"
   ```

4. Verifica que no hay errores críticos en los logs:

   ```bash
   docker compose logs broker 2>&1 | grep -i "error\|exception\|fatal" | head -20
   ```

5. Revisa los logs de Schema Registry para confirmar su conexión exitosa al broker:

   ```bash
   docker compose logs schema-registry 2>&1 | grep -E "started|connected|listening|error" | head -15
   ```

6. Guarda un snapshot de los logs para referencia futura:

   ```bash
   docker compose logs broker > data/broker-startup-logs.txt
   echo "Logs guardados en data/broker-startup-logs.txt"
   wc -l data/broker-startup-logs.txt
   ```

**Salida Esperada:**

```
Logs guardados en data/broker-startup-logs.txt
XXX data/broker-startup-logs.txt
```

> Los logs deben mostrar la secuencia: inicialización KRaft → elección de líder → inicio de listeners → broker listo.

**Verificación:**

- No debe haber mensajes `FATAL` o `ERROR` relacionados con la inicialización.
- Los logs deben mostrar que el broker está escuchando en los puertos configurados.
- Schema Registry debe mostrar que se conectó exitosamente al broker.

---

### Paso 7: Crear Tópicos con Diferentes Configuraciones

**Objetivo:** Dominar el uso de `kafka-topics.sh` para crear tópicos con diferentes configuraciones de particiones y factor de replicación, y verificar su creación correctamente.

**Instrucciones:**

1. Lista los tópicos existentes en el clúster (inicialmente solo tópicos internos):

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list
   ```

2. Crea un tópico básico con la configuración por defecto:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --create \
     --topic mi-primer-topico \
     --partitions 3 \
     --replication-factor 1
   ```

3. Crea un tópico para eventos de usuarios con configuración específica:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --create \
     --topic eventos-usuarios \
     --partitions 6 \
     --replication-factor 1 \
     --config retention.ms=86400000 \
     --config max.message.bytes=1048576
   ```

4. Crea un tópico para transacciones financieras con retención extendida:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --create \
     --topic transacciones-financieras \
     --partitions 12 \
     --replication-factor 1 \
     --config retention.ms=2592000000 \
     --config compression.type=lz4
   ```

5. Crea un tópico con log compaction (útil para tablas de estado):

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --create \
     --topic catalogo-productos \
     --partitions 3 \
     --replication-factor 1 \
     --config cleanup.policy=compact \
     --config min.cleanable.dirty.ratio=0.5
   ```

6. Lista todos los tópicos para confirmar su creación:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list
   ```

7. Obtén una descripción detallada de todos los tópicos creados:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --describe \
     --topic mi-primer-topico,eventos-usuarios,transacciones-financieras,catalogo-productos
   ```

**Salida Esperada de `--list`:**

```
__consumer_offsets
catalogo-productos
eventos-usuarios
mi-primer-topico
transacciones-financieras
```

**Salida Esperada de `--describe`:**

```
Topic: mi-primer-topico         TopicId: XXXXXXXXXXXXXXXXXXXXXXXX  PartitionCount: 3       ReplicationFactor: 1    Configs:
        Topic: mi-primer-topico Partition: 0    Leader: 1       Replicas: 1     Isr: 1
        Topic: mi-primer-topico Partition: 1    Leader: 1       Replicas: 1     Isr: 1
        Topic: mi-primer-topico Partition: 2    Leader: 1       Replicas: 1     Isr: 1

Topic: eventos-usuarios         TopicId: XXXXXXXXXXXXXXXXXXXXXXXX  PartitionCount: 6       ReplicationFactor: 1    Configs: max.message.bytes=1048576,retention.ms=86400000
        Topic: eventos-usuarios Partition: 0    Leader: 1       Replicas: 1     Isr: 1
        ...

Topic: transacciones-financieras  TopicId: XXXXXXXXXXXXXXXXXXXXXXXX  PartitionCount: 12    ReplicationFactor: 1    Configs: compression.type=lz4,retention.ms=2592000000
        ...

Topic: catalogo-productos       TopicId: XXXXXXXXXXXXXXXXXXXXXXXX  PartitionCount: 3       ReplicationFactor: 1    Configs: cleanup.policy=compact,min.cleanable.dirty.ratio=0.5
        ...
```

**Verificación:**

- Los cuatro tópicos deben aparecer en la lista.
- Cada tópico debe mostrar el número correcto de particiones.
- Las configuraciones específicas (`retention.ms`, `compression.type`, `cleanup.policy`) deben aparecer en la descripción.
- El campo `Leader: 1` confirma que el broker con ID 1 es el líder de todas las particiones.

---

### Paso 8: Modificar la Configuración de un Tópico Existente

**Objetivo:** Aprender a modificar configuraciones de tópicos en caliente sin necesidad de recrearlos, una habilidad crítica para la administración de Kafka en producción.

**Instrucciones:**

1. Verifica la configuración actual del tópico `eventos-usuarios`:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --describe \
     --topic eventos-usuarios
   ```

2. Modifica el período de retención del tópico `eventos-usuarios` de 1 día a 3 días:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --alter \
     --entity-type topics \
     --entity-name eventos-usuarios \
     --add-config retention.ms=259200000
   ```

3. Añade una nueva configuración: límite de tamaño de segmento de log:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --alter \
     --entity-type topics \
     --entity-name eventos-usuarios \
     --add-config segment.bytes=536870912
   ```

4. Verifica las configuraciones actuales del tópico usando `kafka-configs`:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --describe \
     --entity-type topics \
     --entity-name eventos-usuarios
   ```

5. Elimina una configuración específica para volver al valor por defecto del broker:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --alter \
     --entity-type topics \
     --entity-name eventos-usuarios \
     --delete-config max.message.bytes
   ```

6. Confirma que la configuración fue eliminada:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --describe \
     --entity-type topics \
     --entity-name eventos-usuarios
   ```

**Salida Esperada:**

```
Dynamic configs for topic eventos-usuarios are:
  retention.ms=259200000 sensitive=false synonyms={DYNAMIC_TOPIC_CONFIG:retention.ms=259200000, ...}
  segment.bytes=536870912 sensitive=false synonyms={DYNAMIC_TOPIC_CONFIG:segment.bytes=536870912, ...}
```

> Nota que `max.message.bytes` ya no aparece después de eliminarlo.

**Verificación:**

- `retention.ms` debe mostrar `259200000` (3 días).
- `segment.bytes` debe mostrar `536870912` (512 MB).
- `max.message.bytes` no debe aparecer en la lista de configuraciones dinámicas.

---

### Paso 9: Producir Mensajes desde la Línea de Comandos

**Objetivo:** Utilizar `kafka-console-producer.sh` para enviar mensajes a tópicos Kafka desde la terminal, explorando diferentes modos de producción incluyendo mensajes con clave.

**Instrucciones:**

1. Produce mensajes simples (sin clave) al tópico `mi-primer-topico`:

   ```bash
   # Este comando abre un prompt interactivo. Escribe cada línea y presiona Enter.
   # Cuando termines, presiona Ctrl+C para salir.
   docker exec -it broker kafka-console-producer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico
   ```

   Cuando aparezca el prompt `>`, escribe los siguientes mensajes (uno por línea):
   ```
   Hola Kafka desde el Lab 02
   Este es mi primer mensaje
   Apache Kafka 4.0 con KRaft
   Confluent Platform 8.0
   Mensaje de prueba número 5
   ```
   Presiona `Ctrl+C` para salir del productor.

2. Produce mensajes con clave al tópico `eventos-usuarios` (formato: `clave:valor`):

   ```bash
   docker exec -it broker kafka-console-producer \
     --bootstrap-server localhost:9092 \
     --topic eventos-usuarios \
     --property "parse.key=true" \
     --property "key.separator=:"
   ```

   Escribe los siguientes mensajes con clave:
   ```
   user-001:{"evento":"login","usuario":"ana@ejemplo.com","timestamp":"2024-01-01T10:00:00"}
   user-002:{"evento":"compra","usuario":"carlos@ejemplo.com","producto":"laptop","monto":1200}
   user-001:{"evento":"logout","usuario":"ana@ejemplo.com","timestamp":"2024-01-01T10:30:00"}
   user-003:{"evento":"registro","usuario":"maria@ejemplo.com","timestamp":"2024-01-01T11:00:00"}
   user-002:{"evento":"login","usuario":"carlos@ejemplo.com","timestamp":"2024-01-01T11:15:00"}
   ```
   Presiona `Ctrl+C` para salir.

3. Produce mensajes al tópico `transacciones-financieras` con un script no interactivo:

   ```bash
   # Crear un archivo con mensajes de prueba
   cat > data/transacciones.txt << 'EOF'
   txn-001:{"id":"txn-001","monto":500.00,"moneda":"USD","tipo":"credito","cuenta":"ACC-1001"}
   txn-002:{"id":"txn-002","monto":1200.50,"moneda":"EUR","tipo":"debito","cuenta":"ACC-2002"}
   txn-003:{"id":"txn-003","monto":75.25,"moneda":"USD","tipo":"credito","cuenta":"ACC-1001"}
   txn-004:{"id":"txn-004","monto":3000.00,"moneda":"MXN","tipo":"transferencia","cuenta":"ACC-3003"}
   txn-005:{"id":"txn-005","monto":250.00,"moneda":"USD","tipo":"debito","cuenta":"ACC-2002"}
   EOF

   # Producir los mensajes desde el archivo
   docker exec -i broker kafka-console-producer \
     --bootstrap-server localhost:9092 \
     --topic transacciones-financieras \
     --property "parse.key=true" \
     --property "key.separator=:" \
     < data/transacciones.txt

   echo "Mensajes de transacciones enviados exitosamente."
   ```

**Salida Esperada:**

```
Mensajes de transacciones enviados exitosamente.
```

> Para los productores interactivos (pasos 1 y 2), no hay salida visible al escribir mensajes. El prompt `>` indica que el productor está listo para recibir input.

**Verificación:**

- El productor interactivo debe mostrar el prompt `>` sin errores.
- Al presionar `Ctrl+C`, debe salir limpiamente sin mostrar stack traces de error.
- El archivo `data/transacciones.txt` debe existir con 5 líneas.

---

### Paso 10: Consumir Mensajes desde la Línea de Comandos

**Objetivo:** Utilizar `kafka-console-consumer.sh` para leer mensajes de tópicos Kafka, explorando diferentes opciones como lectura desde el inicio, filtrado por partición y visualización de metadatos.

**Instrucciones:**

1. Consume todos los mensajes del tópico `mi-primer-topico` desde el inicio:

   ```bash
   # --from-beginning lee todos los mensajes históricos
   # --max-messages 10 limita la cantidad de mensajes a leer
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico \
     --from-beginning \
     --max-messages 10
   ```

2. Consume mensajes con sus claves y metadatos (partición y offset):

   ```bash
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic eventos-usuarios \
     --from-beginning \
     --property print.key=true \
     --property print.partition=true \
     --property print.offset=true \
     --property key.separator=" | " \
     --max-messages 10
   ```

3. Consume mensajes de una partición específica del tópico `transacciones-financieras`:

   ```bash
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic transacciones-financieras \
     --partition 0 \
     --offset earliest \
     --max-messages 5 \
     --property print.key=true \
     --property print.offset=true
   ```

4. Consume mensajes con un grupo de consumidores nombrado (esto registra el offset):

   ```bash
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico \
     --from-beginning \
     --group grupo-analisis-lab02 \
     --max-messages 5
   ```

5. Consume nuevamente con el mismo grupo (debe continuar desde donde se quedó):

   ```bash
   # Este comando no debería mostrar mensajes nuevos si ya se consumieron todos
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico \
     --group grupo-analisis-lab02 \
     --max-messages 5 \
     --timeout-ms 5000
   ```

**Salida Esperada del Paso 1:**

```
Hola Kafka desde el Lab 02
Este es mi primer mensaje
Apache Kafka 4.0 con KRaft
Confluent Platform 8.0
Mensaje de prueba número 5
Processed a total of 5 messages
```

**Salida Esperada del Paso 2:**

```
Partition:0 | user-001 | {"evento":"login","usuario":"ana@ejemplo.com","timestamp":"2024-01-01T10:00:00"}
Partition:1 | user-002 | {"evento":"compra","usuario":"carlos@ejemplo.com","producto":"laptop","monto":1200}
Partition:0 | user-001 | {"evento":"logout","usuario":"ana@ejemplo.com","timestamp":"2024-01-01T10:30:00"}
...
Processed a total of 5 messages
```

> Nota que los mensajes con la misma clave (`user-001`) van a la misma partición, lo cual es el comportamiento esperado del particionador por defecto de Kafka.

**Verificación:**

- El consumidor debe mostrar todos los mensajes producidos anteriormente.
- Los mensajes con la misma clave deben aparecer en la misma partición.
- El consumidor con grupo registrado debe continuar desde el último offset al ejecutarse nuevamente.

---

### Paso 11: Gestionar Grupos de Consumidores y Offsets

**Objetivo:** Explorar las herramientas de administración de grupos de consumidores para monitorear el lag, gestionar offsets y entender el estado de los consumidores.

**Instrucciones:**

1. Lista todos los grupos de consumidores registrados en el clúster:

   ```bash
   docker exec broker kafka-consumer-groups \
     --bootstrap-server localhost:9092 \
     --list
   ```

2. Describe el grupo `grupo-analisis-lab02` para ver el estado de sus offsets:

   ```bash
   docker exec broker kafka-consumer-groups \
     --bootstrap-server localhost:9092 \
     --describe \
     --group grupo-analisis-lab02
   ```

3. Produce más mensajes al tópico `mi-primer-topico` para generar lag:

   ```bash
   cat > data/mensajes-extra.txt << 'EOF'
   Mensaje adicional 1 para generar lag
   Mensaje adicional 2 para generar lag
   Mensaje adicional 3 para generar lag
   Mensaje adicional 4 para generar lag
   Mensaje adicional 5 para generar lag
   EOF

   docker exec -i broker kafka-console-producer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico \
     < data/mensajes-extra.txt

   echo "5 mensajes adicionales enviados."
   ```

4. Verifica el lag del grupo de consumidores después de producir más mensajes:

   ```bash
   docker exec broker kafka-consumer-groups \
     --bootstrap-server localhost:9092 \
     --describe \
     --group grupo-analisis-lab02
   ```

5. Reinicia los offsets del grupo al inicio del tópico (útil para reprocesar mensajes):

   ```bash
   docker exec broker kafka-consumer-groups \
     --bootstrap-server localhost:9092 \
     --reset-offsets \
     --group grupo-analisis-lab02 \
     --topic mi-primer-topico \
     --to-earliest \
     --execute
   ```

6. Verifica que el offset fue reiniciado:

   ```bash
   docker exec broker kafka-consumer-groups \
     --bootstrap-server localhost:9092 \
     --describe \
     --group grupo-analisis-lab02
   ```

7. Consume todos los mensajes desde el inicio con el grupo reiniciado:

   ```bash
   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic mi-primer-topico \
     --group grupo-analisis-lab02 \
     --from-beginning \
     --max-messages 10 \
     --timeout-ms 10000
   ```

**Salida Esperada del Paso 2 (antes del lag):**

```
GROUP                   TOPIC               PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
grupo-analisis-lab02    mi-primer-topico    0          2               2               0               -               -               -
grupo-analisis-lab02    mi-primer-topico    1          2               2               0               -               -               -
grupo-analisis-lab02    mi-primer-topico    2          1               1               0               -               -               -
```

**Salida Esperada del Paso 4 (con lag):**

```
GROUP                   TOPIC               PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
grupo-analisis-lab02    mi-primer-topico    0          2               4               2               -               -               -
grupo-analisis-lab02    mi-primer-topico    1          2               4               2               -               -               -
grupo-analisis-lab02    mi-primer-topico    2          1               2               1               -               -               -
```

**Verificación:**

- El lag debe aumentar después de producir mensajes adicionales.
- Después del reset de offsets, `CURRENT-OFFSET` debe mostrar `0` para todas las particiones.
- El consumo posterior al reset debe mostrar todos los mensajes desde el principio.

---

### Paso 12: Eliminar Tópicos y Verificar la Eliminación

**Objetivo:** Practicar la eliminación segura de tópicos Kafka y verificar que la operación se completa correctamente.

**Instrucciones:**

1. Verifica que la eliminación de tópicos está habilitada en la configuración del broker:

   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --describe \
     --entity-type brokers \
     --entity-name 1 \
     2>&1 | grep delete.topic || echo "delete.topic.enable está habilitado por defecto en Kafka 4.0"
   ```

2. Lista los tópicos actuales antes de eliminar:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list
   ```

3. Elimina el tópico `mi-primer-topico`:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --delete \
     --topic mi-primer-topico
   ```

4. Verifica que el tópico fue eliminado:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list
   ```

5. Intenta describir el tópico eliminado para confirmar que ya no existe:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --describe \
     --topic mi-primer-topico \
     2>&1 || echo "El tópico mi-primer-topico ha sido eliminado correctamente."
   ```

6. Verifica que los tópicos restantes siguen intactos:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --describe
   ```

**Salida Esperada del Paso 3:**

```
(no output - la eliminación exitosa no produce salida en Kafka 4.0)
```

**Salida Esperada del Paso 4:**

```
__consumer_offsets
catalogo-productos
eventos-usuarios
transacciones-financieras
```

> `mi-primer-topico` ya no debe aparecer en la lista.

**Verificación:**

- `mi-primer-topico` no debe aparecer en la lista de tópicos.
- Los tópicos `eventos-usuarios`, `transacciones-financieras` y `catalogo-productos` deben seguir existentes.
- El tópico interno `__consumer_offsets` debe seguir presente.

---

### Paso 13: Explorar el Control Center en el Navegador

**Objetivo:** Familiarizarse con la interfaz web de Confluent Control Center para administrar y monitorear el clúster Kafka de forma visual.

**Instrucciones:**

1. Verifica que el Control Center está completamente operativo:

   ```bash
   # Espera hasta que el Control Center responda correctamente
   for i in {1..12}; do
     STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9021)
     if [ "$STATUS" = "200" ]; then
       echo "Control Center está listo (HTTP $STATUS)"
       break
     else
       echo "Intento $i/12: Control Center respondiendo con HTTP $STATUS. Esperando 15 segundos..."
       sleep 15
     fi
   done
   ```

2. Abre tu navegador web y navega a la siguiente URL:

   ```
   http://localhost:9021
   ```

3. En el Control Center, realiza las siguientes exploraciones (documenta lo que observas):

   ```bash
   # Crea un archivo de notas para registrar tus observaciones
   cat > data/control-center-observations.md << 'EOF'
   # Observaciones del Control Center - Lab 02

   ## Panel Principal (Cluster Overview)
   - Número de brokers: ___
   - Número de tópicos: ___
   - Mensajes por segundo: ___

   ## Sección Topics
   - Tópicos visibles: ___
   - Tópico con más particiones: ___

   ## Sección Consumers
   - Grupos de consumidores registrados: ___
   - Lag total observado: ___

   ## Observaciones adicionales:
   ___
   EOF

   echo "Archivo de observaciones creado en data/control-center-observations.md"
   ```

4. Desde el Control Center, navega a **Topics → eventos-usuarios** y explora:
   - La distribución de mensajes por partición.
   - La configuración del tópico.
   - El throughput de mensajes.

5. Navega a **Consumers** y verifica el estado del grupo `grupo-analisis-lab02`.

6. Verifica el estado del clúster desde la API del Control Center:

   ```bash
   # La API del Control Center expone métricas en formato JSON
   curl -s http://localhost:9021/2.0/clusters | python3 -m json.tool 2>/dev/null | head -30 || \
   curl -s http://localhost:9021/2.0/clusters | head -100
   ```

**Salida Esperada del Paso 1:**

```
Control Center está listo (HTTP 200)
```

> Si el Control Center tarda más de 3 minutos en responder, revisa los logs con `docker compose logs control-center`.

**Verificación:**

- El Control Center debe cargarse correctamente en `http://localhost:9021`.
- Deben ser visibles los tópicos creados durante el laboratorio.
- El grupo de consumidores `grupo-analisis-lab02` debe aparecer en la sección Consumers.

---

### Paso 14: Crear un Script de Verificación del Estado del Clúster

**Objetivo:** Consolidar los conocimientos adquiridos creando un script reutilizable que verifique el estado completo del clúster Kafka de forma automatizada.

**Instrucciones:**

1. Crea el script de verificación de estado:

   ```bash
   cat > scripts/verificar-cluster.sh << 'EOF'
   #!/bin/bash
   # ============================================================
   # Script de Verificación del Estado del Clúster Kafka - Lab 02
   # ============================================================

   BOOTSTRAP_SERVER="localhost:9092"
   SCHEMA_REGISTRY_URL="http://localhost:8081"
   CONTROL_CENTER_URL="http://localhost:9021"

   echo "=============================================="
   echo "  VERIFICACIÓN DEL CLÚSTER KAFKA - LAB 02"
   echo "  $(date)"
   echo "=============================================="
   echo ""

   # 1. Verificar contenedores Docker
   echo "--- [1] ESTADO DE CONTENEDORES DOCKER ---"
   docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
   echo ""

   # 2. Verificar quórum KRaft
   echo "--- [2] ESTADO DEL QUÓRUM KRAFT ---"
   docker exec broker kafka-metadata-quorum \
     --bootstrap-server $BOOTSTRAP_SERVER \
     describe --status 2>/dev/null || echo "ERROR: No se pudo conectar al broker"
   echo ""

   # 3. Listar tópicos
   echo "--- [3] TÓPICOS EN EL CLÚSTER ---"
   docker exec broker kafka-topics \
     --bootstrap-server $BOOTSTRAP_SERVER \
     --list 2>/dev/null
   echo ""

   # 4. Verificar Schema Registry
   echo "--- [4] ESTADO DE SCHEMA REGISTRY ---"
   SR_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SCHEMA_REGISTRY_URL/subjects)
   if [ "$SR_STATUS" = "200" ]; then
     echo "Schema Registry: OPERATIVO (HTTP $SR_STATUS)"
     echo "Esquemas registrados: $(curl -s $SCHEMA_REGISTRY_URL/subjects)"
   else
     echo "Schema Registry: ERROR (HTTP $SR_STATUS)"
   fi
   echo ""

   # 5. Verificar Control Center
   echo "--- [5] ESTADO DEL CONTROL CENTER ---"
   CC_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $CONTROL_CENTER_URL)
   if [ "$CC_STATUS" = "200" ]; then
     echo "Control Center: OPERATIVO (HTTP $CC_STATUS)"
     echo "URL: $CONTROL_CENTER_URL"
   else
     echo "Control Center: NO DISPONIBLE (HTTP $CC_STATUS)"
   fi
   echo ""

   # 6. Verificar grupos de consumidores
   echo "--- [6] GRUPOS DE CONSUMIDORES ---"
   docker exec broker kafka-consumer-groups \
     --bootstrap-server $BOOTSTRAP_SERVER \
     --list 2>/dev/null
   echo ""

   echo "=============================================="
   echo "  VERIFICACIÓN COMPLETADA"
   echo "=============================================="
   EOF

   chmod +x scripts/verificar-cluster.sh
   ```

2. Ejecuta el script de verificación:

   ```bash
   cd ~/kafka-labs/lab-02
   ./scripts/verificar-cluster.sh
   ```

3. Guarda la salida del script para referencia:

   ```bash
   ./scripts/verificar-cluster.sh > data/estado-cluster-$(date +%Y%m%d-%H%M%S).txt
   echo "Estado del clúster guardado en data/"
   ls -la data/estado-cluster-*.txt
   ```

**Salida Esperada:**

```
==============================================
  VERIFICACIÓN DEL CLÚSTER KAFKA - LAB 02
  Mon Jan 01 10:00:00 UTC 2024
==============================================

--- [1] ESTADO DE CONTENEDORES DOCKER ---
NAME               STATUS                    PORTS
broker             Up 45 minutes (healthy)   0.0.0.0:9092->9092/tcp, 0.0.0.0:9101->9101/tcp
control-center     Up 44 minutes             0.0.0.0:9021->9021/tcp
schema-registry    Up 44 minutes (healthy)   0.0.0.0:8081->8081/tcp

--- [2] ESTADO DEL QUÓRUM KRAFT ---
ClusterId:              MkU3OEVBNTcwNTJENDM2Qk
LeaderId:               1
LeaderEpoch:            1
...

--- [3] TÓPICOS EN EL CLÚSTER ---
__consumer_offsets
catalogo-productos
eventos-usuarios
transacciones-financieras

--- [4] ESTADO DE SCHEMA REGISTRY ---
Schema Registry: OPERATIVO (HTTP 200)
Esquemas registrados: []

--- [5] ESTADO DEL CONTROL CENTER ---
Control Center: OPERATIVO (HTTP 200)
URL: http://localhost:9021

--- [6] GRUPOS DE CONSUMIDORES ---
grupo-analisis-lab02

==============================================
  VERIFICACIÓN COMPLETADA
==============================================
```

**Verificación:**

- El script debe ejecutarse sin errores.
- Todos los componentes deben aparecer como operativos.
- Los tópicos y grupos de consumidores creados durante el laboratorio deben aparecer en la salida.

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El archivo `docker-compose.yml` fue creado correctamente y pasa la validación de `docker compose config` sin errores.
- [ ] Los tres contenedores (`broker`, `schema-registry`, `control-center`) están en estado `Up` y el broker muestra `(healthy)`.
- [ ] El comando `kafka-metadata-quorum describe --status` muestra `LeaderId: 1` y `MaxFollowerLag: 0`.
- [ ] Se crearon exitosamente los tópicos `eventos-usuarios`, `transacciones-financieras` y `catalogo-productos` con las configuraciones especificadas.
- [ ] Se produjeron mensajes a los tópicos y se consumieron correctamente con `kafka-console-consumer`.
- [ ] El grupo de consumidores `grupo-analisis-lab02` aparece en la lista de grupos y su lag puede ser monitoreado.
- [ ] El reset de offsets se ejecutó correctamente y los mensajes pudieron ser reprocesados.
- [ ] El Control Center es accesible en `http://localhost:9021` y muestra los tópicos y consumidores.
- [ ] El script `scripts/verificar-cluster.sh` se ejecuta correctamente y reporta todos los componentes operativos.
- [ ] El tópico `mi-primer-topico` fue eliminado exitosamente.

### Procedimiento de Pruebas

1. **Prueba de conectividad del broker:**
   ```bash
   docker exec broker kafka-broker-api-versions --bootstrap-server localhost:9092 2>&1 | grep "Supported API versions"
   ```
   **Resultado Esperado:** Listado de versiones de la API de Kafka sin errores de conexión.

2. **Prueba del quórum KRaft:**
   ```bash
   docker exec broker kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status 2>&1 | grep "LeaderId"
   ```
   **Resultado Esperado:** `LeaderId:               1`

3. **Prueba de creación y consumo de mensajes:**
   ```bash
   echo "mensaje-de-prueba-final" | docker exec -i broker kafka-console-producer \
     --bootstrap-server localhost:9092 \
     --topic eventos-usuarios

   docker exec broker kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic eventos-usuarios \
     --from-beginning \
     --max-messages 1 \
     --timeout-ms 5000 2>/dev/null | grep -c "mensaje\|evento\|user\|txn" && echo "PRUEBA EXITOSA: Mensajes producidos y consumidos correctamente."
   ```
   **Resultado Esperado:** `PRUEBA EXITOSA: Mensajes producidos y consumidos correctamente.`

4. **Prueba de Schema Registry:**
   ```bash
   curl -s -o /dev/null -w "Schema Registry HTTP Status: %{http_code}\n" http://localhost:8081/subjects
   ```
   **Resultado Esperado:** `Schema Registry HTTP Status: 200`

5. **Prueba de persistencia de datos:**
   ```bash
   docker volume inspect lab02-kafka-data | grep -E "Mountpoint|Name"
   ```
   **Resultado Esperado:** Muestra el punto de montaje del volumen nombrado `lab02-kafka-data`.

6. **Prueba de configuración del tópico:**
   ```bash
   docker exec broker kafka-configs \
     --bootstrap-server localhost:9092 \
     --describe \
     --entity-type topics \
     --entity-name eventos-usuarios 2>&1 | grep "retention.ms"
   ```
   **Resultado Esperado:** `retention.ms=259200000`

---

## Solución de Problemas

### Issue 1: El Broker No Inicia - Error de CLUSTER_ID

**Síntomas:**
- El contenedor `broker` se reinicia continuamente (`Restarting` en `docker compose ps`).
- Los logs muestran: `ERROR Fatal error during KafkaServer startup` o `InconsistentClusterIdException`.

**Causa:**
El volumen Docker ya contiene datos de un clúster con un CLUSTER_ID diferente. Esto ocurre cuando se cambia el CLUSTER_ID en el `docker-compose.yml` sin eliminar el volumen de datos previo.

**Solución:**
```bash
# Detener todos los contenedores
docker compose down

# Eliminar el volumen de datos (ADVERTENCIA: esto borra todos los datos de Kafka)
docker volume rm lab02-kafka-data

# Reiniciar el entorno
docker compose up -d

# Verificar que el broker inicia correctamente
docker compose logs -f broker
```

---

### Issue 2: Error de Conexión - "Connection Refused" al Puerto 9092

**Síntomas:**
- Los comandos `kafka-topics`, `kafka-console-producer` o `kafka-console-consumer` fallan con `Connection refused`.
- `docker compose ps` muestra el broker como `Up` pero no `(healthy)`.

**Causa:**
El broker puede estar aún inicializándose. El proceso de arranque de Kafka puede tardar entre 30 y 90 segundos. También puede ser un problema de configuración de listeners.

**Solución:**
```bash
# Verificar el estado de salud del broker
docker compose ps broker

# Revisar los logs para identificar el problema
docker compose logs broker 2>&1 | tail -30

# Verificar que el puerto 9092 está escuchando
docker exec broker ss -tlnp | grep 9092

# Si el puerto no está disponible, esperar y reintentar
sleep 30 && docker exec broker kafka-broker-api-versions --bootstrap-server localhost:9092
```

---

### Issue 3: Schema Registry No Puede Conectarse al Broker

**Síntomas:**
- Schema Registry muestra estado `Restarting` o los logs muestran `Connection refused` al intentar conectarse a `broker:29092`.
- `curl http://localhost:8081/subjects` devuelve error de conexión.

**Causa:**
Schema Registry intenta conectarse al broker antes de que este esté completamente listo. La configuración `depends_on` con `condition: service_healthy` debería prevenir esto, pero puede fallar si el healthcheck del broker no está correctamente configurado.

**Solución:**
```bash
# Verificar el estado del broker primero
docker compose ps broker

# Si el broker está healthy, reiniciar Schema Registry
docker compose restart schema-registry

# Esperar 30 segundos y verificar
sleep 30
curl -s http://localhost:8081/subjects
docker compose logs schema-registry 2>&1 | tail -20
```

---

### Issue 4: Control Center Muestra "Cluster Unavailable"

**Síntomas:**
- El Control Center carga en el navegador pero muestra un error indicando que el clúster no está disponible.
- La URL `http://localhost:9021` muestra una pantalla de error o de carga infinita.

**Causa:**
El Control Center requiere varios minutos para inicializar sus tópicos internos y conectarse al broker. También puede ser un problema de memoria insuficiente.

**Solución:**
```bash
# Verificar los logs del Control Center
docker compose logs control-center 2>&1 | tail -40

# Verificar el uso de memoria de los contenedores
docker stats --no-stream

# Si hay problemas de memoria, reiniciar el Control Center
docker compose restart control-center

# Esperar 3-5 minutos y volver a verificar
sleep 180
curl -s -o /dev/null -w "%{http_code}" http://localhost:9021
```

---

### Issue 5: Error al Producir Mensajes - "Topic Not Found"

**Síntomas:**
- `kafka-console-producer` muestra `UNKNOWN_TOPIC_OR_PARTITION` al intentar enviar mensajes.
- El tópico aparece en la lista pero no acepta mensajes.

**Causa:**
El tópico puede haberse eliminado accidentalmente, o el nombre del tópico tiene un error tipográfico.

**Solución:**
```bash
# Verificar que el tópico existe
docker exec broker kafka-topics \
  --bootstrap-server localhost:9092 \
  --list | grep eventos-usuarios

# Si no existe, recrearlo
docker exec broker kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic eventos-usuarios \
  --partitions 6 \
  --replication-factor 1

# Verificar la creación
docker exec broker kafka-topics \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic eventos-usuarios
```

---

### Issue 6: docker compose up Falla por Falta de Memoria

**Síntomas:**
- Los contenedores se detienen inesperadamente con código de error `137` (OOMKilled).
- `docker compose ps` muestra contenedores en estado `Exited (137)`.

**Causa:**
El sistema no tiene suficiente memoria RAM disponible para ejecutar todos los contenedores simultáneamente.

**Solución:**
```bash
# Verificar el estado de memoria del sistema
free -h

# Verificar qué contenedor fue terminado por OOM
docker inspect broker | grep -i "oomkilled"

# Añadir límites de memoria al docker-compose.yml para el Control Center
# (que es el componente más pesado)
# Edita el docker-compose.yml y añade bajo 'control-center':
#   deploy:
#     resources:
#       limits:
#         memory: 2G

# Alternativamente, detener aplicaciones innecesarias del sistema
# y reiniciar el entorno
docker compose down
docker compose up -d broker schema-registry  # Iniciar solo los servicios esenciales
```

---

## Limpieza

Al finalizar el laboratorio, tienes dos opciones dependiendo de si continuarás con el Laboratorio 3:

**Opción A: Conservar los datos para el próximo laboratorio (RECOMENDADO)**

```bash
cd ~/kafka-labs/lab-02

# Detener los contenedores SIN eliminar los volúmenes de datos
docker compose stop

# Verificar que los contenedores están detenidos pero los volúmenes persisten
docker compose ps
docker volume ls | grep lab02
```

**Opción B: Limpieza completa (elimina todos los datos)**

```bash
cd ~/kafka-labs/lab-02

# Detener y eliminar contenedores, redes Y volúmenes
docker compose down -v

# Verificar que los volúmenes fueron eliminados
docker volume ls | grep lab02

# Limpiar imágenes Docker si deseas liberar espacio en disco (opcional)
# ADVERTENCIA: Esto requiere volver a descargar las imágenes en el próximo laboratorio
# docker rmi confluentinc/cp-kafka:8.0.0
# docker rmi confluentinc/cp-schema-registry:8.0.0
# docker rmi confluentinc/cp-enterprise-control-center:8.0.0
```

> ⚠️ **Advertencia:** Usa `docker compose down -v` SOLO si estás seguro de que no necesitas los datos. Los laboratorios 4, 5 y 6 son secuenciales y dependen de datos generados en sesiones anteriores. Si planeas continuar con el curso, usa siempre `docker compose stop` en lugar de `docker compose down -v`.

> ⚠️ **Advertencia de Seguridad:** El entorno configurado en este laboratorio no tiene autenticación ni TLS habilitados. Esta configuración es **EXCLUSIVAMENTE** para entornos locales de desarrollo y aprendizaje. **NUNCA** repliques esta configuración en un entorno de producción.

---

## Resumen

### Lo que Lograste

- **Construiste desde cero** un archivo `docker-compose.yml` completo para desplegar Apache Kafka 4.0 en modo KRaft con Confluent Platform 8.0, incluyendo broker, Schema Registry y Control Center.
- **Levantaste y verificaste** un clúster Kafka funcional usando Docker Compose, confirmando el estado de todos los componentes mediante herramientas CLI y la API REST.
- **Analizaste en profundidad** los parámetros críticos de configuración del broker: listeners, roles KRaft, factores de replicación, políticas de retención y configuración de almacenamiento.
- **Administraste tópicos** con diferentes configuraciones usando `kafka-topics.sh` y `kafka-configs.sh`, incluyendo creación, modificación, descripción y eliminación.
- **Produjiste y consumiste mensajes** desde la línea de comandos usando `kafka-console-producer.sh` y `kafka-console-consumer.sh`, explorando mensajes con y sin clave.
- **Gestionaste grupos de consumidores** con `kafka-consumer-groups.sh`, monitoreando el lag y realizando resets de offsets para reprocesar mensajes.
- **Creaste un script de verificación** reutilizable que automatiza la comprobación del estado completo del clúster.

### Conceptos Clave Aprendidos

- **Docker Compose como orquestador de entornos Kafka:** Un único archivo YAML puede definir toda la arquitectura de servicios interdependientes, simplificando enormemente el despliegue y la reproducibilidad.
- **Listeners en Kafka:** La distinción entre listeners internos (Docker) y externos (host) es fundamental para que los clientes puedan conectarse correctamente según su ubicación en la red.
- **Modo KRaft sin ZooKeeper:** La variable `KAFKA_PROCESS_ROLES: 'broker,controller'` es la clave que habilita la arquitectura moderna de Kafka 4.0, eliminando la dependencia de ZooKeeper y simplificando la operación del clúster.
- **Gestión de offsets y lag:** El lag de un grupo de consumidores es la diferencia entre el último offset producido y el último offset consumido; monitorearlo es esencial para garantizar el procesamiento en tiempo real.
- **Volúmenes nombrados para persistencia:** Usar `docker compose stop` en lugar de `docker compose down -v` preserva los datos entre sesiones de trabajo.

### Próximos Pasos

- **Laboratorio 3:** Explorarás la configuración avanzada del broker, incluyendo políticas de retención, compresión de mensajes y configuración de replicación en un clúster multi-broker.
- **Práctica recomendada:** Experimenta modificando el `docker-compose.yml` para cambiar el número de particiones por defecto o el período de retención, y observa cómo afecta el comportamiento del clúster.
- **Exploración adicional:** Navega por el Control Center en `http://localhost:9021` y explora las métricas de rendimiento, la distribución de mensajes por partición y las configuraciones de tópicos desde la interfaz gráfica.

---

## Recursos Adicionales

- **Documentación oficial de Docker Compose** - Referencia completa de todas las opciones del archivo `docker-compose.yml`: [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)
- **Repositorio cp-all-in-one de Confluent** - Ejemplos oficiales de Docker Compose para diferentes escenarios de Confluent Platform: [https://github.com/confluentinc/cp-all-in-one](https://github.com/confluentinc/cp-all-in-one)
- **Documentación de Confluent Platform 8.0** - Guía completa de instalación, configuración y operación: [https://docs.confluent.io/platform/current/](https://docs.confluent.io/platform/current/)
- **Documentación KRaft de Apache Kafka** - Arquitectura del modo KRaft y proceso de migración desde ZooKeeper: [https://kafka.apache.org/documentation/#kraft](https://kafka.apache.org/documentation/#kraft)
- **Referencia de configuración del broker Kafka** - Todos los parámetros configurables del broker con sus valores por defecto: [https://kafka.apache.org/documentation/#brokerconfigs](https://kafka.apache.org/documentation/#brokerconfigs)
- **Referencia de kafka-topics.sh** - Documentación completa de la herramienta de administración de tópicos: [https://docs.confluent.io/platform/current/tools/kafka-topics.html](https://docs.confluent.io/platform/current/tools/kafka-topics.html)
