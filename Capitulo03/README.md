# Administración Operacional de Kafka — Particiones, Replicación, Retención y Kafka Connect

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 168 minutos |
| **Complejidad** | Difícil |
| **Nivel Bloom** | Aplicar |
| **Módulo** | Capítulo 3 — Administración Avanzada de Kafka |

---

## Descripción General

En este laboratorio transformarás un entorno Kafka de broker único en un clúster multi-broker de producción con tres nodos, experimentando de primera mano cómo funcionan la replicación, la tolerancia a fallos y la elección automática de líderes. A lo largo de las actividades configurarás políticas de retención, compresión de mensajes, y ajustarás parámetros críticos de productores y consumidores para entender su impacto en rendimiento y confiabilidad.

Este laboratorio tiene relevancia directa en entornos empresariales reales: la mayoría de los incidentes en producción con Kafka están relacionados con una configuración incorrecta de `acks`, `min.insync.replicas`, o políticas de retención mal dimensionadas. Al finalizar habrás simulado fallos de broker y observado la recuperación automática del clúster, consolidando la comprensión teórica en experiencia práctica. Finalmente, desplegarás Kafka Connect en modo distribuido y explorarás su API REST, preparándote para los laboratorios especializados de integración con bases de datos.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Desplegar un clúster Kafka de 3 brokers usando Docker Compose con modo KRaft y verificar la distribución de líderes y réplicas ISR mediante `kafka-topics.sh --describe`.
- [ ] Simular la caída de un broker y observar la elección automática de nuevo líder, verificando la recuperación del ISR tras el reinicio del broker.
- [ ] Configurar tópicos con políticas de retención por tiempo (`retention.ms`) y por tamaño (`retention.bytes`) usando `kafka-configs.sh` y validar su comportamiento.
- [ ] Aplicar y comparar diferentes algoritmos de compresión (`gzip`, `snappy`, `lz4`, `zstd`) en productores y medir el impacto en el tamaño de los mensajes.
- [ ] Crear productores con configuraciones de `acks` (0, 1, all), `batch.size` y `linger.ms`, y analizar el comportamiento ante fallos de broker.
- [ ] Manipular offsets de grupos de consumidores con `kafka-consumer-groups.sh --reset-offsets` y verificar el rebalanceo del grupo.
- [ ] Desplegar Kafka Connect en modo distribuido y configurar un conector FileStream Source mediante la API REST.
- [ ] Utilizar Confluent Control Center para monitorear el estado del clúster, tópicos, conectores y métricas de rendimiento.

---

## Prerrequisitos

### Conocimiento Requerido

- Haber completado el Laboratorio 2 con un clúster Kafka básico funcionando en Docker.
- Comprensión del modelo de particiones y replicación de Kafka (Lección 3.1): conceptos de ISR, líder, seguidor y factor de replicación.
- Familiaridad con la CLI de Kafka: `kafka-topics.sh`, `kafka-console-producer.sh`, `kafka-console-consumer.sh`, `kafka-consumer-groups.sh`.
- Conocimiento básico de Docker Compose: levantar, detener y eliminar contenedores y volúmenes.
- Comprensión básica de conceptos de sistemas distribuidos: tolerancia a fallos, consistencia vs. disponibilidad (teorema CAP).

### Acceso Requerido

- Acceso a una terminal con permisos de administrador (`sudo` en Linux/macOS o WSL2 en Windows).
- Docker Engine 24.0+ y Docker Compose plugin v2.20+ instalados y operativos.
- Al menos 8 GB de RAM disponibles para Docker (se recomiendan 12 GB para este laboratorio).
- Espacio en disco de al menos 5 GB libres para imágenes y volúmenes de este laboratorio.
- Acceso a internet para descargar imágenes de Confluent Platform 8.0 (si no están en caché local).

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación Mínima | Especificación Recomendada |
|------------|----------------------|---------------------------|
| CPU | 4 núcleos físicos | 6-8 núcleos |
| RAM disponible para Docker | 8 GB | 12-16 GB |
| Espacio en disco libre | 5 GB | 10 GB |
| Conexión a internet | 10 Mbps | 50 Mbps |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Motor de contenedores principal |
| Docker Compose | 2.20+ (plugin v2) | Orquestación del clúster multi-broker |
| Confluent Platform | 8.0 (Kafka 4.0) | Brokers Kafka, Control Center, Kafka Connect |
| curl | 7.x+ | Interacción con API REST de Kafka Connect |
| jq | 1.6+ | Formateo de respuestas JSON de la API REST |

### Configuración Inicial

Antes de comenzar, crea el directorio de trabajo para este laboratorio y verifica que Docker esté operativo:

```bash
# Crear directorio de trabajo del laboratorio
mkdir -p ~/kafka-lab03 && cd ~/kafka-lab03

# Crear subdirectorios necesarios
mkdir -p config data/connect-input monitoring/prometheus monitoring/grafana

# Verificar que Docker está en ejecución
docker info | grep -E "Server Version|Total Memory|CPUs"

# Verificar versión de Docker Compose
docker compose version

# Verificar que jq está instalado (necesario para formatear JSON)
jq --version || (echo "Instalando jq..." && sudo apt-get install -y jq 2>/dev/null || brew install jq 2>/dev/null)
```

---

## Instrucciones Paso a Paso

### Paso 1: Crear el archivo Docker Compose para el Clúster Multi-Broker

**Objetivo:** Definir la infraestructura completa del laboratorio: 3 brokers Kafka en modo KRaft, Confluent Control Center y Kafka Connect en modo distribuido.

**Instrucciones:**

1. Crea el archivo `docker-compose.yml` en el directorio `~/kafka-lab03`:

   ```bash
   cd ~/kafka-lab03
   cat > docker-compose.yml << 'EOF'
   version: '3.8'
   
   networks:
     kafka-net:
       driver: bridge
   
   volumes:
     kafka1-data:
     kafka2-data:
     kafka3-data:
     connect-plugins:
   
   services:
   
     # ─────────────────────────────────────────
     # BROKER 1 — También actúa como controlador
     # ─────────────────────────────────────────
     kafka1:
       image: confluentinc/cp-kafka:8.0.0
       hostname: kafka1
       container_name: kafka1
       networks:
         - kafka-net
       ports:
         - "9092:9092"
         - "9101:9101"
       volumes:
         - kafka1-data:/var/lib/kafka/data
       environment:
         # Identificación del nodo
         KAFKA_NODE_ID: 1
         KAFKA_PROCESS_ROLES: 'broker,controller'
         KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka1:29093,2@kafka2:29093,3@kafka3:29093'
         # Listeners
         KAFKA_LISTENERS: 'PLAINTEXT://kafka1:29092,CONTROLLER://kafka1:29093,PLAINTEXT_HOST://0.0.0.0:9092'
         KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka1:29092,PLAINTEXT_HOST://localhost:9092'
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
         KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
         KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
         # Configuración del clúster
         CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
         KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
         # Configuración de replicación por defecto
         KAFKA_DEFAULT_REPLICATION_FACTOR: 3
         KAFKA_MIN_INSYNC_REPLICAS: 2
         KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE: 'false'
         # JMX para métricas
         KAFKA_JMX_PORT: 9101
         KAFKA_JMX_HOSTNAME: localhost
         # Logs
         KAFKA_LOG_DIRS: '/var/lib/kafka/data'
         KAFKA_NUM_PARTITIONS: 3
   
     # ─────────────────────────────────────────
     # BROKER 2
     # ─────────────────────────────────────────
     kafka2:
       image: confluentinc/cp-kafka:8.0.0
       hostname: kafka2
       container_name: kafka2
       networks:
         - kafka-net
       ports:
         - "9093:9093"
         - "9102:9102"
       volumes:
         - kafka2-data:/var/lib/kafka/data
       environment:
         KAFKA_NODE_ID: 2
         KAFKA_PROCESS_ROLES: 'broker,controller'
         KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka1:29093,2@kafka2:29093,3@kafka3:29093'
         KAFKA_LISTENERS: 'PLAINTEXT://kafka2:29092,CONTROLLER://kafka2:29093,PLAINTEXT_HOST://0.0.0.0:9093'
         KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka2:29092,PLAINTEXT_HOST://localhost:9093'
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
         KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
         KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
         CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
         KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
         KAFKA_DEFAULT_REPLICATION_FACTOR: 3
         KAFKA_MIN_INSYNC_REPLICAS: 2
         KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE: 'false'
         KAFKA_JMX_PORT: 9102
         KAFKA_JMX_HOSTNAME: localhost
         KAFKA_LOG_DIRS: '/var/lib/kafka/data'
         KAFKA_NUM_PARTITIONS: 3
   
     # ─────────────────────────────────────────
     # BROKER 3
     # ─────────────────────────────────────────
     kafka3:
       image: confluentinc/cp-kafka:8.0.0
       hostname: kafka3
       container_name: kafka3
       networks:
         - kafka-net
       ports:
         - "9094:9094"
         - "9103:9103"
       volumes:
         - kafka3-data:/var/lib/kafka/data
       environment:
         KAFKA_NODE_ID: 3
         KAFKA_PROCESS_ROLES: 'broker,controller'
         KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka1:29093,2@kafka2:29093,3@kafka3:29093'
         KAFKA_LISTENERS: 'PLAINTEXT://kafka3:29092,CONTROLLER://kafka3:29093,PLAINTEXT_HOST://0.0.0.0:9094'
         KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka3:29092,PLAINTEXT_HOST://localhost:9094'
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
         KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
         KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
         CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
         KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
         KAFKA_DEFAULT_REPLICATION_FACTOR: 3
         KAFKA_MIN_INSYNC_REPLICAS: 2
         KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE: 'false'
         KAFKA_JMX_PORT: 9103
         KAFKA_JMX_HOSTNAME: localhost
         KAFKA_LOG_DIRS: '/var/lib/kafka/data'
         KAFKA_NUM_PARTITIONS: 3
   
     # ─────────────────────────────────────────
     # KAFKA CONNECT — Modo distribuido
     # ─────────────────────────────────────────
     kafka-connect:
       image: confluentinc/cp-kafka-connect:8.0.0
       hostname: kafka-connect
       container_name: kafka-connect
       networks:
         - kafka-net
       ports:
         - "8083:8083"
       volumes:
         - connect-plugins:/usr/share/java/kafka-connect-plugins
         - ./data/connect-input:/data/input
       depends_on:
         - kafka1
         - kafka2
         - kafka3
       environment:
         CONNECT_BOOTSTRAP_SERVERS: 'kafka1:29092,kafka2:29092,kafka3:29092'
         CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
         CONNECT_REST_PORT: 8083
         CONNECT_GROUP_ID: 'connect-cluster-lab03'
         CONNECT_CONFIG_STORAGE_TOPIC: '_connect-configs'
         CONNECT_OFFSET_STORAGE_TOPIC: '_connect-offsets'
         CONNECT_STATUS_STORAGE_TOPIC: '_connect-status'
         CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
         CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
         CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
         CONNECT_KEY_CONVERTER: 'org.apache.kafka.connect.storage.StringConverter'
         CONNECT_VALUE_CONVERTER: 'org.apache.kafka.connect.storage.StringConverter'
         CONNECT_PLUGIN_PATH: '/usr/share/java,/usr/share/confluent-hub-components,/usr/share/java/kafka-connect-plugins'
         CONNECT_LOG4J_LOGGERS: 'org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR'
   
     # ─────────────────────────────────────────
     # CONFLUENT CONTROL CENTER
     # ─────────────────────────────────────────
     control-center:
       image: confluentinc/cp-enterprise-control-center:8.0.0
       hostname: control-center
       container_name: control-center
       networks:
         - kafka-net
       ports:
         - "9021:9021"
       depends_on:
         - kafka1
         - kafka2
         - kafka3
         - kafka-connect
       environment:
         CONTROL_CENTER_BOOTSTRAP_SERVERS: 'kafka1:29092,kafka2:29092,kafka3:29092'
         CONTROL_CENTER_CONNECT_CONNECT-LAB03_CLUSTER: 'http://kafka-connect:8083'
         CONTROL_CENTER_REPLICATION_FACTOR: 3
         CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
         CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
         CONFLUENT_METRICS_TOPIC_REPLICATION: 3
         PORT: 9021
   EOF
   ```

2. Verifica que el archivo fue creado correctamente:

   ```bash
   # Verificar que el archivo tiene contenido válido
   cat docker-compose.yml | grep "container_name:" 
   ```

3. Levanta el clúster completo:

   ```bash
   cd ~/kafka-lab03
   docker compose up -d
   ```

4. Monitorea el inicio de los contenedores (espera aproximadamente 60-90 segundos):

   ```bash
   # Ver el estado de todos los contenedores cada 5 segundos
   watch -n 5 "docker compose ps --format 'table {{.Name}}\t{{.Status}}\t{{.Ports}}'"
   ```

   Presiona `Ctrl+C` cuando todos los contenedores muestren estado `Up` o `healthy`.

**Salida Esperada:**

```
NAME              STATUS          PORTS
control-center    Up              0.0.0.0:9021->9021/tcp
kafka-connect     Up              0.0.0.0:8083->8083/tcp
kafka1            Up              0.0.0.0:9092->9092/tcp, 0.0.0.0:9101->9101/tcp
kafka2            Up              0.0.0.0:9093->9093/tcp, 0.0.0.0:9102->9102/tcp
kafka3            Up              0.0.0.0:9094->9094/tcp, 0.0.0.0:9103->9103/tcp
```

**Verificación:**

```bash
# Verificar que los 3 brokers están activos y se comunican entre sí
docker exec kafka1 kafka-broker-api-versions \
  --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
  2>/dev/null | grep "id:" | head -5

# Verificar el estado del quórum KRaft
docker exec kafka1 kafka-metadata-quorum \
  --bootstrap-server kafka1:29092 \
  describe --status
```

La salida del quórum debe mostrar `LeaderId` con un valor entre 1 y 3, y `CurrentVoters` con los 3 nodos.

---

### Paso 2: Crear Tópicos con Factor de Replicación y Analizar la Distribución de Líderes

**Objetivo:** Crear tópicos con configuraciones específicas de particiones y replicación, y comprender cómo Kafka distribuye los líderes entre los brokers.

**Instrucciones:**

1. Crea el tópico `pedidos` con 6 particiones y factor de replicación 3:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --create \
     --topic pedidos \
     --partitions 6 \
     --replication-factor 3 \
     --config min.insync.replicas=2 \
     --config unclean.leader.election.enable=false
   ```

2. Crea el tópico `inventario` con 3 particiones y factor de replicación 3:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --create \
     --topic inventario \
     --partitions 3 \
     --replication-factor 3 \
     --config min.insync.replicas=2
   ```

3. Crea el tópico `logs-aplicacion` con 12 particiones (alta concurrencia) y factor de replicación 2:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --create \
     --topic logs-aplicacion \
     --partitions 12 \
     --replication-factor 2 \
     --config min.insync.replicas=1 \
     --config retention.ms=86400000
   ```

4. Describe el tópico `pedidos` para analizar la distribución de líderes y réplicas:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe \
     --topic pedidos
   ```

5. Describe todos los tópicos para tener una visión completa:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe
   ```

6. Crea un script para visualizar de forma resumida la distribución de líderes:

   ```bash
   cat > ~/kafka-lab03/analizar-lideres.sh << 'EOF'
   #!/bin/bash
   echo "=== Distribución de Líderes por Broker ==="
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe 2>/dev/null | \
     grep "Leader:" | \
     awk '{print $6}' | \
     sort | uniq -c | \
     awk '{print "Broker " $2 ": " $1 " particiones como líder"}'
   
   echo ""
   echo "=== Estado del ISR por Tópico ==="
   for topic in pedidos inventario logs-aplicacion; do
     echo "--- Tópico: $topic ---"
     docker exec kafka1 kafka-topics \
       --bootstrap-server kafka1:29092 \
       --describe \
       --topic $topic 2>/dev/null | grep -v "^Topic:"
   done
   EOF
   chmod +x ~/kafka-lab03/analizar-lideres.sh
   ~/kafka-lab03/analizar-lideres.sh
   ```

**Salida Esperada:**

```
Topic: pedidos  PartitionCount: 6  ReplicationFactor: 3  Configs: min.insync.replicas=2,...
  Topic: pedidos  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
  Topic: pedidos  Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3,1
  Topic: pedidos  Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1,2
  Topic: pedidos  Partition: 3  Leader: 1  Replicas: 1,3,2  Isr: 1,3,2
  Topic: pedidos  Partition: 4  Leader: 2  Replicas: 2,1,3  Isr: 2,1,3
  Topic: pedidos  Partition: 5  Leader: 3  Replicas: 3,2,1  Isr: 3,2,1
```

Observa que Kafka distribuye los líderes de forma equilibrada entre los 3 brokers (aproximadamente 2 particiones como líder por broker para el tópico `pedidos`).

**Verificación:**

```bash
# Verificar que los 3 tópicos existen
docker exec kafka1 kafka-topics \
  --bootstrap-server kafka1:29092 \
  --list | grep -E "pedidos|inventario|logs-aplicacion"

# Verificar que TODOS los ISR tienen los 3 brokers (todos sincronizados)
docker exec kafka1 kafka-topics \
  --bootstrap-server kafka1:29092 \
  --describe \
  --topic pedidos | grep "Isr:" | grep -v "1,2,3\|1,3,2\|2,1,3\|2,3,1\|3,1,2\|3,2,1"
# Si este último comando no devuelve nada, todos los ISR están completos ✓
```

---

### Paso 3: Simular Fallos de Broker y Observar la Elección de Nuevo Líder

**Objetivo:** Experimentar la tolerancia a fallos de Kafka deteniendo un broker y observando cómo el sistema elige automáticamente nuevos líderes y cómo se recupera el ISR cuando el broker vuelve.

**Instrucciones:**

1. Primero, registra el estado actual de los líderes del tópico `pedidos` ANTES del fallo:

   ```bash
   echo "=== Estado ANTES del fallo ==="
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe \
     --topic pedidos
   ```

   Guarda mentalmente (o en un archivo) qué particiones tienen su líder en `kafka2`.

2. Abre una segunda terminal y ejecuta un consumidor continuo para observar la interrupción:

   ```bash
   # En una SEGUNDA terminal — mantener abierta durante el fallo
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --from-beginning \
     --property print.key=true \
     --property print.timestamp=true
   ```

3. En una TERCERA terminal, produce algunos mensajes al tópico `pedidos`:

   ```bash
   # Producir 20 mensajes de prueba con clave
   for i in $(seq 1 20); do
     echo "order-$i:{\"status\":\"creado\",\"monto\":$((RANDOM % 1000))}" | \
     docker exec -i kafka1 kafka-console-producer \
       --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
       --topic pedidos \
       --property "parse.key=true" \
       --property "key.separator=:"
   done
   echo "✓ 20 mensajes producidos"
   ```

4. **SIMULA EL FALLO**: Detén el broker `kafka2` (sin eliminar sus datos):

   ```bash
   # En la terminal PRINCIPAL — detener kafka2
   echo "=== SIMULANDO FALLO DE kafka2 ==="
   docker stop kafka2
   echo "kafka2 detenido en: $(date)"
   ```

5. Inmediatamente observa cómo cambia el estado del tópico:

   ```bash
   # Espera 5 segundos para que el clúster detecte el fallo
   sleep 5
   
   echo "=== Estado DESPUÉS del fallo de kafka2 ==="
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe \
     --topic pedidos
   ```

6. Verifica que el clúster sigue aceptando escrituras (con 2 brokers activos y `min.insync.replicas=2`):

   ```bash
   # Producir mensajes con solo 2 brokers activos
   for i in $(seq 21 30); do
     echo "order-$i:{\"status\":\"procesando\",\"monto\":$((RANDOM % 1000))}" | \
     docker exec -i kafka1 kafka-console-producer \
       --bootstrap-server kafka1:29092,kafka3:29092 \
       --topic pedidos \
       --property "parse.key=true" \
       --property "key.separator=:"
   done
   echo "✓ Mensajes producidos con 2 brokers activos"
   ```

7. Ahora **RECUPERA** el broker `kafka2` y observa la resincronización del ISR:

   ```bash
   echo "=== RECUPERANDO kafka2 ==="
   docker start kafka2
   echo "kafka2 reiniciado en: $(date)"
   
   # Espera la resincronización (puede tomar 15-30 segundos)
   echo "Esperando resincronización del ISR..."
   sleep 20
   
   echo "=== Estado DESPUÉS de la recuperación de kafka2 ==="
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe \
     --topic pedidos
   ```

8. Analiza el script de líderes nuevamente para confirmar el rebalanceo:

   ```bash
   ~/kafka-lab03/analizar-lideres.sh
   ```

**Salida Esperada después del fallo:**

```
Topic: pedidos  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,3
Topic: pedidos  Partition: 1  Leader: 3  Replicas: 2,3,1  Isr: 3,1
Topic: pedidos  Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1
```

Observa que `kafka2` desapareció del ISR, y las particiones que tenían su líder en `kafka2` ahora tienen un nuevo líder en `kafka1` o `kafka3`.

**Salida Esperada después de la recuperación:**

```
Topic: pedidos  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,3,2
Topic: pedidos  Partition: 1  Leader: 3  Replicas: 2,3,1  Isr: 3,1,2
```

El ISR vuelve a tener los 3 brokers. Nota que el liderazgo puede no regresar automáticamente a `kafka2` (requiere un `preferred-replica-election` explícito).

**Verificación:**

```bash
# Confirmar que todos los mensajes (1-30) son accesibles después de la recuperación
docker exec kafka1 kafka-console-consumer \
  --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
  --topic pedidos \
  --from-beginning \
  --max-messages 30 \
  --property print.key=true 2>/dev/null | wc -l
# Debe mostrar 30
```

---

### Paso 4: Configurar Políticas de Retención y Compresión de Mensajes

**Objetivo:** Aplicar y validar diferentes políticas de retención de datos (por tiempo y por tamaño) usando `kafka-configs.sh`, y comparar algoritmos de compresión en términos de eficiencia.

**Instrucciones:**

1. Verifica la configuración actual del tópico `logs-aplicacion`:

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --describe \
     --entity-type topics \
     --entity-name logs-aplicacion
   ```

2. Modifica la retención del tópico `logs-aplicacion` a 2 horas (para laboratorio):

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --alter \
     --entity-type topics \
     --entity-name logs-aplicacion \
     --add-config retention.ms=7200000
   ```

3. Agrega también una retención por tamaño de 100 MB:

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --alter \
     --entity-type topics \
     --entity-name logs-aplicacion \
     --add-config retention.bytes=104857600
   ```

4. Verifica que los cambios se aplicaron:

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --describe \
     --entity-type topics \
     --entity-name logs-aplicacion
   ```

5. Crea un tópico especial con retención muy corta para demostración:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --create \
     --topic demo-retencion-corta \
     --partitions 3 \
     --replication-factor 3 \
     --config retention.ms=60000 \
     --config segment.ms=30000 \
     --config segment.bytes=1048576
   ```

6. Ahora experimenta con la **compresión**. Crea un script de producción que genere mensajes JSON de tamaño considerable:

   ```bash
   cat > ~/kafka-lab03/test-compresion.sh << 'SCRIPT'
   #!/bin/bash
   
   BROKER="kafka1:29092,kafka2:29092,kafka3:29092"
   MENSAJES=500
   
   # Generar datos de prueba (JSON simulando un evento de log)
   generar_mensaje() {
     echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"INFO\",\"service\":\"order-service\",\"trace_id\":\"$(cat /proc/sys/kernel/random/uuid 2>/dev/null || uuidgen 2>/dev/null || echo 'abc-123-def')\",\"message\":\"Procesando pedido con validación de inventario y cálculo de descuentos aplicables al cliente premium\",\"user_id\":$((RANDOM % 10000)),\"order_id\":$((RANDOM % 100000)),\"amount\":$((RANDOM % 5000))}"
   }
   
   echo "=== Test de Compresión de Mensajes ==="
   echo "Produciendo $MENSAJES mensajes con diferentes algoritmos..."
   echo ""
   
   for CODEC in none gzip snappy lz4 zstd; do
     TOPIC="test-compresion-${CODEC}"
     
     # Crear tópico
     docker exec kafka1 kafka-topics \
       --bootstrap-server $BROKER \
       --create \
       --topic $TOPIC \
       --partitions 3 \
       --replication-factor 3 \
       --if-not-exists 2>/dev/null
     
     # Producir mensajes con el codec especificado
     START=$(date +%s%3N)
     for i in $(seq 1 $MENSAJES); do
       generar_mensaje
     done | docker exec -i kafka1 kafka-console-producer \
       --bootstrap-server $BROKER \
       --topic $TOPIC \
       --compression-codec $CODEC \
       --producer-property batch.size=65536 \
       --producer-property linger.ms=10 2>/dev/null
     END=$(date +%s%3N)
     
     ELAPSED=$((END - START))
     echo "Codec: $CODEC | Tiempo de producción: ${ELAPSED}ms | Tópico: $TOPIC"
   done
   
   echo ""
   echo "=== Comparación de tamaño en disco (aproximado) ==="
   for CODEC in none gzip snappy lz4 zstd; do
     TOPIC="test-compresion-${CODEC}"
     SIZE=$(docker exec kafka1 du -sh /var/lib/kafka/data/${TOPIC}-0/ 2>/dev/null | awk '{print $1}')
     echo "Codec: $CODEC | Tamaño partición 0: ${SIZE:-N/A}"
   done
   SCRIPT
   chmod +x ~/kafka-lab03/test-compresion.sh
   ~/kafka-lab03/test-compresion.sh
   ```

7. Configura compresión a nivel de tópico para `pedidos` (compresión del lado del broker):

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --alter \
     --entity-type topics \
     --entity-name pedidos \
     --add-config compression.type=lz4
   ```

8. Verifica la configuración final del tópico `pedidos`:

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --describe \
     --entity-type topics \
     --entity-name pedidos
   ```

**Salida Esperada:**

```
Dynamic configs for topic pedidos are:
  compression.type=lz4 sensitive=false synonyms={DYNAMIC_TOPIC_CONFIG:compression.type=lz4, ...}
  min.insync.replicas=2 sensitive=false synonyms={...}
  unclean.leader.election.enable=false sensitive=false synonyms={...}
```

```
Dynamic configs for topic logs-aplicacion are:
  retention.ms=7200000 sensitive=false synonyms={...}
  retention.bytes=104857600 sensitive=false synonyms={...}
```

**Verificación:**

```bash
# Verificar todas las configuraciones dinámicas aplicadas
for topic in pedidos inventario logs-aplicacion; do
  echo "--- Configuración de $topic ---"
  docker exec kafka1 kafka-configs \
    --bootstrap-server kafka1:29092 \
    --describe \
    --entity-type topics \
    --entity-name $topic 2>/dev/null
done
```

---

### Paso 5: Configurar Productores con Parámetros de Rendimiento y Confiabilidad

**Objetivo:** Crear productores con diferentes configuraciones de `acks`, `batch.size` y `linger.ms`, y observar el impacto en throughput, latencia y comportamiento ante fallos de broker.

**Instrucciones:**

1. Crea un script Python que simule un productor configurable con métricas:

   ```bash
   cat > ~/kafka-lab03/productor-configurable.py << 'PYEOF'
   #!/usr/bin/env python3
   """
   Productor Kafka configurable para análisis de rendimiento.
   Requiere: pip3 install confluent-kafka
   """
   
   import time
   import json
   import random
   import sys
   from datetime import datetime
   
   try:
       from confluent_kafka import Producer, KafkaException
   except ImportError:
       print("ERROR: confluent-kafka no instalado.")
       print("Ejecuta: pip3 install confluent-kafka")
       sys.exit(1)
   
   def crear_productor(acks, batch_size, linger_ms, broker="localhost:9092"):
       config = {
           'bootstrap.servers': broker,
           'acks': str(acks),
           'batch.size': batch_size,
           'linger.ms': linger_ms,
           'compression.type': 'lz4',
           'retries': 3,
           'retry.backoff.ms': 100,
       }
       return Producer(config)
   
   def generar_pedido(i):
       return json.dumps({
           "order_id": f"order-{i:06d}",
           "timestamp": datetime.utcnow().isoformat(),
           "customer_id": random.randint(1000, 9999),
           "amount": round(random.uniform(10.0, 999.99), 2),
           "status": random.choice(["creado", "pagado", "enviado"]),
           "items": random.randint(1, 10)
       })
   
   def medir_produccion(nombre_config, acks, batch_size, linger_ms, num_mensajes=1000):
       print(f"\n{'='*60}")
       print(f"Configuración: {nombre_config}")
       print(f"  acks={acks}, batch.size={batch_size}, linger.ms={linger_ms}")
       print(f"  Mensajes a producir: {num_mensajes}")
       print('='*60)
       
       producer = crear_productor(acks, batch_size, linger_ms)
       mensajes_enviados = 0
       errores = 0
       
       def delivery_callback(err, msg):
           nonlocal mensajes_enviados, errores
           if err:
               errores += 1
           else:
               mensajes_enviados += 1
       
       inicio = time.time()
       
       for i in range(num_mensajes):
           key = f"order-{i % 100}"
           value = generar_pedido(i)
           try:
               producer.produce(
                   topic='pedidos',
                   key=key.encode('utf-8'),
                   value=value.encode('utf-8'),
                   callback=delivery_callback
               )
               if i % 100 == 0:
                   producer.poll(0)
           except KafkaException as e:
               errores += 1
               print(f"  Error al producir mensaje {i}: {e}")
       
       # Esperar confirmación de todos los mensajes
       producer.flush(timeout=30)
       
       fin = time.time()
       duracion = fin - inicio
       throughput = num_mensajes / duracion if duracion > 0 else 0
       
       print(f"  ✓ Mensajes enviados: {mensajes_enviados}/{num_mensajes}")
       print(f"  ✗ Errores: {errores}")
       print(f"  ⏱  Duración total: {duracion:.2f}s")
       print(f"  📊 Throughput: {throughput:.0f} msg/s")
       
       return {
           "config": nombre_config,
           "enviados": mensajes_enviados,
           "errores": errores,
           "duracion_s": round(duracion, 2),
           "throughput_msg_s": round(throughput, 0)
       }
   
   if __name__ == "__main__":
       resultados = []
       
       # Configuración 1: Alta confiabilidad (producción)
       resultados.append(medir_produccion(
           "Alta Confiabilidad (acks=all)",
           acks="all", batch_size=16384, linger_ms=5, num_mensajes=500
       ))
       
       # Configuración 2: Balance (acks=1)
       resultados.append(medir_produccion(
           "Balance (acks=1)",
           acks=1, batch_size=32768, linger_ms=10, num_mensajes=500
       ))
       
       # Configuración 3: Máximo throughput (acks=0, lotes grandes)
       resultados.append(medir_produccion(
           "Máximo Throughput (acks=0)",
           acks=0, batch_size=65536, linger_ms=20, num_mensajes=500
       ))
       
       print(f"\n{'='*60}")
       print("RESUMEN COMPARATIVO")
       print('='*60)
       print(f"{'Configuración':<35} {'Throughput':>12} {'Duración':>10}")
       print('-'*60)
       for r in resultados:
           print(f"{r['config']:<35} {r['throughput_msg_s']:>10.0f}/s {r['duracion_s']:>8.2f}s")
   PYEOF
   chmod +x ~/kafka-lab03/productor-configurable.py
   ```

2. Si tienes Python 3 con `confluent-kafka`, ejecuta el script directamente. Si no, usa el productor de consola con configuraciones manuales:

   ```bash
   # Opción A: Con Python (si está disponible)
   pip3 install confluent-kafka 2>/dev/null && python3 ~/kafka-lab03/productor-configurable.py
   
   # Opción B: Con kafka-console-producer y parámetros de configuración
   echo "=== Producción con acks=all (alta confiabilidad) ==="
   time (for i in $(seq 1 200); do
     echo "order-$i:{\"amount\":$((RANDOM % 1000)),\"status\":\"creado\"}"
   done | docker exec -i kafka1 kafka-console-producer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --property "parse.key=true" \
     --property "key.separator=:" \
     --producer-property acks=all \
     --producer-property batch.size=16384 \
     --producer-property linger.ms=5)
   
   echo ""
   echo "=== Producción con acks=1 (balance) ==="
   time (for i in $(seq 201 400); do
     echo "order-$i:{\"amount\":$((RANDOM % 1000)),\"status\":\"pagado\"}"
   done | docker exec -i kafka1 kafka-console-producer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --property "parse.key=true" \
     --property "key.separator=:" \
     --producer-property acks=1 \
     --producer-property batch.size=32768 \
     --producer-property linger.ms=10)
   ```

3. Verifica el comportamiento de `acks=all` cuando el ISR cae por debajo de `min.insync.replicas`. Detén `kafka2` y `kafka3` simultáneamente:

   ```bash
   echo "=== Test: ¿Qué pasa cuando ISR < min.insync.replicas? ==="
   echo "Deteniendo kafka2 y kafka3..."
   docker stop kafka2 kafka3
   sleep 5
   
   echo "Intentando producir con acks=all y solo 1 broker activo..."
   echo "order-999:{\"status\":\"test-fallo\"}" | \
   docker exec -i kafka1 kafka-console-producer \
     --bootstrap-server kafka1:29092 \
     --topic pedidos \
     --property "parse.key=true" \
     --property "key.separator=:" \
     --producer-property acks=all \
     --producer-property request.timeout.ms=5000 \
     --producer-property max.block.ms=5000 2>&1 | head -20
   
   echo ""
   echo "Recuperando kafka2 y kafka3..."
   docker start kafka2 kafka3
   sleep 20
   echo "Clúster recuperado."
   ```

**Salida Esperada del test de fallo:**

```
[2024-XX-XX ...] WARN Error while fetching metadata with correlation id ... 
[2024-XX-XX ...] ERROR Error when sending message to topic pedidos with key: ...
org.apache.kafka.common.errors.NotEnoughReplicasException: Messages are rejected since there are fewer in-sync replicas than required.
```

Esto confirma que `min.insync.replicas=2` con `acks=all` protege la integridad de los datos rechazando escrituras cuando el ISR es insuficiente.

**Verificación:**

```bash
# Verificar que el clúster está completamente recuperado
sleep 25
docker exec kafka1 kafka-topics \
  --bootstrap-server kafka1:29092 \
  --describe \
  --topic pedidos | grep -E "Isr:" | head -3
# Todos los ISR deben mostrar los 3 brokers nuevamente
```

---

### Paso 6: Administrar Grupos de Consumidores y Manipular Offsets

**Objetivo:** Crear grupos de consumidores, inspeccionar su estado y lag, y practicar el reset de offsets para reprocesar mensajes o saltar mensajes defectuosos.

**Instrucciones:**

1. Inicia un grupo de consumidores con 2 instancias en paralelo (abre dos terminales o usa `&`):

   ```bash
   # Consumidor 1 del grupo "grupo-procesamiento"
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --group grupo-procesamiento \
     --property print.key=true \
     --property print.partition=true \
     --property print.offset=true \
     --max-messages 50 \
     --timeout-ms 10000 2>/dev/null &
   
   CONSUMER_PID=$!
   echo "Consumidor iniciado con PID: $CONSUMER_PID"
   
   # Esperar a que el consumidor procese los mensajes
   wait $CONSUMER_PID
   echo "Consumidor terminado."
   ```

2. Inspecciona el estado del grupo de consumidores:

   ```bash
   # Listar todos los grupos de consumidores
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --list
   
   # Describir el grupo "grupo-procesamiento" con detalle de lag
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento
   ```

3. Produce más mensajes para crear lag en el grupo:

   ```bash
   echo "=== Generando lag en el grupo ==="
   for i in $(seq 1000 1200); do
     echo "order-$i:{\"amount\":$((RANDOM % 1000)),\"status\":\"nuevo\"}"
   done | docker exec -i kafka1 kafka-console-producer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --property "parse.key=true" \
     --property "key.separator=:" \
     --producer-property acks=all 2>/dev/null
   
   echo "✓ 200 mensajes adicionales producidos"
   
   # Verificar el lag acumulado
   echo ""
   echo "=== Lag actual del grupo ==="
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento
   ```

4. **Reset de offsets al inicio** (reprocesar todos los mensajes):

   ```bash
   # IMPORTANTE: El consumidor debe estar detenido para hacer reset
   echo "=== Reset de offsets al inicio (--to-earliest) ==="
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --group grupo-procesamiento \
     --topic pedidos \
     --reset-offsets \
     --to-earliest \
     --execute
   
   # Verificar que los offsets se reiniciaron
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento
   ```

5. **Reset de offsets al final** (saltar todos los mensajes pendientes):

   ```bash
   echo "=== Reset de offsets al final (--to-latest) ==="
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --group grupo-procesamiento \
     --topic pedidos \
     --reset-offsets \
     --to-latest \
     --execute
   
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento
   ```

6. **Reset de offsets a un punto específico en el tiempo**:

   ```bash
   # Reset a un timestamp específico (hace 30 minutos)
   TIMESTAMP_30MIN_AGO=$(date -d "30 minutes ago" +%Y-%m-%dT%H:%M:%S.000 2>/dev/null || \
                         date -v-30M +%Y-%m-%dT%H:%M:%S.000)
   
   echo "=== Reset de offsets por timestamp: $TIMESTAMP_30MIN_AGO ==="
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --group grupo-procesamiento \
     --topic pedidos \
     --reset-offsets \
     --to-datetime "${TIMESTAMP_30MIN_AGO}" \
     --execute
   
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento
   ```

7. **Simula el rebalanceo** iniciando dos consumidores del mismo grupo simultáneamente:

   ```bash
   cat > ~/kafka-lab03/observar-rebalanceo.sh << 'EOF'
   #!/bin/bash
   echo "=== Iniciando 2 consumidores del mismo grupo para observar rebalanceo ==="
   
   # Consumidor 1
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --group grupo-rebalanceo-test \
     --property print.partition=true \
     --property print.offset=true \
     --from-beginning \
     --max-messages 30 \
     --timeout-ms 15000 > /tmp/consumer1.log 2>&1 &
   C1_PID=$!
   
   sleep 2
   
   # Consumidor 2 (se une al mismo grupo — trigger de rebalanceo)
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos \
     --group grupo-rebalanceo-test \
     --property print.partition=true \
     --property print.offset=true \
     --from-beginning \
     --max-messages 30 \
     --timeout-ms 15000 > /tmp/consumer2.log 2>&1 &
   C2_PID=$!
   
   echo "Consumidor 1 PID: $C1_PID"
   echo "Consumidor 2 PID: $C2_PID"
   echo "Esperando que ambos consumidores terminen..."
   wait $C1_PID $C2_PID
   
   echo ""
   echo "=== Particiones asignadas al Consumidor 1 ==="
   grep "^Partition:" /tmp/consumer1.log | awk -F'[:\t]' '{print $2}' | sort -u | head -10
   
   echo ""
   echo "=== Particiones asignadas al Consumidor 2 ==="
   grep "^Partition:" /tmp/consumer2.log | awk -F'[:\t]' '{print $2}' | sort -u | head -10
   
   echo ""
   echo "=== Estado final del grupo ==="
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-rebalanceo-test
   EOF
   chmod +x ~/kafka-lab03/observar-rebalanceo.sh
   ~/kafka-lab03/observar-rebalanceo.sh
   ```

**Salida Esperada del describe del grupo:**

```
GROUP               TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID  HOST
grupo-procesamiento pedidos   0          45              45              0    -             -
grupo-procesamiento pedidos   1          38              38              0    -             -
grupo-procesamiento pedidos   2          42              42              0    -             -
grupo-procesamiento pedidos   3          41              41              0    -             -
grupo-procesamiento pedidos   4          39              39              0    -             -
grupo-procesamiento pedidos   5          44              44              0    -             -
```

**Verificación:**

```bash
# Verificar que el reset a earliest funcionó (LAG debe ser > 0 con mensajes existentes)
docker exec kafka1 kafka-consumer-groups \
  --bootstrap-server kafka1:29092 \
  --describe \
  --group grupo-procesamiento | awk '{sum += $6} END {print "LAG total: " sum}'
```

---

### Paso 7: Desplegar Kafka Connect en Modo Distribuido y Configurar un Conector FileStream

**Objetivo:** Interactuar con la API REST de Kafka Connect para verificar el estado del clúster de conectores, listar los plugins disponibles y configurar un conector FileStream Source que lea un archivo y publique su contenido en un tópico Kafka.

**Instrucciones:**

1. Verifica que Kafka Connect está operativo mediante su API REST:

   ```bash
   # Verificar estado del worker de Connect
   curl -s http://localhost:8083/ | jq '.'
   ```

2. Lista los conectores disponibles en el clúster:

   ```bash
   # Ver todos los plugins de conectores disponibles
   curl -s http://localhost:8083/connector-plugins | jq '.[].class' | head -20
   ```

3. Verifica que no hay conectores activos aún:

   ```bash
   curl -s http://localhost:8083/connectors | jq '.'
   ```

4. Crea el archivo de datos de entrada para el conector FileStream:

   ```bash
   # Crear archivo con datos de pedidos simulados
   cat > ~/kafka-lab03/data/connect-input/pedidos-externos.txt << 'DATAEOF'
   {"order_id":"EXT-001","source":"sistema-externo","amount":150.00,"status":"pendiente","timestamp":"2024-01-15T10:00:00Z"}
   {"order_id":"EXT-002","source":"sistema-externo","amount":89.99,"status":"pendiente","timestamp":"2024-01-15T10:01:00Z"}
   {"order_id":"EXT-003","source":"sistema-externo","amount":299.50,"status":"pendiente","timestamp":"2024-01-15T10:02:00Z"}
   {"order_id":"EXT-004","source":"sistema-externo","amount":45.00,"status":"pendiente","timestamp":"2024-01-15T10:03:00Z"}
   {"order_id":"EXT-005","source":"sistema-externo","amount":175.25,"status":"pendiente","timestamp":"2024-01-15T10:04:00Z"}
   DATAEOF
   
   echo "✓ Archivo de datos creado:"
   cat ~/kafka-lab03/data/connect-input/pedidos-externos.txt
   ```

5. Crea el tópico destino para el conector:

   ```bash
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --create \
     --topic pedidos-externos \
     --partitions 3 \
     --replication-factor 3 \
     --if-not-exists
   ```

6. Despliega el conector FileStream Source mediante la API REST:

   ```bash
   curl -s -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "filestream-pedidos-source",
       "config": {
         "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
         "tasks.max": "1",
         "file": "/data/input/pedidos-externos.txt",
         "topic": "pedidos-externos",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "org.apache.kafka.connect.storage.StringConverter"
       }
     }' | jq '.'
   ```

7. Verifica el estado del conector:

   ```bash
   # Estado general del conector
   curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | jq '.'
   
   # Estado de las tareas del conector
   curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | \
     jq '.tasks[].state'
   ```

8. Verifica que los mensajes llegaron al tópico `pedidos-externos`:

   ```bash
   sleep 5
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos-externos \
     --from-beginning \
     --max-messages 5 \
     --timeout-ms 10000 2>/dev/null
   ```

9. Agrega más líneas al archivo de entrada y verifica que el conector las procesa automáticamente:

   ```bash
   cat >> ~/kafka-lab03/data/connect-input/pedidos-externos.txt << 'NEWDATA'
   {"order_id":"EXT-006","source":"sistema-externo","amount":520.00,"status":"pendiente","timestamp":"2024-01-15T10:05:00Z"}
   {"order_id":"EXT-007","source":"sistema-externo","amount":33.50,"status":"pendiente","timestamp":"2024-01-15T10:06:00Z"}
   NEWDATA
   
   sleep 5
   
   # Verificar que los nuevos mensajes fueron capturados
   docker exec kafka1 kafka-console-consumer \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     --topic pedidos-externos \
     --from-beginning \
     --max-messages 7 \
     --timeout-ms 10000 2>/dev/null | wc -l
   ```

10. Explora las operaciones de gestión del conector:

    ```bash
    # Pausar el conector
    curl -s -X PUT http://localhost:8083/connectors/filestream-pedidos-source/pause
    echo "Conector pausado"
    
    # Verificar estado pausado
    curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | \
      jq '.connector.state'
    
    # Reanudar el conector
    curl -s -X PUT http://localhost:8083/connectors/filestream-pedidos-source/resume
    echo "Conector reanudado"
    
    # Obtener la configuración actual del conector
    curl -s http://localhost:8083/connectors/filestream-pedidos-source/config | jq '.'
    ```

**Salida Esperada del estado del conector:**

```json
{
  "name": "filestream-pedidos-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect:8083"
    }
  ],
  "type": "source"
}
```

**Verificación:**

```bash
# Verificar que el conector está en estado RUNNING
ESTADO=$(curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | \
  jq -r '.connector.state')
echo "Estado del conector: $ESTADO"
[ "$ESTADO" = "RUNNING" ] && echo "✓ Conector operativo" || echo "✗ Conector con problemas"

# Verificar número total de mensajes en el tópico
docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --bootstrap-server kafka1:29092 \
  --topic pedidos-externos \
  --time -1 2>/dev/null | awk -F: '{sum += $3} END {print "Total mensajes en pedidos-externos: " sum}'
```

---

### Paso 8: Monitorear el Clúster con Confluent Control Center

**Objetivo:** Utilizar Confluent Control Center para obtener una visión completa del estado del clúster, explorar métricas de tópicos, monitorear el conector Kafka Connect y verificar la salud general del sistema.

**Instrucciones:**

1. Abre Confluent Control Center en tu navegador:

   ```bash
   # Verifica que Control Center está accesible
   curl -s -o /dev/null -w "%{http_code}" http://localhost:9021/
   # Debe retornar 200
   
   echo ""
   echo "Abre tu navegador en: http://localhost:9021"
   echo "Si es la primera vez, espera 60-90 segundos para que cargue completamente."
   ```

2. Genera tráfico continuo en el clúster para que las métricas sean visibles en Control Center:

   ```bash
   cat > ~/kafka-lab03/generar-trafico.sh << 'EOF'
   #!/bin/bash
   echo "=== Generando tráfico continuo para métricas en Control Center ==="
   echo "Presiona Ctrl+C para detener"
   
   CONTADOR=0
   while true; do
     # Producir batch de mensajes en pedidos
     for i in $(seq 1 10); do
       CONTADOR=$((CONTADOR + 1))
       echo "order-$CONTADOR:{\"amount\":$((RANDOM % 1000)),\"status\":\"activo\",\"ts\":$(date +%s)}"
     done | docker exec -i kafka1 kafka-console-producer \
       --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
       --topic pedidos \
       --property "parse.key=true" \
       --property "key.separator=:" \
       --producer-property acks=all \
       --producer-property batch.size=32768 \
       --producer-property linger.ms=5 2>/dev/null
     
     # Producir en inventario
     for i in $(seq 1 5); do
       echo "prod-$((RANDOM % 100)):{\"stock\":$((RANDOM % 500)),\"warehouse\":\"WH-$((RANDOM % 5 + 1))\"}"
     done | docker exec -i kafka1 kafka-console-producer \
       --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
       --topic inventario \
       --property "parse.key=true" \
       --property "key.separator=:" 2>/dev/null
     
     echo "Batch enviado. Total mensajes producidos: $CONTADOR"
     sleep 2
   done
   EOF
   chmod +x ~/kafka-lab03/generar-trafico.sh
   
   # Ejecutar en background
   ~/kafka-lab03/generar-trafico.sh &
   TRAFICO_PID=$!
   echo "Generador de tráfico iniciado con PID: $TRAFICO_PID"
   echo "Para detenerlo: kill $TRAFICO_PID"
   ```

3. Navega por las siguientes secciones en Control Center y documenta lo que observas:

   ```bash
   # Imprime las URLs importantes de Control Center para explorar
   cat << 'URLS'
   
   === URLs de Confluent Control Center ===
   
   1. Dashboard Principal:
      http://localhost:9021/clusters
   
   2. Tópicos (lista y métricas):
      http://localhost:9021/clusters/[cluster-id]/topics
   
   3. Detalle del tópico pedidos:
      http://localhost:9021/clusters/[cluster-id]/topics/pedidos/overview
   
   4. Kafka Connect:
      http://localhost:9021/clusters/[cluster-id]/connect/connect-cluster-lab03/connectors
   
   5. Grupos de consumidores:
      http://localhost:9021/clusters/[cluster-id]/consumers
   
   NOTA: Reemplaza [cluster-id] con el ID que aparece en la URL de tu navegador.
   URLS
   ```

4. Usa la API de métricas de Control Center para obtener datos programáticamente:

   ```bash
   # Verificar los tópicos visibles en Control Center via API
   curl -s http://localhost:9021/2.0/metrics/query \
     -H "Content-Type: application/json" \
     -d '{"filter":{"resource":{"type":"kafka","id":"[cluster-id]"}}}' 2>/dev/null | \
     head -5 || echo "API de métricas requiere el cluster-id correcto desde la UI"
   
   # Alternativa: usar kafka-topics para obtener métricas básicas
   echo "=== Resumen de tópicos en el clúster ==="
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --list 2>/dev/null
   
   echo ""
   echo "=== Offsets actuales por tópico ==="
   for topic in pedidos inventario logs-aplicacion pedidos-externos; do
     TOTAL=$(docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
       --bootstrap-server kafka1:29092 \
       --topic $topic \
       --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum+0}')
     echo "  $topic: $TOTAL mensajes totales"
   done
   ```

5. Detén el generador de tráfico:

   ```bash
   kill $TRAFICO_PID 2>/dev/null || true
   echo "Generador de tráfico detenido."
   ```

6. Verifica el estado completo del clúster desde la CLI:

   ```bash
   cat > ~/kafka-lab03/estado-cluster.sh << 'EOF'
   #!/bin/bash
   echo "╔══════════════════════════════════════════════════════╗"
   echo "║        ESTADO COMPLETO DEL CLÚSTER KAFKA             ║"
   echo "╚══════════════════════════════════════════════════════╝"
   
   echo ""
   echo "▶ Brokers activos:"
   docker exec kafka1 kafka-broker-api-versions \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 \
     2>/dev/null | grep "id:" | awk '{print "  Broker " $2}' || echo "  Error al conectar"
   
   echo ""
   echo "▶ Tópicos y configuraciones:"
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe 2>/dev/null | grep "PartitionCount" | \
     awk '{print "  " $1 " | Particiones:" $4 " | RF:" $6}'
   
   echo ""
   echo "▶ Grupos de consumidores activos:"
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --list 2>/dev/null | while read group; do
       LAG=$(docker exec kafka1 kafka-consumer-groups \
         --bootstrap-server kafka1:29092 \
         --describe --group $group 2>/dev/null | \
         awk 'NR>1 {sum += $6} END {print sum+0}')
       echo "  Grupo: $group | LAG total: $LAG"
     done
   
   echo ""
   echo "▶ Conectores Kafka Connect:"
   curl -s http://localhost:8083/connectors | jq -r '.[]' 2>/dev/null | while read connector; do
     STATE=$(curl -s http://localhost:8083/connectors/$connector/status | \
       jq -r '.connector.state' 2>/dev/null)
     echo "  Conector: $connector | Estado: $STATE"
   done
   
   echo ""
   echo "▶ Quórum KRaft:"
   docker exec kafka1 kafka-metadata-quorum \
     --bootstrap-server kafka1:29092 \
     describe --status 2>/dev/null | grep -E "LeaderId|CurrentVoters|CurrentObservers"
   EOF
   chmod +x ~/kafka-lab03/estado-cluster.sh
   ~/kafka-lab03/estado-cluster.sh
   ```

**Salida Esperada:**

```
▶ Brokers activos:
  Broker 1
  Broker 2
  Broker 3

▶ Tópicos y configuraciones:
  Topic:pedidos | Particiones:6 | RF:3
  Topic:inventario | Particiones:3 | RF:3
  Topic:logs-aplicacion | Particiones:12 | RF:2
  Topic:pedidos-externos | Particiones:3 | RF:3

▶ Conectores Kafka Connect:
  Conector: filestream-pedidos-source | Estado: RUNNING

▶ Quórum KRaft:
  LeaderId:          1
  CurrentVoters:     [1, 2, 3]
```

**Verificación:**

```bash
# Verificación final del estado completo
echo "=== Verificación Final ==="

# 1. Los 3 brokers están activos
BROKERS=$(docker compose ps --format "{{.Name}}\t{{.Status}}" | grep kafka | grep -c "Up")
echo "Brokers activos: $BROKERS/3 $([ $BROKERS -eq 3 ] && echo '✓' || echo '✗')"

# 2. Control Center accesible
CC_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9021/)
echo "Control Center (HTTP $CC_STATUS): $([ "$CC_STATUS" = "200" ] && echo '✓' || echo '✗')"

# 3. Kafka Connect accesible
KC_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8083/)
echo "Kafka Connect (HTTP $KC_STATUS): $([ "$KC_STATUS" = "200" ] && echo '✓' || echo '✗')"

# 4. Conector FileStream en estado RUNNING
CONNECTOR_STATE=$(curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | \
  jq -r '.connector.state' 2>/dev/null)
echo "Conector FileStream ($CONNECTOR_STATE): $([ "$CONNECTOR_STATE" = "RUNNING" ] && echo '✓' || echo '✗')"
```

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El clúster de 3 brokers Kafka está operativo con todos los contenedores en estado `Up` y el quórum KRaft muestra los 3 votantes.
- [ ] Los tópicos `pedidos` (6 particiones, RF=3), `inventario` (3 particiones, RF=3) y `logs-aplicacion` (12 particiones, RF=2) fueron creados con las configuraciones correctas.
- [ ] La simulación de fallo de `kafka2` resultó en la elección automática de nuevos líderes, y el ISR se recuperó completamente tras el reinicio del broker.
- [ ] El tópico `logs-aplicacion` tiene `retention.ms=7200000` y `retention.bytes=104857600` configurados dinámicamente.
- [ ] Se verificó que con `acks=all` y `min.insync.replicas=2`, Kafka rechaza escrituras cuando solo hay 1 broker activo (error `NotEnoughReplicasException`).
- [ ] El reset de offsets con `--to-earliest` y `--to-latest` se ejecutó correctamente para el grupo `grupo-procesamiento`.
- [ ] El conector `filestream-pedidos-source` está en estado `RUNNING` y publicó los mensajes del archivo al tópico `pedidos-externos`.
- [ ] Confluent Control Center es accesible en `http://localhost:9021` y muestra el estado del clúster, tópicos y conectores.

### Procedimiento de Pruebas

1. **Prueba de integridad del clúster:**

   ```bash
   # Todos los brokers deben responder
   docker exec kafka1 kafka-broker-api-versions \
     --bootstrap-server kafka1:29092,kafka2:29092,kafka3:29092 2>/dev/null | \
     grep "id:" | wc -l
   # Debe retornar: 3
   ```

   **Resultado Esperado:** `3`

2. **Prueba de replicación completa:**

   ```bash
   # Todos los ISR deben tener los 3 brokers para el tópico pedidos
   docker exec kafka1 kafka-topics \
     --bootstrap-server kafka1:29092 \
     --describe \
     --topic pedidos 2>/dev/null | \
     awk '/Isr:/{print $10}' | sort -u | wc -l
   # Debe retornar 1 (todas las particiones tienen el mismo patrón de ISR completo)
   ```

   **Resultado Esperado:** Los ISR de todas las particiones deben contener los 3 brokers.

3. **Prueba de configuración de retención:**

   ```bash
   docker exec kafka1 kafka-configs \
     --bootstrap-server kafka1:29092 \
     --describe \
     --entity-type topics \
     --entity-name logs-aplicacion 2>/dev/null | \
     grep -E "retention.ms|retention.bytes"
   ```

   **Resultado Esperado:**
   ```
   retention.ms=7200000
   retention.bytes=104857600
   ```

4. **Prueba del conector FileStream:**

   ```bash
   # Verificar que hay mensajes en el tópico pedidos-externos
   MSG_COUNT=$(docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
     --bootstrap-server kafka1:29092 \
     --topic pedidos-externos \
     --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum+0}')
   echo "Mensajes en pedidos-externos: $MSG_COUNT"
   # Debe ser >= 7 (5 originales + 2 adicionales)
   ```

   **Resultado Esperado:** `Mensajes en pedidos-externos: 7` (o más)

5. **Prueba de reset de offsets:**

   ```bash
   # Verificar que el grupo puede hacer reset y consumir desde el inicio
   docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --group grupo-procesamiento \
     --topic pedidos \
     --reset-offsets \
     --to-earliest \
     --execute 2>/dev/null
   
   LAG=$(docker exec kafka1 kafka-consumer-groups \
     --bootstrap-server kafka1:29092 \
     --describe \
     --group grupo-procesamiento 2>/dev/null | \
     awk 'NR>1 {sum += $6} END {print sum+0}')
   echo "LAG después de reset a earliest: $LAG"
   # Debe ser > 0 (hay mensajes para reprocesar)
   ```

   **Resultado Esperado:** LAG mayor que 0, confirmando que el reset funcionó.

---

## Solución de Problemas

### Problema 1: Los Brokers Kafka No Se Inician o Quedan en Estado "Restarting"

**Síntomas:**
- `docker compose ps` muestra uno o más brokers en estado `Restarting` o `Exited`.
- Los logs muestran errores de `CLUSTER_ID` o de inicialización de KRaft.

**Causa:**
El `CLUSTER_ID` en el `docker-compose.yml` no coincide con el que está almacenado en los volúmenes Docker de una ejecución anterior, o los volúmenes tienen datos corruptos de un intento fallido de inicio.

**Solución:**

```bash
# Paso 1: Ver los logs del broker con problemas
docker logs kafka1 --tail 50

# Paso 2: Si el error es de CLUSTER_ID, eliminar los volúmenes y reiniciar
cd ~/kafka-lab03
docker compose down -v
# ADVERTENCIA: -v elimina TODOS los volúmenes y datos

# Paso 3: Regenerar un CLUSTER_ID válido si es necesario
docker run --rm confluentinc/cp-kafka:8.0.0 kafka-storage random-uuid
# Copia el UUID generado y reemplázalo en CLUSTER_ID del docker-compose.yml

# Paso 4: Reiniciar el clúster
docker compose up -d

# Paso 5: Verificar inicio exitoso
sleep 30
docker compose ps
```

---

### Problema 2: El ISR No Se Recupera Después de Reiniciar un Broker

**Síntomas:**
- Después de reiniciar `kafka2`, el comando `kafka-topics --describe` sigue mostrando el ISR sin el broker recuperado.
- Han pasado más de 2 minutos y el ISR permanece incompleto.

**Causa:**
El broker recuperado necesita tiempo para replicar los mensajes que se produjeron durante su ausencia. Si el volumen de datos es grande o la red está saturada, la resincronización puede tardar más de lo esperado.

**Solución:**

```bash
# Paso 1: Verificar que el broker está realmente activo
docker exec kafka2 kafka-broker-api-versions \
  --bootstrap-server kafka2:29092 2>/dev/null | head -3

# Paso 2: Verificar los logs del broker recuperado para ver el progreso de replicación
docker logs kafka2 --tail 30 | grep -E "ISR|replica|sync"

# Paso 3: Forzar la verificación del estado del quórum KRaft
docker exec kafka1 kafka-metadata-quorum \
  --bootstrap-server kafka1:29092 \
  describe --status

# Paso 4: Si el broker sigue sin unirse al ISR después de 5 minutos,
# verificar configuración de red entre contenedores
docker exec kafka1 ping -c 3 kafka2

# Paso 5: Revisar la configuración de replica.lag.time.max.ms
# Un valor muy bajo puede causar que brokers lentos sean expulsados del ISR prematuramente
docker exec kafka1 kafka-configs \
  --bootstrap-server kafka1:29092 \
  --describe \
  --entity-type brokers \
  --entity-name 1 2>/dev/null | grep replica.lag
```

---

### Problema 3: Kafka Connect Muestra el Conector en Estado "FAILED"

**Síntomas:**
- `curl http://localhost:8083/connectors/filestream-pedidos-source/status` retorna `"state": "FAILED"`.
- El tópico `pedidos-externos` no recibe mensajes.

**Causa:**
El archivo de entrada (`/data/input/pedidos-externos.txt`) no existe dentro del contenedor, o el volumen no está montado correctamente.

**Solución:**

```bash
# Paso 1: Verificar que el archivo existe dentro del contenedor
docker exec kafka-connect ls -la /data/input/

# Paso 2: Si el directorio está vacío, verificar el montaje del volumen
docker inspect kafka-connect | jq '.[0].Mounts'

# Paso 3: Crear el archivo directamente dentro del contenedor si el volumen falla
docker exec kafka-connect mkdir -p /data/input
docker cp ~/kafka-lab03/data/connect-input/pedidos-externos.txt \
  kafka-connect:/data/input/pedidos-externos.txt

# Paso 4: Ver los errores detallados del conector
curl -s http://localhost:8083/connectors/filestream-pedidos-source/status | jq '.'

# Paso 5: Eliminar el conector fallido y volver a crearlo
curl -s -X DELETE http://localhost:8083/connectors/filestream-pedidos-source
sleep 3

# Paso 6: Recrear el conector
curl -s -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "filestream-pedidos-source",
    "config": {
      "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
      "tasks.max": "1",
      "file": "/data/input/pedidos-externos.txt",
      "topic": "pedidos-externos",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter": "org.apache.kafka.connect.storage.StringConverter"
    }
  }' | jq '.'
```

---

### Problema 4: Error "NotEnoughReplicasException" en Producción Normal (Sin Fallo Intencional)

**Síntomas:**
- Los productores fallan con `NotEnoughReplicasException` sin que se haya detenido ningún broker manualmente.
- `kafka-topics --describe` muestra ISR con menos brokers de los esperados.

**Causa:**
Uno o más brokers están sobrecargados o con problemas de red, y cayeron del ISR. Con `min.insync.replicas=2` y solo 1 broker en el ISR, las escrituras con `acks=all` son rechazadas.

**Solución:**

```bash
# Paso 1: Identificar qué particiones tienen ISR incompleto
docker exec kafka1 kafka-topics \
  --bootstrap-server kafka1:29092 \
  --describe \
  --topic pedidos 2>/dev/null | \
  awk '{
    split($10, isr, ",")
    if (length(isr) < 3) print "PROBLEMA: " $0
  }'

# Paso 2: Verificar el estado de todos los brokers
docker compose ps | grep kafka

# Paso 3: Ver logs de brokers con posibles problemas
docker logs kafka2 --tail 20 2>/dev/null | grep -E "ERROR|WARN|ISR"
docker logs kafka3 --tail 20 2>/dev/null | grep -E "ERROR|WARN|ISR"

# Paso 4: Verificar uso de recursos de los contenedores
docker stats kafka1 kafka2 kafka3 --no-stream

# Paso 5: Si un broker está consumiendo demasiada memoria, reiniciarlo
docker restart kafka2

# Paso 6: Opción temporal — reducir min.insync.replicas para recuperar disponibilidad
# SOLO en emergencias. Revertir después de resolver el problema de fondo.
docker exec kafka1 kafka-configs \
  --bootstrap-server kafka1:29092 \
  --alter \
  --entity-type topics \
  --entity-name pedidos \
  --add-config min.insync.replicas=1
# RECORDAR revertir a 2 después: --add-config min.insync.replicas=2
```

---

### Problema 5: Control Center No Carga o Muestra Error de Conexión

**Síntomas:**
- El navegador muestra error de conexión en `http://localhost:9021`.
- `docker logs control-center` muestra errores de conexión a los brokers.

**Causa:**
Control Center tarda entre 60-120 segundos en inicializarse completamente, especialmente si los brokers aún están arrancando. También puede ser que los tópicos internos de Control Center no se hayan creado aún.

**Solución:**

```bash
# Paso 1: Verificar que Control Center está corriendo
docker compose ps control-center

# Paso 2: Ver los logs para identificar el error específico
docker logs control-center --tail 50 | grep -E "ERROR|WARN|Started"

# Paso 3: Esperar al menos 2 minutos si el contenedor acaba de iniciar
echo "Esperando que Control Center termine de inicializar..."
for i in $(seq 1 12); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9021/ 2>/dev/null)
  echo "Intento $i/12: HTTP $STATUS"
  [ "$STATUS" = "200" ] && echo "✓ Control Center listo!" && break
  sleep 10
done

# Paso 4: Si persiste el problema, reiniciar solo Control Center
docker compose restart control-center

# Paso 5: Verificar que los tópicos internos de Control Center existen
docker exec kafka1 kafka-topics \
  --bootstrap-server kafka1:29092 \
  --list 2>/dev/null | grep "_confluent"
```

---

## Limpieza

```bash
# ─────────────────────────────────────────────────────────────
# LIMPIEZA DEL LABORATORIO 03-00-01
# ─────────────────────────────────────────────────────────────

cd ~/kafka-lab03

# Paso 1: Detener el generador de tráfico si sigue activo
pkill -f "generar-trafico.sh" 2>/dev/null || true
pkill -f "kafka-console-consumer" 2>/dev/null || true
echo "✓ Procesos en background detenidos"

# Paso 2: Eliminar el conector FileStream antes de bajar el stack
curl -s -X DELETE http://localhost:8083/connectors/filestream-pedidos-source 2>/dev/null
echo "✓ Conector FileStream eliminado"

# Paso 3: Detener todos los contenedores SIN eliminar volúmenes
# (para preservar datos entre sesiones del laboratorio)
docker compose down
echo "✓ Contenedores detenidos (volúmenes preservados)"

# ─────────────────────────────────────────────────────────────
# LIMPIEZA COMPLETA (solo si no continuarás con más laboratorios)
# ─────────────────────────────────────────────────────────────

# Para eliminar TODO incluyendo volúmenes (datos de Kafka):
# docker compose down -v
# echo "✓ Contenedores Y volúmenes eliminados"

# Para eliminar también las imágenes Docker descargadas:
# docker rmi confluentinc/cp-kafka:8.0.0 \
#            confluentinc/cp-kafka-connect:8.0.0 \
#            confluentinc/cp-enterprise-control-center:8.0.0
# echo "✓ Imágenes Docker eliminadas"

# Para eliminar el directorio de trabajo completo:
# cd ~ && rm -rf ~/kafka-lab03
# echo "✓ Directorio de trabajo eliminado"
```

> ⚠️ **Advertencia:** Los Laboratorios 4, 5 y 6 son secuenciales y dependen de los datos y la configuración generados en este laboratorio. Usa `docker compose down` (SIN `-v`) para preservar los volúmenes de datos entre sesiones. Usa `docker compose down -v` ÚNICAMENTE si deseas empezar desde cero o has terminado completamente con el módulo.

> ⚠️ **Nota sobre recursos:** Los 5 contenedores de este laboratorio (3 brokers + Connect + Control Center) consumen aproximadamente 4-6 GB de RAM. Si tu máquina tiene memoria limitada, puedes detener Control Center cuando no lo estés usando activamente: `docker stop control-center`.

---

## Resumen

### Lo que Lograste

- **Clúster multi-broker real**: Desplegaste y configuraste un clúster Kafka de producción con 3 brokers en modo KRaft, sin ZooKeeper, usando Docker Compose con volúmenes persistentes y configuraciones de red apropiadas.
- **Replicación y tolerancia a fallos en acción**: Simulaste la caída de un broker y observaste en tiempo real cómo Kafka elige automáticamente nuevos líderes desde el ISR, y cómo el sistema se recupera cuando el broker vuelve a estar disponible.
- **Comportamiento de `min.insync.replicas`**: Verificaste empíricamente que con `acks=all` y `min.insync.replicas=2`, Kafka protege la integridad de los datos rechazando escrituras cuando el ISR es insuficiente, priorizando la consistencia sobre la disponibilidad.
- **Gestión dinámica de configuraciones**: Aplicaste políticas de retención por tiempo y por tamaño sin reiniciar brokers, usando `kafka-configs.sh` para modificar configuraciones en caliente.
- **Comparación de algoritmos de compresión**: Experimentaste con `none`, `gzip`, `snappy`, `lz4` y `zstd`, observando las diferencias en tiempo de producción y tamaño en disco.
- **Administración de grupos de consumidores**: Inspeccionaste el lag de grupos, ejecutaste resets de offsets a diferentes posiciones (`earliest`, `latest`, timestamp) y observaste el proceso de rebalanceo cuando múltiples consumidores se unen al mismo grupo.
- **Kafka Connect en modo distribuido**: Desplegaste un worker de Kafka Connect, interactuaste con su API REST y configuraste un conector FileStream Source que captura datos de un archivo y los publica automáticamente en un tópico Kafka.
- **Monitoreo con Control Center**: Utilizaste Confluent Control Center para obtener una visión completa del estado del clúster, métricas de tópicos y estado de conectores.

### Conceptos Clave Aprendidos

- **ISR (In-Sync Replicas)**: Solo las réplicas en el ISR son candidatas a ser elegidas como nuevo líder. Un ISR reducido es una señal de alerta en producción.
- **La tríada de durabilidad**: La combinación `acks=all` + `min.insync.replicas=2` + `replication.factor=3` es el estándar de facto para entornos de producción que no pueden perder datos.
- **`unclean.leader.election.enable=false`**: Esta configuración es crítica para evitar pérdida de datos. Nunca debe activarse en producción sin entender sus consecuencias.
- **Configuración dinámica vs. estática**: Muchos parámetros de Kafka pueden modificarse en caliente con `kafka-configs.sh --alter` sin necesidad de reiniciar el broker.
- **Kafka Connect API REST**: La API REST es la interfaz principal para gestionar conectores en modo distribuido. Permite crear, pausar, reanudar, eliminar y monitorear conectores programáticamente.
- **Reset de offsets como herramienta operacional**: El reset de offsets es una operación poderosa para reprocesar datos, saltar mensajes problemáticos o reiniciar el procesamiento desde un punto temporal específico.

### Próximos Pasos

- **Laboratorio 4 — Kafka Connect con JDBC**: Aplicarás los conocimientos de Kafka Connect de este laboratorio para integrar PostgreSQL con Kafka usando el conector JDBC en modalidades Source y Sink, trabajando con datos relacionales reales.
- **Exploración adicional**: Investiga el comando `kafka-reassign-partitions.sh` para redistribuir manualmente particiones entre brokers, útil cuando se agrega un nuevo broker al clúster o cuando la distribución de líderes queda desbalanceada tras múltiples fallos.
- **Monitoreo avanzado**: En el Laboratorio 7 añadirás Prometheus y Grafana al stack de este laboratorio para crear dashboards de monitoreo profesionales con métricas JMX de los brokers.

---

## Recursos Adicionales

- [Documentación oficial de Apache Kafka — Replication](https://kafka.apache.org/documentation/#replication) — Guía técnica completa sobre ISR, factor de replicación y parámetros de durabilidad.
- [Confluent Developer — Kafka Internals](https://developer.confluent.io/learn-kafka/) — Serie de artículos y videos sobre el funcionamiento interno de Kafka, incluyendo particiones, replicación y tolerancia a fallos.
- [Documentación de Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) — Referencia completa de la API REST de Kafka Connect y guías de configuración para modo distribuido.
- [Confluent Control Center Documentation](https://docs.confluent.io/platform/current/control-center/index.html) — Manual de usuario de Control Center con descripción de todas las métricas y paneles disponibles.
- [kafka-configs.sh Reference](https://kafka.apache.org/documentation/#dynamicbrokerconfigs) — Lista completa de configuraciones dinámicas aplicables a brokers y tópicos sin reinicio.
- **"Kafka: The Definitive Guide" (2ª edición)** — Shapira, Palino, Sivaram, Petty. O'Reilly Media. Capítulos 5 (Confiabilidad) y 7 (Construcción de pipelines de datos) son especialmente relevantes para este laboratorio.
