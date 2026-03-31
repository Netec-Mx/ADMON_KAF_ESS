# Laboratorio 4: Conector JDBC Source (Base de datos → Kafka)

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Kafka Connect, JDBC Source Connector, PostgreSQL 15, Schema Registry, Apache Avro |

## Descripción General

En este laboratorio configurarás un pipeline completo de ingesta de datos desde una base de datos PostgreSQL hacia Apache Kafka utilizando el conector JDBC Source de Confluent. Aprenderás a desplegar el conector mediante la API REST de Kafka Connect, explorarás los distintos modos de captura de cambios (`incrementing` y `timestamp`), y verificarás que los mensajes fluyen correctamente hacia los tópicos Kafka con esquemas Avro gestionados por Schema Registry.

Este tipo de integración es uno de los casos de uso más frecuentes en arquitecturas de datos modernas: extraer datos de sistemas transaccionales relacionales y publicarlos como eventos en tiempo real en Kafka, habilitando arquitecturas orientadas a eventos sin modificar las aplicaciones existentes.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Configurar y desplegar el conector JDBC Source de Confluent para ingestar datos desde PostgreSQL hacia tópicos Kafka mediante la API REST.
- [ ] Aplicar los modos de captura de cambios `incrementing` y `timestamp` del conector JDBC para detectar registros nuevos y modificados.
- [ ] Integrar Schema Registry con el conector JDBC para gestionar automáticamente los esquemas Avro de los mensajes producidos.
- [ ] Verificar el flujo completo de datos desde PostgreSQL hasta el tópico Kafka usando `kafka-avro-console-consumer` y Control Center.
- [ ] Diagnosticar y resolver errores comunes de conectividad y configuración del conector JDBC.

## Prerrequisitos

### Conocimientos Requeridos

- Haber completado el Laboratorio 3 con comprensión de Kafka Connect en modo distribuido.
- Conocimiento básico de bases de datos relacionales y SQL (`SELECT`, `INSERT`, `CREATE TABLE`, `UPDATE`).
- Comprensión del concepto de Schema Registry y serialización Avro.
- Familiaridad con la API REST de Kafka Connect (endpoints `/connectors`, `/connectors/{name}/status`).
- Comprensión de los tópicos internos de Kafka Connect (`connect-configs`, `connect-offsets`, `connect-status`).

### Acceso Requerido

- Docker Engine 24.0+ y Docker Compose 2.20+ instalados y operativos.
- Acceso a terminal con permisos para ejecutar comandos `docker` y `docker compose`.
- Puerto `8083` (Kafka Connect), `8081` (Schema Registry), `9021` (Control Center) y `5432` (PostgreSQL) disponibles en el sistema host.
- Al menos 6 GB de RAM disponibles para Docker en el momento de ejecutar el laboratorio.

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Mínimo 4 núcleos (recomendado para múltiples contenedores) |
| RAM | Mínimo 8 GB disponibles para Docker |
| Disco | Mínimo 5 GB libres para imágenes y volúmenes |
| Red | Conexión a internet para descarga de imágenes (primera ejecución) |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Ejecución de contenedores |
| Docker Compose | 2.20+ (plugin v2) | Orquestación del entorno |
| Confluent Platform | 8.0.0 | Kafka, Schema Registry, Kafka Connect, Control Center |
| PostgreSQL | 15 | Base de datos fuente para el conector JDBC |
| curl | 7.x+ | Interacción con la API REST de Kafka Connect |
| jq | 1.6+ | Formateo de respuestas JSON en terminal |

### Configuración Inicial

Crea el directorio de trabajo para este laboratorio y todos los archivos necesarios:

```bash
# Crear directorio del laboratorio
mkdir -p ~/kafka-labs/lab04
cd ~/kafka-labs/lab04

# Crear subdirectorios para configuraciones y scripts
mkdir -p connectors scripts init-db

# Verificar que Docker está corriendo
docker info | grep "Server Version"

# Verificar que jq está instalado
jq --version || (echo "Instalando jq..." && sudo apt-get install -y jq 2>/dev/null || brew install jq 2>/dev/null)
```

## Instrucciones Paso a Paso

### Paso 1: Crear el archivo Docker Compose del entorno

**Objetivo:** Desplegar el entorno completo con Kafka (KRaft), Schema Registry, Kafka Connect con driver JDBC PostgreSQL, PostgreSQL y Control Center usando Docker Compose.

**Instrucciones:**

1. Crea el archivo `docker-compose.yml` en el directorio del laboratorio:

   ```bash
   cat > ~/kafka-labs/lab04/docker-compose.yml << 'EOF'
   version: '3.8'

   services:

     broker:
       image: confluentinc/cp-kafka:8.0.0
       hostname: broker
       container_name: broker
       ports:
         - "9092:9092"
         - "9101:9101"
       environment:
         KAFKA_NODE_ID: 1
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
         KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092'
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
         KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
         KAFKA_JMX_PORT: 9101
         KAFKA_JMX_HOSTNAME: localhost
         KAFKA_PROCESS_ROLES: 'broker,controller'
         KAFKA_CONTROLLER_QUORUM_VOTERS: '1@broker:29093'
         KAFKA_LISTENERS: 'PLAINTEXT://broker:29092,CONTROLLER://broker:29093,PLAINTEXT_HOST://0.0.0.0:9092'
         KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
         KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
         KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
         CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
       volumes:
         - broker-data:/tmp/kraft-combined-logs
       healthcheck:
         test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
         interval: 30s
         timeout: 10s
         retries: 5

     schema-registry:
       image: confluentinc/cp-schema-registry:8.0.0
       hostname: schema-registry
       container_name: schema-registry
       depends_on:
         broker:
           condition: service_healthy
       ports:
         - "8081:8081"
       environment:
         SCHEMA_REGISTRY_HOST_NAME: schema-registry
         SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker:29092'
         SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:8081/subjects"]
         interval: 30s
         timeout: 10s
         retries: 5

     kafka-connect:
       image: confluentinc/cp-kafka-connect:8.0.0
       hostname: kafka-connect
       container_name: kafka-connect
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_healthy
       ports:
         - "8083:8083"
       environment:
         CONNECT_BOOTSTRAP_SERVERS: 'broker:29092'
         CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
         CONNECT_REST_PORT: 8083
         CONNECT_GROUP_ID: lab04-connect-group
         CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
         CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
         CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
         CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
         CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
         CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
         CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
         CONNECT_LOG4J_LOGGERS: "org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR"
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:8083/connectors"]
         interval: 30s
         timeout: 10s
         retries: 10
         start_period: 60s

     postgres:
       image: postgres:15
       hostname: postgres
       container_name: postgres
       ports:
         - "5432:5432"
       environment:
         POSTGRES_DB: tienda_db
         POSTGRES_USER: kafka_user
         POSTGRES_PASSWORD: kafka_pass
       volumes:
         - postgres-data:/var/lib/postgresql/data
         - ./init-db/init.sql:/docker-entrypoint-initdb.d/init.sql
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U kafka_user -d tienda_db"]
         interval: 10s
         timeout: 5s
         retries: 5

     control-center:
       image: confluentinc/cp-enterprise-control-center:8.0.0
       hostname: control-center
       container_name: control-center
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_healthy
         kafka-connect:
           condition: service_healthy
       ports:
         - "9021:9021"
       environment:
         CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker:29092'
         CONTROL_CENTER_CONNECT_CONNECT-DEFAULT_CLUSTER: 'http://kafka-connect:8083'
         CONTROL_CENTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
         CONTROL_CENTER_REPLICATION_FACTOR: 1
         CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
         CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
         CONFLUENT_METRICS_TOPIC_REPLICATION: 1
         PORT: 9021

   volumes:
     broker-data:
     postgres-data:

   EOF
   ```

2. Verifica que el archivo se creó correctamente:

   ```bash
   cat ~/kafka-labs/lab04/docker-compose.yml | head -20
   ```

**Salida Esperada:**

```
version: '3.8'

services:

  broker:
    image: confluentinc/cp-kafka:8.0.0
    hostname: broker
    container_name: broker
    ...
```

**Verificación:**

- El archivo `docker-compose.yml` existe en `~/kafka-labs/lab04/`.
- Contiene los servicios: `broker`, `schema-registry`, `kafka-connect`, `postgres`, `control-center`.

---

### Paso 2: Crear el script de inicialización de PostgreSQL

**Objetivo:** Preparar la base de datos PostgreSQL con una tabla de pedidos que contenga datos de ejemplo y una columna de timestamp para los modos de captura.

**Instrucciones:**

1. Crea el script SQL de inicialización de la base de datos:

   ```bash
   cat > ~/kafka-labs/lab04/init-db/init.sql << 'EOF'
   -- Crear tabla de pedidos con columnas para modos incrementing y timestamp
   CREATE TABLE IF NOT EXISTS pedidos (
       id          SERIAL PRIMARY KEY,
       cliente     VARCHAR(100) NOT NULL,
       producto    VARCHAR(200) NOT NULL,
       cantidad    INTEGER NOT NULL DEFAULT 1,
       precio      DECIMAL(10, 2) NOT NULL,
       estado      VARCHAR(50) NOT NULL DEFAULT 'pendiente',
       creado_en   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
       modificado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
   );

   -- Crear índice en la columna de timestamp para optimizar las consultas del conector
   CREATE INDEX IF NOT EXISTS idx_pedidos_modificado_en ON pedidos(modificado_en);
   CREATE INDEX IF NOT EXISTS idx_pedidos_id ON pedidos(id);

   -- Insertar datos de ejemplo iniciales
   INSERT INTO pedidos (cliente, producto, cantidad, precio, estado) VALUES
       ('Ana García',    'Laptop Dell XPS 15',      1,  1299.99, 'completado'),
       ('Carlos López',  'Monitor Samsung 27"',      2,   349.50, 'completado'),
       ('María Torres',  'Teclado Mecánico Keychron',1,   129.00, 'pendiente'),
       ('Juan Martínez', 'Mouse Logitech MX Master', 1,    89.99, 'completado'),
       ('Laura Sánchez', 'Auriculares Sony WH-1000', 1,   279.00, 'enviado'),
       ('Pedro Romero',  'Webcam Logitech C920',     1,    79.99, 'pendiente'),
       ('Isabel Díaz',   'SSD Samsung 1TB',          2,   119.00, 'completado'),
       ('Miguel Herrera','Tablet iPad Air',           1,   749.00, 'enviado'),
       ('Sofía Morales', 'Impresora HP LaserJet',     1,   299.99, 'pendiente'),
       ('Roberto Castro','Cable HDMI 4K 2m',          3,    15.99, 'completado');

   -- Crear función para actualizar automáticamente modificado_en
   CREATE OR REPLACE FUNCTION actualizar_modificado_en()
   RETURNS TRIGGER AS $$
   BEGIN
       NEW.modificado_en = CURRENT_TIMESTAMP;
       RETURN NEW;
   END;
   $$ LANGUAGE plpgsql;

   -- Crear trigger que actualiza modificado_en en cada UPDATE
   CREATE TRIGGER trigger_actualizar_modificado_en
       BEFORE UPDATE ON pedidos
       FOR EACH ROW
       EXECUTE FUNCTION actualizar_modificado_en();

   -- Confirmar la inicialización
   SELECT COUNT(*) AS total_pedidos_iniciales FROM pedidos;
   EOF
   ```

2. Verifica el contenido del script SQL:

   ```bash
   cat ~/kafka-labs/lab04/init-db/init.sql
   ```

**Salida Esperada:**

```sql
-- Crear tabla de pedidos con columnas para modos incrementing y timestamp
CREATE TABLE IF NOT EXISTS pedidos (
    id          SERIAL PRIMARY KEY,
    ...
```

**Verificación:**

- El archivo `init.sql` existe en `~/kafka-labs/lab04/init-db/`.
- Contiene la definición de la tabla `pedidos` con columnas `id`, `modificado_en` y 10 registros de ejemplo.

---

### Paso 3: Instalar el conector JDBC en Kafka Connect

**Objetivo:** Instalar el plugin del conector JDBC de Confluent en el worker de Kafka Connect para habilitar la integración con PostgreSQL.

**Instrucciones:**

1. Levanta primero solo los servicios base para instalar el plugin:

   ```bash
   cd ~/kafka-labs/lab04
   docker compose up -d broker schema-registry postgres
   ```

2. Espera a que los servicios estén saludables (aproximadamente 60 segundos):

   ```bash
   echo "Esperando a que los servicios base estén listos..."
   sleep 30
   docker compose ps
   ```

3. Levanta el servicio de Kafka Connect:

   ```bash
   docker compose up -d kafka-connect
   echo "Esperando a que Kafka Connect inicie (puede tardar 60-90 segundos)..."
   ```

4. Espera a que Kafka Connect esté completamente operativo verificando su API REST:

   ```bash
   # Esperar hasta que la API REST responda
   until curl -s http://localhost:8083/connectors > /dev/null 2>&1; do
     echo "Kafka Connect aún no está listo, esperando 10 segundos..."
     sleep 10
   done
   echo "✅ Kafka Connect está operativo"
   ```

5. Instala el conector JDBC de Confluent dentro del contenedor de Kafka Connect:

   ```bash
   docker exec kafka-connect \
     confluent-hub install confluentinc/kafka-connect-jdbc:10.7.6 \
     --no-prompt
   ```

6. Instala también el driver JDBC de PostgreSQL:

   ```bash
   docker exec kafka-connect bash -c \
     "curl -L https://jdbc.postgresql.org/download/postgresql-42.7.3.jar \
     -o /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib/postgresql-42.7.3.jar"
   ```

7. Reinicia el worker de Kafka Connect para que detecte el nuevo plugin:

   ```bash
   docker compose restart kafka-connect
   echo "Esperando a que Kafka Connect se reinicie..."
   sleep 30

   # Verificar que el worker está listo nuevamente
   until curl -s http://localhost:8083/connectors > /dev/null 2>&1; do
     echo "Esperando reinicio de Kafka Connect..."
     sleep 10
   done
   echo "✅ Kafka Connect reiniciado correctamente"
   ```

8. Verifica que el plugin JDBC está disponible:

   ```bash
   curl -s http://localhost:8083/connector-plugins | jq '.[].class' | grep -i jdbc
   ```

**Salida Esperada:**

```
"io.confluent.connect.jdbc.JdbcSinkConnector"
"io.confluent.connect.jdbc.JdbcSourceConnector"
```

**Verificación:**

- La respuesta incluye `JdbcSourceConnector` y `JdbcSinkConnector`.
- El worker de Kafka Connect responde en `http://localhost:8083`.

---

### Paso 4: Verificar la conectividad con PostgreSQL

**Objetivo:** Confirmar que la base de datos PostgreSQL está correctamente inicializada y accesible antes de configurar el conector.

**Instrucciones:**

1. Verifica que PostgreSQL está corriendo y la base de datos fue inicializada:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db -c "SELECT COUNT(*) AS total FROM pedidos;"
   ```

2. Consulta los primeros registros para confirmar la estructura de la tabla:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db \
     -c "SELECT id, cliente, producto, precio, estado, creado_en FROM pedidos ORDER BY id LIMIT 5;"
   ```

3. Verifica la estructura completa de la tabla:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db \
     -c "\d pedidos"
   ```

4. Prueba la conectividad desde el contenedor de Kafka Connect hacia PostgreSQL:

   ```bash
   docker exec kafka-connect bash -c \
     "curl -s telnet://postgres:5432 || nc -zv postgres 5432 2>&1 || echo 'Conectividad verificada mediante resolución DNS'"
   
   # Alternativa más confiable
   docker exec kafka-connect bash -c \
     "getent hosts postgres && echo 'DNS resolution OK'"
   ```

**Salida Esperada:**

```
 total 
-------
    10
(1 row)

 id |    cliente     |         producto          |  precio  |   estado   |        creado_en        
----+----------------+---------------------------+----------+------------+-------------------------
  1 | Ana García     | Laptop Dell XPS 15        | 1299.99  | completado | 2024-01-15 10:30:00.123
  2 | Carlos López   | Monitor Samsung 27"       |  349.50  | completado | 2024-01-15 10:30:00.123
...
```

**Verificación:**

- La consulta retorna 10 registros en la tabla `pedidos`.
- La tabla tiene las columnas `id` (SERIAL), `modificado_en` (TIMESTAMP) visibles en `\d pedidos`.

---

### Paso 5: Configurar el conector JDBC Source en modo `incrementing`

**Objetivo:** Crear el conector JDBC Source usando el modo `incrementing` basado en la columna `id`, que detecta únicamente registros nuevos mediante un ID autoincremental.

**Instrucciones:**

1. Crea el conector JDBC Source en modo `incrementing` mediante la API REST:

   ```bash
   curl -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "jdbc-source-pedidos-incrementing",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "tasks.max": "1",
         "connection.url": "jdbc:postgresql://postgres:5432/tienda_db",
         "connection.user": "kafka_user",
         "connection.password": "kafka_pass",
         "table.whitelist": "pedidos",
         "mode": "incrementing",
         "incrementing.column.name": "id",
         "poll.interval.ms": "5000",
         "topic.prefix": "jdbc.",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "transforms": "createKey,extractInt",
         "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
         "transforms.createKey.fields": "id",
         "transforms.extractInt.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
         "transforms.extractInt.field": "id"
       }
     }' | jq .
   ```

2. Espera 10 segundos y verifica el estado del conector:

   ```bash
   sleep 10
   curl -s http://localhost:8083/connectors/jdbc-source-pedidos-incrementing/status | jq .
   ```

3. Lista todos los conectores registrados:

   ```bash
   curl -s http://localhost:8083/connectors | jq .
   ```

**Salida Esperada:**

```json
{
  "name": "jdbc-source-pedidos-incrementing",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "mode": "incrementing",
    ...
  },
  "tasks": [{"connector": "jdbc-source-pedidos-incrementing", "task": 0}],
  "type": "source"
}
```

```json
{
  "name": "jdbc-source-pedidos-incrementing",
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

- El estado del conector y de la tarea 0 muestran `"state": "RUNNING"`.
- No aparece ningún campo `"trace"` con errores en la respuesta.

---

### Paso 6: Verificar los mensajes en el tópico Kafka

**Objetivo:** Confirmar que los 10 registros de la tabla `pedidos` fueron publicados correctamente en el tópico `jdbc.pedidos` en formato Avro.

**Instrucciones:**

1. Verifica que el tópico `jdbc.pedidos` fue creado automáticamente:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list | grep jdbc
   ```

2. Consume los mensajes del tópico usando `kafka-avro-console-consumer`:

   ```bash
   docker exec schema-registry \
     kafka-avro-console-consumer \
     --bootstrap-server broker:29092 \
     --topic jdbc.pedidos \
     --from-beginning \
     --max-messages 10 \
     --property schema.registry.url=http://schema-registry:8081 \
     --property print.key=true \
     --property key.separator=" → "
   ```

3. Consulta el esquema Avro registrado automáticamente en Schema Registry:

   ```bash
   curl -s http://localhost:8081/subjects | jq .
   ```

4. Obtén el esquema específico del tópico de pedidos:

   ```bash
   curl -s http://localhost:8081/subjects/jdbc.pedidos-value/versions/latest | jq '.schema | fromjson'
   ```

**Salida Esperada:**

```
1 → {"id":1,"cliente":"Ana García","producto":"Laptop Dell XPS 15","cantidad":1,"precio":1299.99,"estado":"completado","creado_en":1705312200123,"modificado_en":1705312200123}
2 → {"id":2,"cliente":"Carlos López","producto":"Monitor Samsung 27\"","cantidad":2,"precio":349.50,"estado":"completado","creado_en":1705312200123,"modificado_en":1705312200123}
...
Processed a total of 10 messages
```

```json
["jdbc.pedidos-value"]
```

```json
{
  "type": "record",
  "name": "pedidos",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "cliente", "type": ["null", "string"]},
    {"name": "producto", "type": ["null", "string"]},
    ...
  ]
}
```

**Verificación:**

- Se consumen exactamente 10 mensajes desde el inicio del tópico.
- El esquema `jdbc.pedidos-value` aparece registrado en Schema Registry.
- Los campos del esquema Avro corresponden a las columnas de la tabla `pedidos`.

---

### Paso 7: Insertar nuevos registros y verificar la captura incremental

**Objetivo:** Demostrar el comportamiento del modo `incrementing` insertando nuevos registros en PostgreSQL y verificando que son capturados automáticamente por el conector.

**Instrucciones:**

1. Inserta 3 nuevos pedidos en la base de datos PostgreSQL:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db -c "
   INSERT INTO pedidos (cliente, producto, cantidad, precio, estado) VALUES
     ('Elena Vásquez',  'Disco Duro Externo 2TB',    1,  89.99, 'pendiente'),
     ('Diego Moreno',   'Tarjeta Gráfica RTX 4060',  1, 399.00, 'pendiente'),
     ('Carmen Ruiz',    'Hub USB-C 7 puertos',        2,  45.50, 'completado');
   SELECT id, cliente, producto, estado FROM pedidos WHERE id > 10 ORDER BY id;
   "
   ```

2. Espera el intervalo de polling del conector (5 segundos) y consume los nuevos mensajes:

   ```bash
   sleep 8
   docker exec schema-registry \
     kafka-avro-console-consumer \
     --bootstrap-server broker:29092 \
     --topic jdbc.pedidos \
     --from-beginning \
     --max-messages 13 \
     --property schema.registry.url=http://schema-registry:8081 \
     --property print.key=true \
     --property key.separator=" → " 2>/dev/null | tail -5
   ```

3. Verifica el offset del conector para confirmar que procesó los nuevos registros:

   ```bash
   curl -s http://localhost:8083/connectors/jdbc-source-pedidos-incrementing/status | \
     jq '.tasks[0].state'
   ```

4. Consulta cuántos mensajes hay ahora en el tópico:

   ```bash
   docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
     --bootstrap-server localhost:9092 \
     --topic jdbc.pedidos \
     --time -1
   ```

**Salida Esperada:**

```
INSERT 3
 id |    cliente    |           producto           |   estado   
----+---------------+------------------------------+------------
 11 | Elena Vásquez | Disco Duro Externo 2TB       | pendiente
 12 | Diego Moreno  | Tarjeta Gráfica RTX 4060     | pendiente
 13 | Carmen Ruiz   | Hub USB-C 7 puertos          | completado
(3 rows)
```

```
11 → {"id":11,"cliente":"Elena Vásquez","producto":"Disco Duro Externo 2TB",...}
12 → {"id":12,"cliente":"Diego Moreno","producto":"Tarjeta Gráfica RTX 4060",...}
13 → {"id":13,"cliente":"Carmen Ruiz","producto":"Hub USB-C 7 puertos",...}
```

```
jdbc.pedidos:0:13
```

**Verificación:**

- El tópico `jdbc.pedidos` ahora tiene 13 mensajes (offset final = 13).
- Los 3 nuevos registros aparecen en el tópico con sus IDs 11, 12 y 13.
- El estado de la tarea del conector sigue siendo `"RUNNING"`.

---

### Paso 8: Configurar el conector JDBC Source en modo `timestamp`

**Objetivo:** Crear un segundo conector usando el modo `timestamp` que detecta cambios basados en la columna `modificado_en`, permitiendo capturar tanto inserciones como actualizaciones.

**Instrucciones:**

1. Crea el conector JDBC Source en modo `timestamp`:

   ```bash
   curl -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "jdbc-source-pedidos-timestamp",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "tasks.max": "1",
         "connection.url": "jdbc:postgresql://postgres:5432/tienda_db",
         "connection.user": "kafka_user",
         "connection.password": "kafka_pass",
         "table.whitelist": "pedidos",
         "mode": "timestamp",
         "timestamp.column.name": "modificado_en",
         "poll.interval.ms": "5000",
         "topic.prefix": "jdbc.ts.",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "timestamp.delay.interval.ms": "1000"
       }
     }' | jq .
   ```

2. Espera 15 segundos y verifica el estado del nuevo conector:

   ```bash
   sleep 15
   curl -s http://localhost:8083/connectors/jdbc-source-pedidos-timestamp/status | jq .
   ```

3. Verifica cuántos mensajes se capturaron inicialmente en el nuevo tópico:

   ```bash
   docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
     --bootstrap-server localhost:9092 \
     --topic jdbc.ts.pedidos \
     --time -1
   ```

4. Ahora actualiza un pedido existente para demostrar la captura de cambios:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db -c "
   UPDATE pedidos 
   SET estado = 'enviado', precio = 1249.99
   WHERE id = 1;
   SELECT id, cliente, producto, precio, estado, modificado_en 
   FROM pedidos WHERE id = 1;
   "
   ```

5. Espera el polling y verifica que la actualización aparece en el tópico:

   ```bash
   sleep 8
   docker exec schema-registry \
     kafka-avro-console-consumer \
     --bootstrap-server broker:29092 \
     --topic jdbc.ts.pedidos \
     --from-beginning \
     --max-messages 14 \
     --property schema.registry.url=http://schema-registry:8081 2>/dev/null | \
     grep -E '"id":1,' | tail -2
   ```

**Salida Esperada:**

```json
{
  "name": "jdbc-source-pedidos-timestamp",
  "connector": {"state": "RUNNING", "worker_id": "kafka-connect:8083"},
  "tasks": [{"id": 0, "state": "RUNNING", "worker_id": "kafka-connect:8083"}],
  "type": "source"
}
```

```
jdbc.ts.pedidos:0:13
```

```
UPDATE 1
 id |   cliente   |     producto       |  precio  |  estado  |      modificado_en       
----+-------------+--------------------+----------+----------+--------------------------
  1 | Ana García  | Laptop Dell XPS 15 | 1249.99  | enviado  | 2024-01-15 11:45:23.456
```

```
{"id":1,"cliente":"Ana García","producto":"Laptop Dell XPS 15","precio":1249.99,"estado":"enviado",...}
```

**Verificación:**

- El conector `jdbc-source-pedidos-timestamp` está en estado `RUNNING`.
- El tópico `jdbc.ts.pedidos` contiene 13 mensajes iniciales más el registro actualizado.
- El mensaje del pedido con `id=1` aparece dos veces: el original y el actualizado con `estado="enviado"` y `precio=1249.99`.

---

### Paso 9: Explorar los conectores en Control Center

**Objetivo:** Utilizar la interfaz web de Confluent Control Center para visualizar el estado de los conectores, los tópicos generados y el flujo de mensajes.

**Instrucciones:**

1. Levanta Control Center si aún no está corriendo:

   ```bash
   cd ~/kafka-labs/lab04
   docker compose up -d control-center
   echo "Esperando a que Control Center inicie (puede tardar 2-3 minutos)..."
   sleep 60
   ```

2. Verifica que Control Center está accesible:

   ```bash
   curl -s http://localhost:9021 | grep -o "<title>.*</title>" | head -1
   ```

3. Abre el navegador y navega a Control Center:

   ```bash
   # En Linux, abre el navegador directamente
   xdg-open http://localhost:9021 2>/dev/null || \
   echo "Abre manualmente en tu navegador: http://localhost:9021"
   ```

4. Verifica desde la CLI los tópicos creados por los conectores:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server localhost:9092 \
     --list | sort
   ```

5. Consulta el estado de ambos conectores desde la API REST:

   ```bash
   for connector in jdbc-source-pedidos-incrementing jdbc-source-pedidos-timestamp; do
     echo "=== Estado de: $connector ==="
     curl -s "http://localhost:8083/connectors/$connector/status" | \
       jq '{nombre: .name, estado_conector: .connector.state, estado_tarea: .tasks[0].state}'
     echo ""
   done
   ```

**Salida Esperada:**

```
docker-connect-configs
docker-connect-offsets
docker-connect-status
jdbc.pedidos
jdbc.ts.pedidos
_schemas
```

```json
=== Estado de: jdbc-source-pedidos-incrementing ===
{
  "nombre": "jdbc-source-pedidos-incrementing",
  "estado_conector": "RUNNING",
  "estado_tarea": "RUNNING"
}

=== Estado de: jdbc-source-pedidos-timestamp ===
{
  "nombre": "jdbc-source-pedidos-timestamp",
  "estado_conector": "RUNNING",
  "estado_tarea": "RUNNING"
}
```

**Verificación:**

- Ambos conectores aparecen en estado `RUNNING`.
- Los tópicos `jdbc.pedidos` y `jdbc.ts.pedidos` aparecen en la lista.
- En Control Center (http://localhost:9021) se pueden ver los conectores bajo la sección "Connect".

---

### Paso 10: Comparar modos `incrementing` vs `timestamp` con una prueba práctica

**Objetivo:** Demostrar empíricamente la diferencia entre los dos modos: el modo `incrementing` no captura actualizaciones, mientras que el modo `timestamp` sí lo hace.

**Instrucciones:**

1. Registra el número actual de mensajes en ambos tópicos:

   ```bash
   echo "=== Mensajes actuales en tópicos ==="
   for topic in jdbc.pedidos jdbc.ts.pedidos; do
     count=$(docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
       --bootstrap-server localhost:9092 \
       --topic $topic \
       --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum}')
     echo "  $topic: $count mensajes"
   done
   ```

2. Actualiza varios registros en PostgreSQL:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db -c "
   UPDATE pedidos SET estado = 'entregado' WHERE id IN (2, 3, 5);
   UPDATE pedidos SET precio = precio * 0.90, estado = 'oferta' WHERE id = 7;
   SELECT id, cliente, estado, precio FROM pedidos WHERE id IN (2, 3, 5, 7) ORDER BY id;
   "
   ```

3. Espera el intervalo de polling y verifica los nuevos mensajes en cada tópico:

   ```bash
   sleep 10
   echo ""
   echo "=== Mensajes después de las actualizaciones ==="
   for topic in jdbc.pedidos jdbc.ts.pedidos; do
     count=$(docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
       --bootstrap-server localhost:9092 \
       --topic $topic \
       --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum}')
     echo "  $topic: $count mensajes"
   done
   ```

4. Inserta un nuevo registro para confirmar que el modo `incrementing` sí captura inserciones:

   ```bash
   docker exec postgres psql -U kafka_user -d tienda_db -c "
   INSERT INTO pedidos (cliente, producto, cantidad, precio, estado)
   VALUES ('Nuevo Cliente', 'Producto Test', 1, 99.99, 'pendiente');
   SELECT id, cliente FROM pedidos ORDER BY id DESC LIMIT 1;
   "
   sleep 8
   echo ""
   echo "=== Mensajes después de inserción ==="
   for topic in jdbc.pedidos jdbc.ts.pedidos; do
     count=$(docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
       --bootstrap-server localhost:9092 \
       --topic $topic \
       --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum}')
     echo "  $topic: $count mensajes"
   done
   ```

**Salida Esperada:**

```
=== Mensajes actuales en tópicos ===
  jdbc.pedidos: 13 mensajes
  jdbc.ts.pedidos: 14 mensajes
```

```
UPDATE 3
UPDATE 1
 id |    cliente   |  estado   |  precio 
----+--------------+-----------+---------
  2 | Carlos López | entregado |  349.50
  3 | María Torres | entregado |  129.00
  5 | Laura Sánchez| entregado |  279.00
  7 | Isabel Díaz  | oferta    |  107.10
```

```
=== Mensajes después de las actualizaciones ===
  jdbc.pedidos: 13 mensajes      ← Sin cambio (no captura updates)
  jdbc.ts.pedidos: 18 mensajes   ← +4 mensajes (capturó los 4 updates)
```

```
=== Mensajes después de inserción ===
  jdbc.pedidos: 14 mensajes      ← +1 (capturó la inserción)
  jdbc.ts.pedidos: 19 mensajes   ← +1 (también capturó la inserción)
```

**Verificación:**

- El tópico `jdbc.pedidos` (modo `incrementing`) **no** aumentó al actualizar registros, pero sí al insertar uno nuevo.
- El tópico `jdbc.ts.pedidos` (modo `timestamp`) **sí** capturó las 4 actualizaciones y la nueva inserción.
- Esta diferencia demuestra el comportamiento fundamental de cada modo.

## Validación y Pruebas

### Criterios de Éxito

- [ ] El entorno Docker Compose levanta correctamente los 5 servicios: `broker`, `schema-registry`, `kafka-connect`, `postgres`, `control-center`.
- [ ] El plugin `JdbcSourceConnector` aparece en la lista de plugins disponibles en `http://localhost:8083/connector-plugins`.
- [ ] El conector `jdbc-source-pedidos-incrementing` está en estado `RUNNING` y capturó los 10 registros iniciales en el tópico `jdbc.pedidos`.
- [ ] El conector `jdbc-source-pedidos-timestamp` está en estado `RUNNING` y el tópico `jdbc.ts.pedidos` contiene mensajes.
- [ ] Los mensajes en el tópico `jdbc.pedidos` son deserializables con `kafka-avro-console-consumer` usando Schema Registry.
- [ ] El esquema `jdbc.pedidos-value` aparece registrado en Schema Registry (`http://localhost:8081/subjects`).
- [ ] Nuevas inserciones en PostgreSQL aparecen en ambos tópicos después del intervalo de polling.
- [ ] Actualizaciones en PostgreSQL aparecen **solo** en el tópico del modo `timestamp`, no en el de modo `incrementing`.

### Procedimiento de Pruebas

1. Prueba de conectividad de la API REST de Kafka Connect:

   ```bash
   curl -s http://localhost:8083/ | jq '{version, commit}'
   ```
   **Resultado Esperado:** Respuesta JSON con la versión de Kafka Connect (ej: `"version": "8.0.0"`).

2. Prueba de listado de conectores activos:

   ```bash
   curl -s http://localhost:8083/connectors | jq 'length'
   ```
   **Resultado Esperado:** `2` (dos conectores registrados).

3. Prueba de Schema Registry con el esquema del tópico:

   ```bash
   curl -s http://localhost:8081/subjects/jdbc.pedidos-value/versions/1 | jq '.id'
   ```
   **Resultado Esperado:** Un número entero positivo (ID del esquema, ej: `1`).

4. Prueba de consumo de mensajes Avro:

   ```bash
   docker exec schema-registry \
     kafka-avro-console-consumer \
     --bootstrap-server broker:29092 \
     --topic jdbc.pedidos \
     --from-beginning \
     --max-messages 1 \
     --property schema.registry.url=http://schema-registry:8081 2>/dev/null
   ```
   **Resultado Esperado:** Un mensaje JSON con los campos de la tabla `pedidos`.

5. Prueba de captura incremental en tiempo real:

   ```bash
   # Inserta un registro de prueba
   docker exec postgres psql -U kafka_user -d tienda_db \
     -c "INSERT INTO pedidos (cliente, producto, cantidad, precio, estado) VALUES ('Test Final', 'Producto Validacion', 1, 1.00, 'test');"
   
   sleep 8
   
   # Verifica que el offset aumentó
   docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
     --bootstrap-server localhost:9092 \
     --topic jdbc.pedidos \
     --time -1
   ```
   **Resultado Esperado:** El offset del tópico `jdbc.pedidos` aumentó en 1 respecto al valor anterior.

## Solución de Problemas

### Problema 1: El conector queda en estado FAILED con error de conectividad JDBC

**Síntomas:**

- `curl -s http://localhost:8083/connectors/jdbc-source-pedidos-incrementing/status` muestra `"state": "FAILED"`.
- El campo `"trace"` contiene mensajes como `Connection refused` o `FATAL: password authentication failed`.

**Causa:**

El conector no puede establecer conexión con PostgreSQL. Las causas más comunes son: credenciales incorrectas, el hostname `postgres` no resuelve desde el contenedor de Kafka Connect, o el servicio de PostgreSQL no está completamente iniciado.

**Solución:**

```bash
# 1. Verificar que PostgreSQL está corriendo y saludable
docker compose ps postgres

# 2. Verificar que las credenciales son correctas conectándose directamente
docker exec postgres psql -U kafka_user -d tienda_db -c "SELECT 1 AS test;"

# 3. Verificar resolución DNS desde el contenedor de Kafka Connect
docker exec kafka-connect bash -c "getent hosts postgres"

# 4. Verificar los logs del conector para el error exacto
docker logs kafka-connect 2>&1 | grep -i "error\|exception\|failed" | tail -20

# 5. Si las credenciales son correctas pero falla la conexión, reiniciar la tarea
curl -X POST \
  http://localhost:8083/connectors/jdbc-source-pedidos-incrementing/tasks/0/restart

# 6. Si persiste el error, eliminar y recrear el conector
curl -X DELETE http://localhost:8083/connectors/jdbc-source-pedidos-incrementing
sleep 5
# Volver a ejecutar el comando POST del Paso 5
```

---

### Problema 2: El plugin JdbcSourceConnector no aparece después de la instalación

**Síntomas:**

- `curl -s http://localhost:8083/connector-plugins | jq '.[].class' | grep -i jdbc` no retorna resultados.
- Al intentar crear el conector, aparece el error `"Connector class not found"`.

**Causa:**

El worker de Kafka Connect no detectó el plugin porque no se reinició después de la instalación, o el JAR del driver PostgreSQL no está en el directorio correcto del plugin.

**Solución:**

```bash
# 1. Verificar que el plugin está instalado en el directorio correcto
docker exec kafka-connect ls /usr/share/confluent-hub-components/ | grep jdbc

# 2. Verificar que el driver PostgreSQL está presente
docker exec kafka-connect ls /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib/ | grep postgresql

# 3. Si el driver no está, descargarlo nuevamente
docker exec kafka-connect bash -c \
  "curl -L https://jdbc.postgresql.org/download/postgresql-42.7.3.jar \
  -o /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib/postgresql-42.7.3.jar"

# 4. Forzar reinicio del worker de Kafka Connect
docker compose restart kafka-connect

# 5. Esperar a que el worker esté listo
sleep 45
until curl -s http://localhost:8083/connectors > /dev/null 2>&1; do
  echo "Esperando..."
  sleep 5
done

# 6. Verificar nuevamente los plugins disponibles
curl -s http://localhost:8083/connector-plugins | jq '.[].class' | grep -i jdbc
```

---

### Problema 3: Los mensajes Avro no se pueden consumir con kafka-avro-console-consumer

**Síntomas:**

- `kafka-avro-console-consumer` lanza error `SerializationException` o `Schema not found`.
- Los mensajes se muestran como caracteres ilegibles o binarios.

**Causa:**

El consumidor no puede conectarse a Schema Registry, o el esquema fue registrado con una URL diferente a la que usa el consumidor. También puede ocurrir si el `value.converter` del conector no apunta correctamente a Schema Registry.

**Solución:**

```bash
# 1. Verificar que Schema Registry está accesible
curl -s http://localhost:8081/subjects | jq .

# 2. Verificar que el esquema del tópico está registrado
curl -s http://localhost:8081/subjects/jdbc.pedidos-value/versions | jq .

# 3. Si el esquema no existe, verificar la configuración del conector
curl -s http://localhost:8083/connectors/jdbc-source-pedidos-incrementing/config | \
  jq '{"value_converter": .["value.converter"], "schema_registry_url": .["value.converter.schema.registry.url"]}'

# 4. Verificar que Schema Registry está saludable
docker compose ps schema-registry
docker logs schema-registry 2>&1 | tail -10

# 5. Intentar consumir con la URL correcta de Schema Registry
docker exec schema-registry \
  kafka-avro-console-consumer \
  --bootstrap-server broker:29092 \
  --topic jdbc.pedidos \
  --from-beginning \
  --max-messages 3 \
  --property schema.registry.url=http://schema-registry:8081 \
  --property auto.offset.reset=earliest 2>&1 | head -20
```

---

### Problema 4: El modo `timestamp` no captura actualizaciones recientes

**Síntomas:**

- Se actualizan registros en PostgreSQL pero no aparecen nuevos mensajes en el tópico `jdbc.ts.pedidos`.
- El offset del tópico no cambia después de ejecutar `UPDATE`.

**Causa:**

El trigger `trigger_actualizar_modificado_en` no se creó correctamente, por lo que la columna `modificado_en` no se actualiza automáticamente. También puede ser que el `timestamp.delay.interval.ms` sea demasiado alto o que el conector no se haya reiniciado después de los cambios.

**Solución:**

```bash
# 1. Verificar que el trigger existe en PostgreSQL
docker exec postgres psql -U kafka_user -d tienda_db \
  -c "SELECT trigger_name, event_manipulation, action_timing FROM information_schema.triggers WHERE trigger_name = 'trigger_actualizar_modificado_en';"

# 2. Si el trigger no existe, crearlo manualmente
docker exec postgres psql -U kafka_user -d tienda_db -c "
CREATE OR REPLACE FUNCTION actualizar_modificado_en()
RETURNS TRIGGER AS \$\$
BEGIN
    NEW.modificado_en = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
\$\$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trigger_actualizar_modificado_en ON pedidos;
CREATE TRIGGER trigger_actualizar_modificado_en
    BEFORE UPDATE ON pedidos
    FOR EACH ROW
    EXECUTE FUNCTION actualizar_modificado_en();
"

# 3. Hacer un UPDATE de prueba y verificar que modificado_en cambia
docker exec postgres psql -U kafka_user -d tienda_db -c "
UPDATE pedidos SET estado = 'test_trigger' WHERE id = 1;
SELECT id, estado, modificado_en FROM pedidos WHERE id = 1;
"

# 4. Esperar el polling y verificar el tópico
sleep 10
docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
  --bootstrap-server localhost:9092 \
  --topic jdbc.ts.pedidos \
  --time -1
```

## Limpieza

```bash
# Navegar al directorio del laboratorio
cd ~/kafka-labs/lab04

# Detener y eliminar todos los contenedores
# OPCIÓN 1: Preservar los volúmenes (para continuar en el siguiente laboratorio)
docker compose down

# OPCIÓN 2: Eliminar también los volúmenes (limpieza completa)
# ⚠️ ADVERTENCIA: Esto eliminará todos los datos de Kafka y PostgreSQL
# docker compose down -v

# Verificar que los contenedores fueron eliminados
docker ps | grep -E "broker|schema-registry|kafka-connect|postgres|control-center"

# Limpiar imágenes no utilizadas (opcional, libera espacio en disco)
# docker image prune -f

# Verificar el espacio liberado
docker system df
```

> ⚠️ **Advertencia:** Usa `docker compose down` (sin `-v`) si planeas continuar con el Laboratorio 5, que depende de los datos generados en este laboratorio. Usa `docker compose down -v` **solo** si deseas hacer una limpieza completa y empezar desde cero. Los volúmenes `broker-data` y `postgres-data` contienen todos los mensajes Kafka y los datos de PostgreSQL respectivamente.

> ⚠️ **Nota sobre Confluent Platform:** Los componentes de Confluent Platform (Control Center, JDBC Connector) operan en modo de prueba por 30 días. En entornos de producción reales, se debe evaluar la licencia comercial o la versión Community Edition.

## Resumen

### Lo que Lograste

- Desplegaste un entorno completo de Kafka Connect con integración PostgreSQL usando Docker Compose, incluyendo Kafka en modo KRaft, Schema Registry, Kafka Connect con plugin JDBC y Control Center.
- Instalaste el plugin del conector JDBC de Confluent y el driver JDBC de PostgreSQL dentro del worker de Kafka Connect mediante `confluent-hub install`.
- Configuraste y desplegaste el conector JDBC Source en **modo `incrementing`** usando la API REST, capturando 10 registros iniciales y detectando nuevas inserciones basadas en la columna `id`.
- Configuraste y desplegaste el conector JDBC Source en **modo `timestamp`** que detecta tanto inserciones como actualizaciones usando la columna `modificado_en` con un trigger de PostgreSQL.
- Verificaste el flujo de datos end-to-end consumiendo mensajes en formato Avro con `kafka-avro-console-consumer` y consultando los esquemas registrados en Schema Registry.
- Demostraste empíricamente la diferencia fundamental entre los modos `incrementing` y `timestamp`: el primero no captura actualizaciones, el segundo sí.

### Conceptos Clave

- **Modo `incrementing`**: Ideal para tablas con IDs autoincrementales donde solo se insertan registros nuevos. Simple y eficiente, pero no detecta cambios en registros existentes.
- **Modo `timestamp`**: Captura inserciones y actualizaciones usando una columna de fecha de modificación. Requiere que la aplicación (o un trigger) actualice esta columna en cada cambio.
- **Modo `timestamp+incrementing`**: Combina ambas columnas para mayor robustez, evitando duplicados cuando dos registros tienen el mismo timestamp.
- **Schema Registry + Avro**: El conector JDBC genera automáticamente el esquema Avro a partir de la estructura de la tabla y lo registra en Schema Registry, habilitando la evolución controlada del esquema.
- **Dead Letter Queue (DLQ)**: Para entornos de producción, siempre configura `errors.tolerance: all` y un tópico DLQ para evitar que errores de procesamiento detengan el pipeline.
- **API REST de Kafka Connect**: Es la interfaz estándar para administrar conectores de forma programática, compatible con herramientas de CI/CD y automatización de infraestructura.

### Próximos Pasos

- Explorar el **Laboratorio 5** donde configurarás un conector JDBC Sink para escribir datos desde Kafka de regreso a PostgreSQL, completando el ciclo bidireccional de datos.
- Investigar el conector **Debezium PostgreSQL** como alternativa al JDBC Source para implementar Change Data Capture (CDC) completo con soporte para operaciones DELETE.
- Revisar la documentación de **Confluent Hub** (https://www.confluent.io/hub) para explorar conectores adicionales: S3, Elasticsearch, BigQuery, Salesforce y más de 200 integraciones disponibles.

## Recursos Adicionales

- [Documentación oficial del conector JDBC de Confluent](https://docs.confluent.io/kafka-connectors/jdbc/current/source-connector/overview.html) - Referencia completa de todos los parámetros de configuración del JDBC Source Connector, incluyendo modos, transformaciones y manejo de errores.
- [Confluent Hub - JDBC Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) - Página oficial del plugin con notas de versión, compatibilidad y ejemplos de configuración.
- [Documentación de Apache Kafka Connect](https://kafka.apache.org/documentation/#connect) - Referencia completa de la API REST, configuración de workers y modelo de ejecución de tareas.
- [Debezium PostgreSQL Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html) - Alternativa CDC completa que captura también operaciones DELETE mediante replicación lógica de PostgreSQL.
- [Schema Registry API Reference](https://docs.confluent.io/platform/current/schema-registry/develop/api.html) - Documentación completa de la API REST de Schema Registry para gestión de esquemas Avro, JSON Schema y Protobuf.
- [Single Message Transforms (SMT)](https://kafka.apache.org/documentation/#connect_transforms) - Guía de transformaciones disponibles para modificar mensajes en tiempo real dentro del pipeline de Kafka Connect, como `ValueToKey`, `ExtractField` y `ReplaceField`.

---

# Laboratorio 5: Conector JDBC Sink (Kafka → Base de datos)

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Kafka Connect, JDBC Sink Connector, PostgreSQL 15, Schema Registry, Avro, DLQ |

## Descripción General

En este laboratorio completarás el ciclo de integración bidireccional entre bases de datos y Apache Kafka configurando el conector JDBC Sink, que lee mensajes de un tópico Kafka y los escribe automáticamente en una tabla PostgreSQL de destino. Partiendo del pipeline construido en el Laboratorio 4 (JDBC Source), añadirás la segunda mitad del flujo de datos para lograr una arquitectura completa de replicación: Base de datos fuente → Kafka → Base de datos destino.

Este patrón es ampliamente utilizado en producción para sincronización de bases de datos, construcción de data lakes, replicación entre entornos (producción → staging) y desacoplamiento de microservicios. Al finalizar, habrás construido un pipeline end-to-end completamente funcional y comprenderás cómo gestionar actualizaciones sin duplicados usando el modo `upsert`.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Configurar y desplegar el conector JDBC Sink de Confluent para escribir mensajes de un tópico Kafka hacia una tabla en PostgreSQL.
- [ ] Comprender y aplicar los modos de escritura del conector JDBC Sink: `insert`, `upsert` y `update`, seleccionando el adecuado según el caso de uso.
- [ ] Gestionar la compatibilidad de esquemas entre mensajes Kafka en formato Avro y la estructura de la tabla destino en PostgreSQL.
- [ ] Construir y verificar un pipeline completo Source → Kafka → Sink comprobando la integridad de los datos en ambos extremos.
- [ ] Configurar una Dead Letter Queue (DLQ) para manejar mensajes que no pueden ser procesados sin detener el pipeline.

## Prerrequisitos

### Conocimiento Requerido

- Haber completado el Laboratorio 4 (JDBC Source) con el conector `jdbc-source-employees` activo y datos fluyendo hacia el tópico `postgres.public.employees`.
- Comprensión del funcionamiento del conector JDBC Source y Schema Registry (Lección 4.1 y Laboratorio 4).
- Conocimiento básico de SQL DDL: `CREATE TABLE`, `PRIMARY KEY`, `INSERT ON CONFLICT` (UPSERT).
- Familiaridad con la API REST de Kafka Connect (endpoints POST, GET, DELETE sobre el puerto 8083).

### Acceso Requerido

- Entorno Docker Compose del curso completamente operativo con los servicios: `broker`, `schema-registry`, `kafka-connect`, `postgres` activos.
- Acceso a terminal con permisos para ejecutar comandos `docker exec` y `curl`.
- Herramienta `jq` instalada en el sistema host para formatear respuestas JSON.

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| RAM disponible para Docker | Mínimo 6 GB (recomendado 8 GB) |
| CPU | Mínimo 2 núcleos disponibles |
| Espacio en disco | Al menos 2 GB libres adicionales |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Motor de contenedores |
| Docker Compose | 2.20+ (plugin v2) | Orquestación de servicios |
| Confluent Platform | 8.0.0 | Kafka Connect + Schema Registry |
| PostgreSQL | 15 | Base de datos destino del Sink |
| curl | 7.x+ | Interacción con la API REST |
| jq | 1.6+ | Formateo de respuestas JSON |

### Configuración Inicial

Antes de comenzar, verifica que todos los servicios necesarios estén en ejecución:

```bash
# Verificar que los contenedores estén activos
docker compose ps

# Verificar que el conector JDBC Source del Lab 4 esté RUNNING
curl -s http://localhost:8083/connectors/jdbc-source-employees/status | jq '.connector.state'

# Verificar que el tópico con datos del Lab 4 exista y tenga mensajes
docker exec broker kafka-topics --bootstrap-server broker:29092 \
  --describe --topic postgres.public.employees

# Contar mensajes disponibles en el tópico fuente
docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
  --bootstrap-server broker:29092 \
  --topic postgres.public.employees
```

Si el conector Source no está activo o el tópico no existe, regresa al Laboratorio 4 antes de continuar.

## Instrucciones Paso a Paso

### Paso 1: Preparar la Base de Datos Destino en PostgreSQL

**Objetivo:** Crear una base de datos y esquema de destino en PostgreSQL donde el conector JDBC Sink escribirá los datos replicados desde Kafka.

**Instrucciones:**

1. Conéctate al contenedor de PostgreSQL y abre una sesión `psql`:

   ```bash
   docker exec -it postgres psql -U postgres
   ```

2. Crea la base de datos de destino que simulará el sistema receptor de la replicación:

   ```sql
   -- Crear base de datos destino
   CREATE DATABASE employees_sink;

   -- Conectarse a la nueva base de datos
   \c employees_sink
   ```

3. Crea el esquema y la tabla destino. Aunque el conector puede crearla automáticamente con `auto.create=true`, crearla manualmente te permite controlar los tipos de datos y restricciones:

   ```sql
   -- Crear tabla destino con la misma estructura que la tabla fuente
   CREATE TABLE employees (
       id          SERIAL PRIMARY KEY,
       first_name  VARCHAR(100),
       last_name   VARCHAR(100),
       email       VARCHAR(150),
       department  VARCHAR(100),
       salary      NUMERIC(10,2),
       hire_date   DATE,
       updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- Verificar que la tabla fue creada correctamente
   \d employees
   ```

4. Crea un usuario dedicado para el conector Sink con permisos de escritura:

   ```sql
   -- Crear usuario para el conector Sink
   CREATE USER kafka_sink_user WITH PASSWORD 'sink_password_2024';

   -- Otorgar permisos sobre la base de datos destino
   GRANT CONNECT ON DATABASE employees_sink TO kafka_sink_user;
   GRANT USAGE ON SCHEMA public TO kafka_sink_user;
   GRANT INSERT, UPDATE, SELECT ON TABLE employees TO kafka_sink_user;
   GRANT USAGE, SELECT ON SEQUENCE employees_id_seq TO kafka_sink_user;

   -- Verificar permisos
   \dp employees
   ```

5. Sal de la sesión psql:

   ```sql
   \q
   ```

**Salida Esperada:**

```
CREATE DATABASE
You are now connected to database "employees_sink" as user "postgres".
CREATE TABLE
                      Table "public.employees"
   Column    |            Type             | Nullable |      Default
-------------+-----------------------------+----------+--------------------
 id          | integer                     | not null | nextval(...)
 first_name  | character varying(100)      |          |
 last_name   | character varying(100)      |          |
 email       | character varying(150)      |          |
 department  | character varying(100)      |          |
 salary      | numeric(10,2)               |          |
 hire_date   | date                        |          |
 updated_at  | timestamp without time zone |          | CURRENT_TIMESTAMP
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)
```

**Verificación:**

```bash
# Confirmar que la base de datos destino existe
docker exec postgres psql -U postgres -c "\l" | grep employees_sink

# Confirmar que la tabla fue creada
docker exec postgres psql -U postgres -d employees_sink -c "\dt"
```

---

### Paso 2: Verificar los Plugins Disponibles y el Esquema Avro del Tópico Fuente

**Objetivo:** Confirmar que el plugin JDBC Sink está disponible en el worker de Kafka Connect y revisar el esquema Avro de los mensajes en el tópico fuente para asegurar la compatibilidad con la tabla destino.

**Instrucciones:**

1. Verifica que el conector JDBC Sink esté disponible como plugin en el worker:

   ```bash
   curl -s http://localhost:8083/connector-plugins | jq \
     '[.[] | select(.class | contains("JdbcSink"))] | .[].class'
   ```

2. Lista todos los plugins disponibles para tener una visión completa:

   ```bash
   curl -s http://localhost:8083/connector-plugins | jq \
     '.[].class' | sort
   ```

3. Consulta el esquema Avro registrado en Schema Registry para el tópico fuente. Esto te permitirá entender la estructura exacta de los mensajes que el Sink deberá procesar:

   ```bash
   # Listar todos los subjects (esquemas) registrados
   curl -s http://localhost:8081/subjects | jq .

   # Obtener las versiones del esquema del tópico de empleados
   curl -s http://localhost:8081/subjects/postgres.public.employees-value/versions | jq .

   # Ver el esquema Avro completo de la versión más reciente
   curl -s http://localhost:8081/subjects/postgres.public.employees-value/versions/latest | jq '.schema | fromjson'
   ```

4. Verifica que hay mensajes en el tópico fuente consumiendo algunos de muestra:

   ```bash
   docker exec schema-registry kafka-avro-console-consumer \
     --bootstrap-server broker:29092 \
     --topic postgres.public.employees \
     --property schema.registry.url=http://schema-registry:8081 \
     --from-beginning \
     --max-messages 3
   ```

**Salida Esperada:**

```
# Plugins disponibles (debe incluir JdbcSink):
"io.confluent.connect.jdbc.JdbcSinkConnector"

# Esquema Avro del tópico (estructura aproximada):
{
  "type": "record",
  "name": "employees",
  "namespace": "postgres.public",
  "fields": [
    {"name": "id", "type": ["null", "int"], "default": null},
    {"name": "first_name", "type": ["null", "string"], "default": null},
    {"name": "last_name", "type": ["null", "string"], "default": null},
    {"name": "email", "type": ["null", "string"], "default": null},
    {"name": "department", "type": ["null", "string"], "default": null},
    {"name": "salary", "type": ["null", "double"], "default": null},
    {"name": "hire_date", "type": ["null", "string"], "default": null},
    {"name": "updated_at", "type": ["null", "long"], "default": null}
  ]
}
```

**Verificación:**

- El plugin `io.confluent.connect.jdbc.JdbcSinkConnector` debe aparecer en la lista.
- El esquema del tópico debe mostrar los campos que coinciden con las columnas de la tabla destino.
- Los mensajes de muestra deben mostrar registros de empleados con datos válidos.

---

### Paso 3: Desplegar el Conector JDBC Sink en Modo INSERT

**Objetivo:** Crear el conector JDBC Sink en su configuración más básica usando `insert.mode=insert` para escribir todos los mensajes del tópico como nuevas filas en la tabla destino.

**Instrucciones:**

1. Crea el conector JDBC Sink con modo `insert`. Este modo es el más simple: cada mensaje del tópico se convierte en un `INSERT` en la tabla destino:

   ```bash
   curl -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "jdbc-sink-employees-insert",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
         "tasks.max": "1",
         "topics": "postgres.public.employees",
         "connection.url": "jdbc:postgresql://postgres:5432/employees_sink",
         "connection.user": "kafka_sink_user",
         "connection.password": "sink_password_2024",
         "insert.mode": "insert",
         "auto.create": "false",
         "auto.evolve": "false",
         "table.name.format": "employees",
         "pk.mode": "none",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "transforms": "dropPrefix",
         "transforms.dropPrefix.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
         "transforms.dropPrefix.exclude": "updated_at"
       }
     }'
   ```

2. Verifica inmediatamente el estado del conector recién creado:

   ```bash
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-insert/status | jq .
   ```

3. Espera 10-15 segundos y vuelve a verificar el estado para confirmar que las tareas están en `RUNNING`:

   ```bash
   # Verificar estado del conector y sus tareas
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-insert/status | \
     jq '{connector_state: .connector.state, task_state: .tasks[0].state}'
   ```

4. Confirma que los datos están llegando a la tabla destino:

   ```bash
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT COUNT(*) as total_registros FROM employees;"

   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT id, first_name, last_name, email, department FROM employees LIMIT 5;"
   ```

**Salida Esperada:**

```json
{
  "name": "jdbc-sink-employees-insert",
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
  "type": "sink"
}
```

```
 total_registros
-----------------
              25
(1 row)

 id | first_name | last_name |         email          | department
----+------------+-----------+------------------------+------------
  1 | Ana        | García    | ana.garcia@empresa.com | Engineering
  2 | Carlos     | López     | carlos.lopez@emp.com   | Marketing
...
```

**Verificación:**

```bash
# El número de registros en la tabla destino debe ser igual al de la fuente
docker exec postgres psql -U postgres -d postgres \
  -c "SELECT COUNT(*) FROM employees;"

docker exec postgres psql -U postgres -d employees_sink \
  -c "SELECT COUNT(*) FROM employees;"
```

Ambos conteos deben coincidir (o estar muy cerca si el Source sigue ingiriendo datos).

---

### Paso 4: Probar el Pipeline End-to-End con Nuevos Registros

**Objetivo:** Verificar que el pipeline completo funciona en tiempo real insertando nuevos registros en la base de datos fuente y confirmando que aparecen en la base de datos destino a través de Kafka.

**Instrucciones:**

1. Inserta nuevos registros en la tabla fuente (base de datos `postgres`, esquema `public`):

   ```bash
   docker exec postgres psql -U postgres -d postgres -c "
   INSERT INTO employees (first_name, last_name, email, department, salary, hire_date)
   VALUES
     ('Laura',   'Martínez', 'laura.martinez@empresa.com',  'Data Engineering', 75000.00, '2024-01-15'),
     ('Roberto', 'Sánchez',  'roberto.sanchez@empresa.com', 'DevOps',           82000.00, '2024-02-20'),
     ('Sofía',   'Ramírez',  'sofia.ramirez@empresa.com',   'Product',          70000.00, '2024-03-10');
   "
   ```

2. Espera 5-10 segundos para que el conector Source detecte los cambios y los publique en Kafka, y luego el Sink los consuma:

   ```bash
   sleep 10
   ```

3. Verifica que los nuevos registros llegaron a la base de datos destino:

   ```bash
   docker exec postgres psql -U postgres -d employees_sink -c "
   SELECT id, first_name, last_name, department, salary
   FROM employees
   WHERE first_name IN ('Laura', 'Roberto', 'Sofía')
   ORDER BY id DESC;
   "
   ```

4. Compara el conteo total entre ambas bases de datos:

   ```bash
   echo "=== Registros en BD FUENTE ==="
   docker exec postgres psql -U postgres -d postgres \
     -c "SELECT COUNT(*) as total FROM employees;"

   echo "=== Registros en BD DESTINO ==="
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT COUNT(*) as total FROM employees;"
   ```

5. Verifica el offset del consumidor del conector Sink para confirmar que está al día:

   ```bash
   # Ver el lag del consumer group del conector Sink
   docker exec broker kafka-consumer-groups \
     --bootstrap-server broker:29092 \
     --group connect-jdbc-sink-employees-insert \
     --describe
   ```

**Salida Esperada:**

```
=== Registros en BD FUENTE ===
 total
-------
    28
(1 row)

=== Registros en BD DESTINO ===
 total
-------
    28
(1 row)

 id | first_name | last_name  |   department    |  salary
----+------------+------------+-----------------+----------
 26 | Laura      | Martínez   | Data Engineering| 75000.00
 27 | Roberto    | Sánchez    | DevOps          | 82000.00
 28 | Sofía      | Ramírez    | Product         | 70000.00
```

**Verificación:**

- Los 3 nuevos registros deben aparecer en la BD destino.
- El conteo total debe ser idéntico en ambas bases de datos.
- El lag del consumer group debe ser 0 o cercano a 0.

---

### Paso 5: Configurar el Conector JDBC Sink en Modo UPSERT

**Objetivo:** Reemplazar el conector de modo `insert` por uno en modo `upsert` para manejar correctamente las actualizaciones de registros sin generar duplicados, usando la clave primaria `id` como campo de deduplicación.

**Instrucciones:**

1. Elimina el conector anterior en modo `insert` para evitar conflictos:

   ```bash
   curl -X DELETE http://localhost:8083/connectors/jdbc-sink-employees-insert

   # Verificar que fue eliminado
   curl -s http://localhost:8083/connectors | jq .
   ```

2. Limpia los datos de la tabla destino para empezar con el modo upsert desde cero:

   ```bash
   docker exec postgres psql -U postgres -d employees_sink \
     -c "TRUNCATE TABLE employees RESTART IDENTITY;"
   ```

3. Crea el nuevo conector en modo `upsert`. Este modo usa `INSERT ... ON CONFLICT DO UPDATE` en PostgreSQL, garantizando que si un registro con la misma clave primaria ya existe, se actualiza en lugar de duplicarse:

   ```bash
   curl -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "jdbc-sink-employees-upsert",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
         "tasks.max": "1",
         "topics": "postgres.public.employees",
         "connection.url": "jdbc:postgresql://postgres:5432/employees_sink",
         "connection.user": "kafka_sink_user",
         "connection.password": "sink_password_2024",
         "insert.mode": "upsert",
         "auto.create": "false",
         "auto.evolve": "false",
         "table.name.format": "employees",
         "pk.mode": "record_value",
         "pk.fields": "id",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "errors.tolerance": "all",
         "errors.deadletterqueue.topic.name": "employees-sink-dlq",
         "errors.deadletterqueue.topic.replication.factor": "1",
         "errors.log.enable": "true",
         "errors.log.include.messages": "true"
       }
     }'
   ```

4. Verifica que el conector upsert está en estado `RUNNING`:

   ```bash
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-upsert/status | \
     jq '{name: .name, connector_state: .connector.state, task_state: .tasks[0].state, type: .type}'
   ```

5. Espera que el conector procese todos los mensajes del tópico desde el inicio:

   ```bash
   sleep 20

   # Verificar cuántos registros llegaron a la tabla destino
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT COUNT(*) as total_registros FROM employees;"
   ```

**Salida Esperada:**

```json
{
  "name": "jdbc-sink-employees-upsert",
  "connector_state": "RUNNING",
  "task_state": "RUNNING",
  "type": "sink"
}
```

```
 total_registros
-----------------
              28
(1 row)
```

**Verificación:**

```bash
# Confirmar que no hay duplicados (el COUNT debe igualar el número de IDs únicos)
docker exec postgres psql -U postgres -d employees_sink \
  -c "SELECT COUNT(*) as total, COUNT(DISTINCT id) as unicos FROM employees;"
```

Los valores `total` y `unicos` deben ser idénticos.

---

### Paso 6: Probar el Comportamiento UPSERT con Actualizaciones

**Objetivo:** Demostrar que el modo `upsert` maneja correctamente las actualizaciones de registros existentes, actualizando los datos en la tabla destino sin crear duplicados.

**Instrucciones:**

1. Actualiza un registro existente en la base de datos fuente y observa cómo se propaga:

   ```bash
   # Primero, ver el valor actual del registro con id=1 en la BD destino
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT id, first_name, last_name, department, salary FROM employees WHERE id = 1;"

   # Actualizar el registro en la BD fuente
   docker exec postgres psql -U postgres -d postgres -c "
   UPDATE employees
   SET salary = 95000.00, department = 'Senior Engineering'
   WHERE id = 1;
   "
   ```

2. Espera que el conector Source detecte el cambio y el Sink lo procese:

   ```bash
   sleep 15
   ```

3. Verifica que el registro fue actualizado en la BD destino (no duplicado):

   ```bash
   # Verificar el registro actualizado en BD destino
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT id, first_name, last_name, department, salary FROM employees WHERE id = 1;"

   # Confirmar que el total de registros no cambió (no hubo duplicación)
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT COUNT(*) as total FROM employees;"
   ```

4. Realiza múltiples actualizaciones para estresar el pipeline:

   ```bash
   docker exec postgres psql -U postgres -d postgres -c "
   UPDATE employees SET salary = salary * 1.10 WHERE department = 'Engineering';
   UPDATE employees SET department = 'Tech' WHERE department = 'DevOps';
   "

   sleep 15

   # Verificar que los cambios se propagaron correctamente
   echo "=== BD FUENTE ==="
   docker exec postgres psql -U postgres -d postgres \
     -c "SELECT id, department, salary FROM employees WHERE id IN (1,2,3,27) ORDER BY id;"

   echo "=== BD DESTINO ==="
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT id, department, salary FROM employees WHERE id IN (1,2,3,27) ORDER BY id;"
   ```

**Salida Esperada:**

```
# Antes de la actualización (BD destino):
 id | first_name | last_name | department  | salary
----+------------+-----------+-------------+----------
  1 | Ana        | García    | Engineering | 75000.00

# Después de la actualización (BD destino):
 id | first_name | last_name |     department     | salary
----+------------+-----------+--------------------+----------
  1 | Ana        | García    | Senior Engineering | 95000.00

# Total de registros (debe mantenerse igual, sin duplicados):
 total
-------
    28
```

**Verificación:**

```bash
# Comparación final de datos entre ambas bases de datos
echo "=== Comparación de totales ==="
echo -n "Fuente: "
docker exec postgres psql -U postgres -d postgres -t \
  -c "SELECT COUNT(*) FROM employees;"
echo -n "Destino: "
docker exec postgres psql -U postgres -d employees_sink -t \
  -c "SELECT COUNT(*) FROM employees;"
```

---

### Paso 7: Explorar la Dead Letter Queue (DLQ)

**Objetivo:** Verificar que la Dead Letter Queue configurada en el conector Sink captura mensajes problemáticos sin detener el pipeline, y aprender a inspeccionarlos para diagnóstico.

**Instrucciones:**

1. Verifica si el tópico DLQ fue creado automáticamente por el conector:

   ```bash
   docker exec broker kafka-topics \
     --bootstrap-server broker:29092 \
     --list | grep dlq
   ```

2. Para simular un error y enviar un mensaje a la DLQ, produce un mensaje con formato inválido directamente en el tópico fuente:

   ```bash
   # Producir un mensaje malformado (texto plano en lugar de Avro)
   echo "este-mensaje-no-es-avro-valido" | \
     docker exec -i broker kafka-console-producer \
       --bootstrap-server broker:29092 \
       --topic postgres.public.employees
   ```

3. Espera unos segundos y verifica si el mensaje problemático fue enviado a la DLQ:

   ```bash
   sleep 5

   # Consumir mensajes del tópico DLQ
   docker exec broker kafka-console-consumer \
     --bootstrap-server broker:29092 \
     --topic employees-sink-dlq \
     --from-beginning \
     --max-messages 5 \
     --timeout-ms 5000
   ```

4. Consulta los logs del conector para ver los errores registrados:

   ```bash
   # Ver logs del contenedor kafka-connect filtrando por errores del conector
   docker logs kafka-connect 2>&1 | grep -i "error" | grep -i "employees" | tail -10
   ```

5. Verifica que a pesar del mensaje malformado, el conector Sink sigue en estado `RUNNING`:

   ```bash
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-upsert/status | \
     jq '{connector_state: .connector.state, task_state: .tasks[0].state}'
   ```

**Salida Esperada:**

```
# Tópico DLQ creado:
employees-sink-dlq

# Estado del conector (debe seguir RUNNING a pesar del mensaje malformado):
{
  "connector_state": "RUNNING",
  "task_state": "RUNNING"
}
```

**Verificación:**

- El conector debe permanecer en estado `RUNNING` después del mensaje malformado.
- El tópico `employees-sink-dlq` debe existir y contener el mensaje problemático.
- Los registros válidos en la BD destino no deben verse afectados.

---

### Paso 8: Revisar la Configuración Completa del Conector y Listar Todos los Conectores

**Objetivo:** Obtener una visión completa del estado del entorno Kafka Connect con todos los conectores activos, y revisar la configuración detallada del conector Sink como referencia final.

**Instrucciones:**

1. Lista todos los conectores activos en el worker:

   ```bash
   curl -s http://localhost:8083/connectors | jq .
   ```

2. Obtén el estado de todos los conectores en un solo comando:

   ```bash
   # Estado de todos los conectores
   for connector in $(curl -s http://localhost:8083/connectors | jq -r '.[]'); do
     echo "=== $connector ==="
     curl -s http://localhost:8083/connectors/$connector/status | \
       jq '{state: .connector.state, tasks: [.tasks[] | {id: .id, state: .state}]}'
   done
   ```

3. Consulta la configuración completa del conector Sink para verificar todos los parámetros:

   ```bash
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-upsert/config | jq .
   ```

4. Realiza una consulta de validación final comparando los datos en ambas bases de datos:

   ```bash
   echo "=== RESUMEN FINAL DEL PIPELINE ==="
   echo ""
   echo "--- Registros en BD FUENTE (postgres.public.employees) ---"
   docker exec postgres psql -U postgres -d postgres \
     -c "SELECT department, COUNT(*) as empleados, AVG(salary)::numeric(10,2) as salario_promedio FROM employees GROUP BY department ORDER BY department;"

   echo ""
   echo "--- Registros en BD DESTINO (employees_sink.public.employees) ---"
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT department, COUNT(*) as empleados, AVG(salary)::numeric(10,2) as salario_promedio FROM employees GROUP BY department ORDER BY department;"
   ```

**Salida Esperada:**

```
# Lista de conectores activos:
[
  "jdbc-source-employees",
  "jdbc-sink-employees-upsert"
]

# Resumen final (los datos deben coincidir entre fuente y destino):
--- BD FUENTE ---
    department     | empleados | salario_promedio
-------------------+-----------+-----------------
 Data Engineering  |         1 |        75000.00
 Marketing         |         5 |        68000.00
 Product           |         3 |        72000.00
 Senior Engineering|         1 |        95000.00
 Tech              |         2 |        82000.00
...

--- BD DESTINO ---
(mismos resultados)
```

**Verificación:**

- Ambas bases de datos deben mostrar los mismos datos agrupados por departamento.
- Los dos conectores (Source y Sink) deben estar en estado `RUNNING`.

## Validación y Pruebas

### Criterios de Éxito

- [ ] La base de datos `employees_sink` contiene la tabla `employees` con el mismo número de registros que la tabla fuente.
- [ ] El conector `jdbc-sink-employees-upsert` está en estado `RUNNING` con todas sus tareas activas.
- [ ] Las actualizaciones realizadas en la BD fuente se reflejan correctamente en la BD destino sin crear duplicados.
- [ ] El tópico `employees-sink-dlq` existe y captura mensajes problemáticos sin detener el pipeline.
- [ ] El pipeline end-to-end (BD fuente → Source Connector → Kafka → Sink Connector → BD destino) funciona en tiempo real con latencia inferior a 30 segundos.

### Procedimiento de Pruebas

1. **Prueba de integridad de datos:**

   ```bash
   # Comparar hash de datos entre fuente y destino
   echo "=== Hash de datos en BD FUENTE ==="
   docker exec postgres psql -U postgres -d postgres -t \
     -c "SELECT MD5(STRING_AGG(id::text || first_name || last_name || email, ',' ORDER BY id)) as hash FROM employees;"

   echo "=== Hash de datos en BD DESTINO ==="
   docker exec postgres psql -U postgres -d employees_sink -t \
     -c "SELECT MD5(STRING_AGG(id::text || first_name || last_name || email, ',' ORDER BY id)) as hash FROM employees;"
   ```
   **Resultado Esperado:** Ambos hashes deben ser idénticos.

2. **Prueba de latencia del pipeline:**

   ```bash
   # Insertar un registro con timestamp y medir cuándo aparece en destino
   docker exec postgres psql -U postgres -d postgres -c "
   INSERT INTO employees (first_name, last_name, email, department, salary, hire_date)
   VALUES ('Test', 'Latencia', 'test.latencia@empresa.com', 'QA', 60000.00, CURRENT_DATE);
   "
   
   echo "Registro insertado a las: $(date)"
   
   # Verificar cada 5 segundos hasta que aparezca
   for i in {1..6}; do
     sleep 5
     count=$(docker exec postgres psql -U postgres -d employees_sink -t \
       -c "SELECT COUNT(*) FROM employees WHERE email = 'test.latencia@empresa.com';" | tr -d ' ')
     echo "Intento $i ($(date +%H:%M:%S)): registros encontrados = $count"
     if [ "$count" -gt "0" ]; then
       echo "✓ Registro llegó al destino en menos de $((i*5)) segundos"
       break
     fi
   done
   ```
   **Resultado Esperado:** El registro debe aparecer en la BD destino en menos de 30 segundos.

3. **Prueba de resiliencia con DLQ:**

   ```bash
   # Verificar que el conector sigue RUNNING después de mensajes problemáticos
   curl -s http://localhost:8083/connectors/jdbc-sink-employees-upsert/status | \
     jq '.connector.state, .tasks[0].state'
   ```
   **Resultado Esperado:** Ambos estados deben ser `"RUNNING"`.

4. **Prueba de no-duplicación en modo upsert:**

   ```bash
   # Actualizar el mismo registro 3 veces y verificar que no se duplica
   for salary in 80000 85000 90000; do
     docker exec postgres psql -U postgres -d postgres -c \
       "UPDATE employees SET salary = $salary WHERE id = 2;"
     sleep 8
   done
   
   echo "=== Conteo en BD DESTINO (debe ser 1 para id=2) ==="
   docker exec postgres psql -U postgres -d employees_sink \
     -c "SELECT COUNT(*) as ocurrencias, salary FROM employees WHERE id = 2 GROUP BY salary;"
   ```
   **Resultado Esperado:** Solo debe haber 1 fila con `id=2` y el salary más reciente (90000).

## Solución de Problemas

### Problema 1: El Conector Sink Falla con Error de Conexión a PostgreSQL

**Síntomas:**
- El estado del conector muestra `"state": "FAILED"` en la tarea.
- El campo `trace` contiene `Connection refused` o `FATAL: password authentication failed`.

**Causa:**
El conector no puede conectarse a PostgreSQL. Las causas más comunes son: URL de conexión incorrecta, credenciales inválidas, o el usuario `kafka_sink_user` no tiene permisos suficientes en la base de datos destino.

**Solución:**

```bash
# 1. Verificar la URL de conexión probando desde dentro de la red Docker
docker exec kafka-connect curl -s \
  "http://postgres:5432" 2>&1 || echo "Puerto accesible"

# 2. Probar la conexión con las credenciales del conector
docker exec postgres psql \
  -U kafka_sink_user \
  -d employees_sink \
  -h localhost \
  -c "SELECT current_user, current_database();"

# 3. Si falla, recrear el usuario y permisos
docker exec postgres psql -U postgres -d employees_sink -c "
DROP USER IF EXISTS kafka_sink_user;
CREATE USER kafka_sink_user WITH PASSWORD 'sink_password_2024';
GRANT CONNECT ON DATABASE employees_sink TO kafka_sink_user;
GRANT USAGE ON SCHEMA public TO kafka_sink_user;
GRANT INSERT, UPDATE, SELECT ON ALL TABLES IN SCHEMA public TO kafka_sink_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO kafka_sink_user;
"

# 4. Reiniciar la tarea fallida del conector
curl -X POST \
  http://localhost:8083/connectors/jdbc-sink-employees-upsert/tasks/0/restart

# 5. Verificar el nuevo estado
curl -s http://localhost:8083/connectors/jdbc-sink-employees-upsert/status | jq .
```

---

### Problema 2: Error de Esquema - Campo no Encontrado en la Tabla

**Síntomas:**
- La tarea del conector falla con un error que contiene `column "campo_x" of relation "employees" does not exist`.
- Los mensajes Avro tienen campos que no existen en la tabla destino de PostgreSQL.

**Causa:**
El esquema Avro del tópico tiene campos adicionales que no están mapeados en la tabla destino, o los tipos de datos son incompatibles entre Avro y PostgreSQL.

**Solución:**

```bash
# 1. Ver el esquema Avro exacto del tópico
curl -s http://localhost:8081/subjects/postgres.public.employees-value/versions/latest | \
  jq '.schema | fromjson | .fields[] | {name: .name, type: .type}'

# 2. Ver las columnas de la tabla destino
docker exec postgres psql -U postgres -d employees_sink \
  -c "\d employees"

# 3. Opción A: Agregar las columnas faltantes a la tabla destino
docker exec postgres psql -U postgres -d employees_sink -c "
ALTER TABLE employees ADD COLUMN IF NOT EXISTS campo_faltante TEXT;
"

# 4. Opción B: Usar transforms para excluir campos del mensaje antes de escribir
# Actualizar la configuración del conector para excluir campos problemáticos
curl -X PUT http://localhost:8083/connectors/jdbc-sink-employees-upsert/config \
  -H "Content-Type: application/json" \
  -d '{
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "postgres.public.employees",
    "connection.url": "jdbc:postgresql://postgres:5432/employees_sink",
    "connection.user": "kafka_sink_user",
    "connection.password": "sink_password_2024",
    "insert.mode": "upsert",
    "auto.create": "false",
    "auto.evolve": "false",
    "table.name.format": "employees",
    "pk.mode": "record_value",
    "pk.fields": "id",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "transforms": "excludeFields",
    "transforms.excludeFields.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.excludeFields.exclude": "campo_problematico,otro_campo",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "employees-sink-dlq",
    "errors.deadletterqueue.topic.replication.factor": "1",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true"
  }'

# 5. Reiniciar la tarea después de la actualización
curl -X POST \
  http://localhost:8083/connectors/jdbc-sink-employees-upsert/tasks/0/restart
```

---

### Problema 3: Duplicados en la Tabla Destino con Modo UPSERT

**Síntomas:**
- La tabla destino tiene más registros que la fuente.
- `SELECT COUNT(*) != SELECT COUNT(DISTINCT id)` en la tabla destino.
- Se observan múltiples filas con el mismo `id`.

**Causa:**
El modo `upsert` requiere que `pk.fields` apunte a columnas que son `PRIMARY KEY` o tienen una restricción `UNIQUE` en la tabla destino. Si la tabla no tiene esta restricción, PostgreSQL no puede realizar el `ON CONFLICT DO UPDATE` y en su lugar inserta duplicados.

**Solución:**

```bash
# 1. Verificar si la tabla tiene la restricción PRIMARY KEY
docker exec postgres psql -U postgres -d employees_sink \
  -c "\d employees" | grep -E "PRIMARY|UNIQUE|Index"

# 2. Si no tiene PRIMARY KEY, agregar la restricción
# Primero, eliminar duplicados existentes
docker exec postgres psql -U postgres -d employees_sink -c "
DELETE FROM employees a
USING employees b
WHERE a.ctid < b.ctid AND a.id = b.id;
"

# 3. Agregar la restricción PRIMARY KEY si no existe
docker exec postgres psql -U postgres -d employees_sink -c "
ALTER TABLE employees ADD CONSTRAINT employees_pkey PRIMARY KEY (id);
"

# 4. Verificar que la restricción fue creada
docker exec postgres psql -U postgres -d employees_sink \
  -c "\d employees"

# 5. Reiniciar el conector para que reintente los mensajes
curl -X POST \
  http://localhost:8083/connectors/jdbc-sink-employees-upsert/restart

# 6. Verificar que no hay más duplicados
docker exec postgres psql -U postgres -d employees_sink \
  -c "SELECT COUNT(*) as total, COUNT(DISTINCT id) as unicos FROM employees;"
```

---

### Problema 4: El Conector Sink No Procesa Mensajes Históricos (Offset Incorrecto)

**Síntomas:**
- El conector Sink está en estado `RUNNING` pero la tabla destino está vacía o tiene muy pocos registros.
- El lag del consumer group es muy alto o el conector empieza desde el offset más reciente.

**Causa:**
Por defecto, un nuevo conector Sink comienza a consumir desde el offset más reciente (`latest`). Si el tópico ya tenía mensajes antes de crear el conector, estos no serán procesados automáticamente.

**Solución:**

```bash
# 1. Verificar el offset actual del consumer group del conector
docker exec broker kafka-consumer-groups \
  --bootstrap-server broker:29092 \
  --group connect-jdbc-sink-employees-upsert \
  --describe

# 2. Resetear el offset del consumer group al inicio del tópico
# IMPORTANTE: El conector debe estar pausado antes de resetear
curl -X PUT http://localhost:8083/connectors/jdbc-sink-employees-upsert/pause

sleep 3

# 3. Resetear el offset al inicio
docker exec broker kafka-consumer-groups \
  --bootstrap-server broker:29092 \
  --group connect-jdbc-sink-employees-upsert \
  --topic postgres.public.employees \
  --reset-offsets \
  --to-earliest \
  --execute

# 4. Reanudar el conector
curl -X PUT http://localhost:8083/connectors/jdbc-sink-employees-upsert/resume

# 5. Monitorear el progreso
sleep 15
docker exec postgres psql -U postgres -d employees_sink \
  -c "SELECT COUNT(*) FROM employees;"
```

## Limpieza

Ejecuta los siguientes comandos para limpiar los recursos creados en este laboratorio. **Lee cuidadosamente las advertencias antes de ejecutar.**

```bash
# 1. Eliminar los conectores Sink creados en este laboratorio
curl -X DELETE http://localhost:8083/connectors/jdbc-sink-employees-upsert
curl -X DELETE http://localhost:8083/connectors/jdbc-sink-employees-insert 2>/dev/null || true

# 2. Verificar que los conectores fueron eliminados
curl -s http://localhost:8083/connectors | jq .

# 3. Eliminar el tópico DLQ creado durante el laboratorio
docker exec broker kafka-topics \
  --bootstrap-server broker:29092 \
  --delete \
  --topic employees-sink-dlq

# 4. Limpiar la base de datos destino (preservar la estructura para laboratorios futuros)
docker exec postgres psql -U postgres -d employees_sink \
  -c "TRUNCATE TABLE employees RESTART IDENTITY;"

# 5. (OPCIONAL) Eliminar completamente la base de datos destino si no se necesita más
# docker exec postgres psql -U postgres -c "DROP DATABASE employees_sink;"

# 6. Verificar el estado final del entorno
echo "=== Conectores activos restantes ==="
curl -s http://localhost:8083/connectors | jq .

echo "=== Tópicos en Kafka ==="
docker exec broker kafka-topics \
  --bootstrap-server broker:29092 \
  --list
```

> ⚠️ **Advertencia:** NO ejecutes `docker compose down -v` al finalizar este laboratorio si planeas continuar con el Laboratorio 6. Los volúmenes Docker contienen los datos de Kafka y PostgreSQL que serán necesarios en la siguiente sesión. Usa únicamente `docker compose down` (sin `-v`) para detener los servicios preservando los datos. El conector JDBC Source del Laboratorio 4 debe permanecer activo para los laboratorios posteriores.

> ⚠️ **Nota sobre el usuario kafka_sink_user:** Si eliminas la base de datos `employees_sink`, el usuario `kafka_sink_user` permanecerá en PostgreSQL. Puedes eliminarlo con `DROP USER kafka_sink_user;` desde una sesión con el usuario `postgres`.

## Resumen

### Lo que Lograste

- Creaste una base de datos PostgreSQL de destino con la estructura correcta para recibir datos replicados desde Kafka.
- Desplegaste el conector JDBC Sink en modo `insert` y comprendiste su funcionamiento básico.
- Migraste el conector al modo `upsert` con `pk.mode=record_value` para manejar actualizaciones sin duplicados.
- Construiste y validaste un pipeline end-to-end completo: BD fuente → JDBC Source → Tópico Kafka → JDBC Sink → BD destino.
- Configuraste una Dead Letter Queue (DLQ) para tolerancia a errores en producción.
- Verificaste la integridad de los datos comparando registros en ambos extremos del pipeline.

### Conceptos Clave Aprendidos

- **Modos de escritura del JDBC Sink:** `insert` para cargas iniciales simples, `upsert` para sincronización continua con actualizaciones, y la importancia de `pk.fields` para la deduplicación.
- **`auto.create` vs tabla manual:** Crear la tabla manualmente ofrece control total sobre tipos de datos, restricciones e índices; `auto.create=true` es conveniente para prototipos pero arriesgado en producción.
- **Dead Letter Queue (DLQ):** El patrón `errors.tolerance=all` + `errors.deadletterqueue.topic.name` permite construir pipelines resilientes que no se detienen ante mensajes problemáticos.
- **Compatibilidad de esquemas:** El Avro Converter con Schema Registry garantiza que el conector Sink entiende la estructura de los mensajes y puede mapearlos correctamente a columnas de la tabla destino.
- **Consumer Groups en Kafka Connect:** Cada conector Sink tiene su propio consumer group, lo que permite gestionar offsets independientemente y resetearlos si es necesario.

### Próximos Pasos

- **Laboratorio 6:** Explora ksqlDB para realizar procesamiento de flujos en tiempo real sobre los datos que ahora fluyen por el pipeline construido en estos dos laboratorios.
- **Exploración adicional:** Investiga el modo `update` del JDBC Sink (solo actualiza registros existentes, nunca inserta nuevos) y cuándo es apropiado usarlo.
- **Producción:** Revisa la documentación de Confluent sobre configuración de `batch.size` y `consumer.override.max.poll.records` para optimizar el rendimiento del conector Sink con grandes volúmenes de datos.

## Recursos Adicionales

- [Documentación oficial del JDBC Sink Connector](https://docs.confluent.io/kafka-connectors/jdbc/current/sink-connector/overview.html) - Referencia completa de todas las propiedades de configuración del conector JDBC Sink, incluyendo modos de escritura, transformaciones y manejo de errores.
- [Confluent Hub - JDBC Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) - Página oficial del plugin con historial de versiones, changelog y ejemplos de configuración.
- [Dead Letter Queue en Kafka Connect](https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/) - Artículo técnico de Confluent que explica en profundidad el manejo de errores y el patrón DLQ en Kafka Connect.
- [Documentación de Kafka Connect Transforms](https://kafka.apache.org/documentation/#connect_transforms) - Referencia de las Single Message Transforms (SMT) disponibles para filtrar, renombrar y transformar campos antes de escribirlos en el destino.
- [PostgreSQL - INSERT ON CONFLICT](https://www.postgresql.org/docs/15/sql-insert.html#SQL-ON-CONFLICT) - Documentación oficial de PostgreSQL sobre el comportamiento de UPSERT que subyace al modo `upsert` del conector JDBC Sink.

---

# Consultas en Tiempo Real con ksqlDB

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 42 minutos |
| **Complejidad** | Difícil |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio aprenderás a usar ksqlDB para procesar flujos de eventos en tiempo real directamente sobre tópicos de Apache Kafka, usando una sintaxis SQL familiar. Construirás streams, tablas materializadas y pipelines de transformación que consumen, filtran, agregan y producen eventos hacia nuevos tópicos Kafka, todo sin escribir código Java o Python.

Este conocimiento es directamente aplicable en escenarios de producción donde se requiere detectar patrones en tiempo real, calcular métricas en ventanas de tiempo o enriquecer eventos al vuelo, como en sistemas de detección de fraude, monitoreo de transacciones o análisis de comportamiento de usuarios.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear STREAM y TABLE en ksqlDB sobre tópicos Kafka existentes definiendo el esquema explícitamente.
- [ ] Escribir y ejecutar consultas push (`SELECT ... EMIT CHANGES`) y pull para observar y consultar datos en tiempo real.
- [ ] Aplicar operaciones de filtrado, transformación y agregación con ventanas temporales TUMBLING sobre streams de eventos.
- [ ] Construir un pipeline de procesamiento completo: stream crudo → transformación → nuevo tópico Kafka con datos procesados.

## Prerrequisitos

### Conocimiento Requerido

- Haber completado los Laboratorios 4 y 5 con datos fluyendo en tópicos Kafka (especialmente el tópico `orders` generado por el conector Datagen).
- Conocimiento básico de SQL: `SELECT`, `WHERE`, `GROUP BY`, funciones de agregación (`COUNT`, `SUM`).
- Comprensión del concepto de streams (datos en movimiento) vs tablas (estado materializado) en procesamiento de eventos.
- Familiaridad con la API REST de Kafka Connect y el entorno Docker Compose del curso.

### Acceso Requerido

- Entorno Docker Compose del curso ejecutándose con los servicios: `broker`, `schema-registry`, `kafka-connect`, `ksqldb-server`, `ksqldb-cli` y `control-center`.
- Acceso a la terminal con Docker instalado (versión 24.0+).
- Acceso al navegador web para Confluent Control Center en `http://localhost:9021`.
- Puerto `8088` disponible para la API REST de ksqlDB.

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| RAM disponible para Docker | Mínimo 10 GB (recomendado 12 GB) |
| CPU | Mínimo 4 núcleos físicos |
| Espacio en disco | Al menos 5 GB libres adicionales |
| Conectividad | Red local entre contenedores Docker |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Ejecutar todos los servicios del laboratorio |
| Docker Compose | 2.20+ (plugin v2) | Orquestar el ecosistema Kafka completo |
| ksqlDB Server | 8.0.0 (Confluent Platform) | Motor de procesamiento de flujos SQL |
| ksqlDB CLI | 8.0.0 | Interfaz de línea de comandos para ksqlDB |
| Apache Kafka | 4.0 (via Confluent Platform 8.0) | Broker de mensajería y almacenamiento de tópicos |
| Schema Registry | 8.0.0 | Gestión de esquemas Avro |
| Confluent Control Center | 8.0.0 | Interfaz web de administración |
| curl | 7.x+ | Interacción con la API REST de ksqlDB |
| jq | 1.6+ | Formateo de respuestas JSON en terminal |

### Configuración Inicial

Antes de comenzar, verifica que el entorno Docker Compose del curso esté completamente operativo. Si no tienes el archivo `docker-compose.yml` con ksqlDB configurado, usa el siguiente bloque para agregar los servicios necesarios o verifica que ya estén presentes:

```bash
# Verificar que todos los contenedores estén en estado "running"
docker compose ps

# La salida debe mostrar los siguientes servicios como "running":
# broker, schema-registry, kafka-connect, ksqldb-server, ksqldb-cli, control-center
```

Si el servicio `ksqldb-server` no aparece, agrega la siguiente configuración a tu `docker-compose.yml` y ejecuta `docker compose up -d`:

```yaml
# Agregar al docker-compose.yml existente del curso

  ksqldb-server:
    image: confluentinc/cp-ksqldb-server:8.0.0
    hostname: ksqldb-server
    container_name: ksqldb-server
    depends_on:
      - broker
      - kafka-connect
    ports:
      - "8088:8088"
    environment:
      KSQL_CONFIG_DIR: "/etc/ksql"
      KSQL_BOOTSTRAP_SERVERS: "broker:29092"
      KSQL_HOST_NAME: ksqldb-server
      KSQL_LISTENERS: "http://0.0.0.0:8088"
      KSQL_CACHE_MAX_BYTES_BUFFERING: 0
      KSQL_KSQL_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_PRODUCER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor"
      KSQL_CONSUMER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor"
      KSQL_KSQL_CONNECT_URL: "http://kafka-connect:8083"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_REPLICATION_FACTOR: 1
      KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"

  ksqldb-cli:
    image: confluentinc/cp-ksqldb-cli:8.0.0
    container_name: ksqldb-cli
    depends_on:
      - broker
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true
```

```bash
# Levantar los servicios actualizados (sin eliminar volúmenes)
docker compose up -d

# Esperar 60 segundos para que ksqlDB Server inicialice completamente
sleep 60

# Verificar que ksqlDB Server responde en el puerto 8088
curl -s http://localhost:8088/info | jq .
```

Verifica que el tópico `orders` existe con datos del laboratorio anterior:

```bash
# Listar tópicos disponibles en el broker
docker exec broker kafka-topics --bootstrap-server broker:29092 --list | grep orders

# Verificar que hay mensajes en el tópico orders
docker exec broker kafka-console-consumer \
  --bootstrap-server broker:29092 \
  --topic orders \
  --from-beginning \
  --max-messages 3 \
  --timeout-ms 10000
```

Si el tópico `orders` no tiene datos, reactiva el conector Datagen del laboratorio anterior:

```bash
# Verificar estado del conector datagen-orders
curl -s http://localhost:8083/connectors/datagen-orders/status | jq .

# Si no existe, recrearlo
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "datagen-orders",
    "config": {
      "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
      "kafka.topic": "orders",
      "quickstart": "orders",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter": "io.confluent.connect.avro.AvroConverter",
      "value.converter.schema.registry.url": "http://schema-registry:8081",
      "max.interval": 1000,
      "iterations": 10000,
      "tasks.max": "1"
    }
  }'
```

## Instrucciones Paso a Paso

### Paso 1: Conexión al ksqlDB CLI y Exploración del Entorno

**Objetivo:** Conectarse al ksqlDB CLI interactivo y explorar los tópicos, streams y tablas disponibles en el entorno para comprender el estado inicial antes de crear objetos.

**Instrucciones:**

1. Abre una terminal y conéctate al ksqlDB CLI ejecutando el siguiente comando:

   ```bash
   docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
   ```

2. Una vez dentro del CLI, verifica la versión y el estado del servidor:

   ```sql
   SHOW PROPERTIES;
   ```

3. Explora los tópicos de Kafka disponibles que ksqlDB puede ver:

   ```sql
   SHOW TOPICS;
   ```

4. Verifica que no existan streams ni tablas previas (entorno limpio):

   ```sql
   SHOW STREAMS;
   SHOW TABLES;
   ```

5. Inspecciona el contenido crudo del tópico `orders` directamente desde ksqlDB para entender la estructura de los datos antes de crear el stream:

   ```sql
   PRINT 'orders' FROM BEGINNING LIMIT 5;
   ```

**Salida Esperada:**

```
                  CLI v8.0.0

===========================================
=        _  __ _____  ____  _       ____  =
=       | |/ // ____|/ __ \| |     |  _ \ =
=       | ' /| (___ | |  | | |     | |_) |=
=       |  <  \___ \| |  | | |     |  _ < =
=       | . \ ____) | |__| | |____ | |_) |=
=       |_|\_\_____/ \___\_\______|____/  =
=                                         =
=        The Database purpose-built       =
=        for stream processing apps       =
===========================================

Copyright 2017-2022 Confluent Inc.

CLI v8.0.0, Server v8.0.0 located at http://ksqldb-server:8088
Server Status: RUNNING
```

Para `SHOW TOPICS`, deberías ver el tópico `orders` en la lista:

```
 Kafka Topic                 | Partitions | Partition Replicas
-------------------------------------------------------------
 orders                      | 1          | 1
 docker-connect-configs      | 1          | 1
 docker-connect-offsets      | 25         | 1
 docker-connect-status       | 5          | 1
-------------------------------------------------------------
```

Para `PRINT 'orders'`, verás mensajes Avro deserializados con campos como `orderid`, `itemid`, `orderunits`, `address`:

```
Key format: KAFKA_STRING
Value format: AVRO
rowtime: 2024/01/15 10:23:45.123 Z, key: 1, value: {"orderid": 1, "itemid": "Item_5", "orderunits": 3.5, "address": {"city": "City_9", "state": "State_4", "zipcode": 90210}}
```

**Verificación:**

- El CLI muestra `Server Status: RUNNING`.
- El tópico `orders` aparece en `SHOW TOPICS`.
- `PRINT 'orders'` muestra mensajes con estructura JSON/Avro legible.
- `SHOW STREAMS` y `SHOW TABLES` retornan listas vacías (entorno limpio).

---

### Paso 2: Creación de un STREAM sobre el Tópico de Órdenes

**Objetivo:** Definir un STREAM de ksqlDB sobre el tópico `orders` existente, especificando explícitamente el esquema de columnas para habilitar consultas SQL tipadas sobre los eventos.

**Instrucciones:**

1. Dentro del ksqlDB CLI, configura el offset de lectura para comenzar desde el inicio del tópico (importante para ver datos históricos en este laboratorio):

   ```sql
   SET 'auto.offset.reset' = 'earliest';
   ```

2. Crea el STREAM `ORDERS_STREAM` sobre el tópico `orders`. El esquema debe coincidir con la estructura de los mensajes generados por Datagen:

   ```sql
   CREATE STREAM ORDERS_STREAM (
     ORDERID    BIGINT,
     ITEMID     VARCHAR,
     ORDERUNITS DOUBLE,
     ADDRESS    STRUCT<
       CITY    VARCHAR,
       STATE   VARCHAR,
       ZIPCODE BIGINT
     >
   ) WITH (
     KAFKA_TOPIC   = 'orders',
     VALUE_FORMAT  = 'AVRO',
     KEY_FORMAT    = 'KAFKA'
   );
   ```

3. Verifica que el stream fue creado correctamente:

   ```sql
   SHOW STREAMS;
   ```

4. Describe la estructura del stream para confirmar las columnas y tipos de datos:

   ```sql
   DESCRIBE ORDERS_STREAM;
   ```

5. Ejecuta una consulta simple para ver los primeros registros del stream:

   ```sql
   SELECT ORDERID, ITEMID, ORDERUNITS
   FROM ORDERS_STREAM
   LIMIT 5;
   ```

**Salida Esperada:**

Para `SHOW STREAMS`:

```
 Stream Name    | Kafka Topic | Key Format | Value Format | Windowed
--------------------------------------------------------------------
 ORDERS_STREAM  | orders      | KAFKA      | AVRO         | false
--------------------------------------------------------------------
```

Para `DESCRIBE ORDERS_STREAM`:

```
Name                 : ORDERS_STREAM
 Field      | Type
---------------------------------------------
 ORDERID    | BIGINT
 ITEMID     | VARCHAR(STRING)
 ORDERUNITS | DOUBLE
 ADDRESS    | STRUCT<CITY VARCHAR(STRING), STATE VARCHAR(STRING), ZIPCODE BIGINT>
---------------------------------------------
For runtime statistics and query details run: DESCRIBE EXTENDED <Stream,Table>;
```

Para el `SELECT`:

```
+----------+--------+------------+
|ORDERID   |ITEMID  |ORDERUNITS  |
+----------+--------+------------+
|1         |Item_5  |3.5         |
|2         |Item_2  |7.0         |
|3         |Item_8  |1.5         |
|4         |Item_1  |5.0         |
|5         |Item_6  |2.5         |
Limit Reached
Query terminated
```

**Verificación:**

- `SHOW STREAMS` lista `ORDERS_STREAM` con el tópico `orders`.
- `DESCRIBE ORDERS_STREAM` muestra todos los campos con sus tipos correctos.
- El `SELECT` retorna filas con datos reales del tópico.
- No hay errores de deserialización Avro.

---

### Paso 3: Consultas Push para Observar Eventos en Tiempo Real

**Objetivo:** Ejecutar consultas push con `EMIT CHANGES` para observar el flujo de eventos en tiempo real mientras nuevos mensajes llegan al tópico, comprendiendo la diferencia fundamental entre consultas push y pull en ksqlDB.

**Instrucciones:**

1. Abre una **segunda terminal** (mantén la primera con el CLI activo) y verifica que el conector Datagen sigue generando datos:

   ```bash
   # En una segunda terminal - verificar que el conector está activo
   curl -s http://localhost:8083/connectors/datagen-orders/status | jq '.connector.state'
   ```

2. Vuelve a la **primera terminal** (ksqlDB CLI) y ejecuta una consulta push básica para ver todos los eventos nuevos en tiempo real:

   ```sql
   SELECT ORDERID, ITEMID, ORDERUNITS, ROWTIME
   FROM ORDERS_STREAM
   EMIT CHANGES;
   ```

   > **Nota:** Esta consulta no termina sola. Observa cómo llegan nuevos registros cada segundo (según la configuración `max.interval=1000` del conector Datagen). Presiona `Ctrl+C` después de observar al menos 10 registros para detenerla.

3. Ejecuta una consulta push con filtrado para ver únicamente órdenes con más de 5 unidades:

   ```sql
   SELECT ORDERID, ITEMID, ORDERUNITS
   FROM ORDERS_STREAM
   WHERE ORDERUNITS > 5
   EMIT CHANGES;
   ```

   Observa durante 15 segundos y presiona `Ctrl+C`.

4. Ejecuta una consulta push que extrae campos de la estructura anidada `ADDRESS`:

   ```sql
   SELECT
     ORDERID,
     ITEMID,
     ORDERUNITS,
     ADDRESS->CITY    AS CIUDAD,
     ADDRESS->STATE   AS ESTADO,
     ADDRESS->ZIPCODE AS CODIGO_POSTAL
   FROM ORDERS_STREAM
   WHERE ADDRESS->STATE = 'State_1'
   EMIT CHANGES;
   ```

   Observa durante 20 segundos y presiona `Ctrl+C`.

5. Ahora ejecuta una consulta **pull** (sin `EMIT CHANGES`) para obtener una instantánea puntual. Nota que las consultas pull sobre streams requieren una condición de tiempo o se ejecutan sobre tablas materializadas:

   ```sql
   SELECT ORDERID, ITEMID, ORDERUNITS
   FROM ORDERS_STREAM
   WHERE ORDERID = 1;
   ```

**Salida Esperada:**

Para la consulta push básica (los valores variarán):

```
+----------+--------+------------+---------------------+
|ORDERID   |ITEMID  |ORDERUNITS  |ROWTIME              |
+----------+--------+------------+---------------------+
|156       |Item_3  |4.5         |2024-01-15 10:25:01  |
|157       |Item_7  |8.0         |2024-01-15 10:25:02  |
|158       |Item_1  |2.5         |2024-01-15 10:25:03  |
|159       |Item_9  |6.5         |2024-01-15 10:25:04  |
^CQuery terminated
```

Para la consulta push con filtro `ORDERUNITS > 5`:

```
+----------+--------+------------+
|ORDERID   |ITEMID  |ORDERUNITS  |
+----------+--------+------------+
|157       |Item_7  |8.0         |
|162       |Item_4  |7.5         |
|165       |Item_2  |9.0         |
^CQuery terminated
```

**Verificación:**

- La consulta push sin `LIMIT` continúa emitiendo resultados hasta que se interrumpe con `Ctrl+C`.
- Los registros llegan aproximadamente cada segundo (configuración del conector Datagen).
- El filtro `WHERE ORDERUNITS > 5` efectivamente excluye órdenes con pocas unidades.
- La extracción de campos anidados con `->` funciona correctamente.

---

### Paso 4: Creación de un Stream Derivado con Transformaciones

**Objetivo:** Crear un nuevo STREAM persistente usando `CREATE STREAM AS SELECT` que transforme y enriquezca los datos del stream original, generando automáticamente un nuevo tópico Kafka con los datos procesados.

**Instrucciones:**

1. En el ksqlDB CLI, crea un stream derivado que clasifique las órdenes por tamaño y extraiga los campos de dirección a nivel plano:

   ```sql
   CREATE STREAM ORDERS_ENRICHED
   WITH (
     KAFKA_TOPIC  = 'orders-enriched',
     VALUE_FORMAT = 'AVRO',
     PARTITIONS   = 1
   ) AS
   SELECT
     ORDERID,
     ITEMID,
     ORDERUNITS,
     ADDRESS->CITY                    AS CITY,
     ADDRESS->STATE                   AS STATE,
     ADDRESS->ZIPCODE                 AS ZIPCODE,
     CASE
       WHEN ORDERUNITS < 3  THEN 'SMALL'
       WHEN ORDERUNITS < 7  THEN 'MEDIUM'
       ELSE                      'LARGE'
     END                              AS ORDER_SIZE,
     ORDERUNITS * 29.99               AS ESTIMATED_VALUE_USD,
     TIMESTAMPTOSTRING(ROWTIME, 'yyyy-MM-dd HH:mm:ss', 'UTC') AS ORDER_TIMESTAMP
   FROM ORDERS_STREAM
   EMIT CHANGES;
   ```

2. Verifica que el stream derivado fue creado y la consulta persistente está activa:

   ```sql
   SHOW STREAMS;
   SHOW QUERIES;
   ```

3. Confirma que el nuevo tópico `orders-enriched` fue creado automáticamente en Kafka:

   ```bash
   # En una segunda terminal (fuera del CLI de ksqlDB)
   docker exec broker kafka-topics \
     --bootstrap-server broker:29092 \
     --list | grep orders
   ```

4. Vuelve al ksqlDB CLI y consulta el nuevo stream enriquecido:

   ```sql
   SELECT
     ORDERID,
     ITEMID,
     ORDER_SIZE,
     ESTIMATED_VALUE_USD,
     CITY,
     ORDER_TIMESTAMP
   FROM ORDERS_ENRICHED
   LIMIT 10;
   ```

5. Verifica la distribución de tamaños de órdenes con una consulta push de conteo:

   ```sql
   SELECT ORDER_SIZE, COUNT(*) AS TOTAL
   FROM ORDERS_ENRICHED
   GROUP BY ORDER_SIZE
   EMIT CHANGES
   LIMIT 15;
   ```

**Salida Esperada:**

Para `SHOW QUERIES`:

```
 Query ID                          | Query Type | Status  | Sink Name       | Sink Kafka Topic
--------------------------------------------------------------------------------------------
 CSAS_ORDERS_ENRICHED_0            | PERSISTENT | RUNNING | ORDERS_ENRICHED | orders-enriched
--------------------------------------------------------------------------------------------
```

Para la consulta al stream enriquecido:

```
+----------+--------+------------+---------------------+--------+---------------------+
|ORDERID   |ITEMID  |ORDER_SIZE  |ESTIMATED_VALUE_USD  |CITY    |ORDER_TIMESTAMP      |
+----------+--------+------------+---------------------+--------+---------------------+
|1         |Item_5  |MEDIUM      |104.965              |City_9  |2024-01-15 10:23:45  |
|2         |Item_2  |LARGE       |209.93               |City_3  |2024-01-15 10:23:46  |
|3         |Item_8  |SMALL       |44.985               |City_7  |2024-01-15 10:23:47  |
Limit Reached
Query terminated
```

Para la lista de tópicos en Kafka:

```
orders
orders-enriched
```

**Verificación:**

- `SHOW QUERIES` muestra la consulta persistente `CSAS_ORDERS_ENRICHED_0` con estado `RUNNING`.
- El tópico `orders-enriched` existe en Kafka.
- Los campos `ORDER_SIZE` y `ESTIMATED_VALUE_USD` son calculados correctamente.
- `ORDER_TIMESTAMP` tiene formato legible de fecha y hora.

---

### Paso 5: Agregaciones con Ventanas Temporales TUMBLING

**Objetivo:** Crear consultas de agregación con ventanas temporales TUMBLING para calcular métricas en intervalos de tiempo fijos, como el total de órdenes y el valor acumulado por estado en ventanas de 1 minuto.

**Instrucciones:**

1. En el ksqlDB CLI, ejecuta una consulta push de agregación con ventana TUMBLING de 1 minuto para contar órdenes por estado:

   ```sql
   SELECT
     STATE,
     WINDOWSTART                                                     AS WINDOW_START,
     WINDOWEND                                                       AS WINDOW_END,
     COUNT(*)                                                        AS TOTAL_ORDERS,
     SUM(ORDERUNITS)                                                 AS TOTAL_UNITS,
     ROUND(SUM(ESTIMATED_VALUE_USD), 2)                              AS TOTAL_VALUE_USD
   FROM ORDERS_ENRICHED
   WINDOW TUMBLING (SIZE 1 MINUTE)
   GROUP BY STATE
   EMIT CHANGES
   LIMIT 20;
   ```

   Observa cómo los conteos se acumulan dentro de cada ventana de 1 minuto.

2. Crea una TABLE materializada persistente que mantenga el estado actualizado de las agregaciones por ventana TUMBLING. Primero, crea la tabla:

   ```sql
   CREATE TABLE ORDERS_BY_STATE_PER_MINUTE
   WITH (
     KAFKA_TOPIC  = 'orders-by-state-per-minute',
     VALUE_FORMAT = 'AVRO',
     PARTITIONS   = 1
   ) AS
   SELECT
     STATE,
     WINDOWSTART                                AS WINDOW_START,
     WINDOWEND                                  AS WINDOW_END,
     COUNT(*)                                   AS TOTAL_ORDERS,
     ROUND(SUM(ESTIMATED_VALUE_USD), 2)         AS TOTAL_VALUE_USD,
     ROUND(AVG(ORDERUNITS), 2)                  AS AVG_UNITS
   FROM ORDERS_ENRICHED
   WINDOW TUMBLING (SIZE 1 MINUTE)
   GROUP BY STATE
   EMIT CHANGES;
   ```

3. Verifica que la tabla fue creada:

   ```sql
   SHOW TABLES;
   DESCRIBE ORDERS_BY_STATE_PER_MINUTE EXTENDED;
   ```

4. Espera 60 segundos para que se complete al menos una ventana de 1 minuto y luego ejecuta una consulta pull sobre la tabla materializada para obtener el estado actual:

   ```sql
   SELECT
     STATE,
     TOTAL_ORDERS,
     TOTAL_VALUE_USD,
     AVG_UNITS
   FROM ORDERS_BY_STATE_PER_MINUTE
   WHERE STATE = 'State_1';
   ```

5. Realiza una consulta pull sin filtro para ver todas las entradas de la tabla materializada (nota: en ksqlDB, las consultas pull sobre tablas con ventanas requieren especificar el estado o un rango de ventana):

   ```sql
   SELECT
     STATE,
     TOTAL_ORDERS,
     TOTAL_VALUE_USD
   FROM ORDERS_BY_STATE_PER_MINUTE
   WHERE STATE IN ('State_1', 'State_2', 'State_3', 'State_4', 'State_5');
   ```

**Salida Esperada:**

Para la consulta con `WINDOW TUMBLING`:

```
+----------+---------------------+---------------------+--------------+-------------+-----------------+
|STATE     |WINDOW_START         |WINDOW_END           |TOTAL_ORDERS  |TOTAL_UNITS  |TOTAL_VALUE_USD  |
+----------+---------------------+---------------------+--------------+-------------+-----------------+
|State_1   |2024-01-15 10:25:00  |2024-01-15 10:26:00  |7             |28.5         |854.715          |
|State_3   |2024-01-15 10:25:00  |2024-01-15 10:26:00  |5             |19.0         |569.81           |
|State_2   |2024-01-15 10:25:00  |2024-01-15 10:26:00  |9             |38.5         |1154.615         |
```

Para `SHOW TABLES`:

```
 Table Name                    | Kafka Topic                  | Key Format | Value Format | Windowed
----------------------------------------------------------------------------------------------------
 ORDERS_BY_STATE_PER_MINUTE    | orders-by-state-per-minute   | KAFKA      | AVRO         | true
----------------------------------------------------------------------------------------------------
```

Para la consulta pull:

```
+----------+--------------+-----------------+----------+
|STATE     |TOTAL_ORDERS  |TOTAL_VALUE_USD  |AVG_UNITS |
+----------+--------------+-----------------+----------+
|State_1   |12            |359.88           |4.5       |
Query terminated
```

**Verificación:**

- La tabla `ORDERS_BY_STATE_PER_MINUTE` aparece en `SHOW TABLES` con `Windowed = true`.
- El tópico `orders-by-state-per-minute` fue creado automáticamente en Kafka.
- Las consultas pull retornan resultados inmediatamente (sin `EMIT CHANGES`).
- Los valores de `TOTAL_VALUE_USD` son consistentes con `ORDERUNITS * 29.99`.

---

### Paso 6: Agregaciones por Categoría de Producto y Tabla de Resumen

**Objetivo:** Crear una tabla de agregación permanente por tipo de ítem para mantener conteos y totales actualizados en tiempo real, demostrando cómo ksqlDB mantiene estado materializado que puede ser consultado de forma puntual.

**Instrucciones:**

1. Crea una tabla que agregue las métricas por `ITEMID` y `ORDER_SIZE` de forma continua:

   ```sql
   CREATE TABLE ORDERS_SUMMARY_BY_ITEM
   WITH (
     KAFKA_TOPIC  = 'orders-summary-by-item',
     VALUE_FORMAT = 'AVRO',
     PARTITIONS   = 1
   ) AS
   SELECT
     ITEMID,
     ORDER_SIZE,
     COUNT(*)                            AS TOTAL_ORDERS,
     ROUND(SUM(ORDERUNITS), 2)           AS TOTAL_UNITS,
     ROUND(SUM(ESTIMATED_VALUE_USD), 2)  AS TOTAL_REVENUE_USD,
     ROUND(AVG(ESTIMATED_VALUE_USD), 2)  AS AVG_ORDER_VALUE_USD
   FROM ORDERS_ENRICHED
   GROUP BY ITEMID, ORDER_SIZE
   EMIT CHANGES;
   ```

2. Verifica el estado de todas las consultas persistentes activas:

   ```sql
   SHOW QUERIES;
   ```

3. Espera 30 segundos y luego consulta la tabla para ver el resumen actualizado de los primeros ítems:

   ```sql
   SELECT
     ITEMID,
     ORDER_SIZE,
     TOTAL_ORDERS,
     TOTAL_REVENUE_USD,
     AVG_ORDER_VALUE_USD
   FROM ORDERS_SUMMARY_BY_ITEM
   WHERE ITEMID = 'Item_1';
   ```

4. Consulta un segundo ítem para comparar:

   ```sql
   SELECT
     ITEMID,
     ORDER_SIZE,
     TOTAL_ORDERS,
     TOTAL_REVENUE_USD
   FROM ORDERS_SUMMARY_BY_ITEM
   WHERE ITEMID = 'Item_5';
   ```

5. Verifica los tópicos Kafka generados automáticamente por ksqlDB:

   ```bash
   # En una segunda terminal
   docker exec broker kafka-topics \
     --bootstrap-server broker:29092 \
     --list | grep -E "orders|ksql"
   ```

**Salida Esperada:**

Para `SHOW QUERIES` (verás múltiples consultas persistentes activas):

```
 Query ID                                    | Query Type | Status  | Sink Name                    | Sink Kafka Topic
----------------------------------------------------------------------------------------------------------------------
 CSAS_ORDERS_ENRICHED_0                      | PERSISTENT | RUNNING | ORDERS_ENRICHED              | orders-enriched
 CTAS_ORDERS_BY_STATE_PER_MINUTE_1           | PERSISTENT | RUNNING | ORDERS_BY_STATE_PER_MINUTE   | orders-by-state-per-minute
 CTAS_ORDERS_SUMMARY_BY_ITEM_2               | PERSISTENT | RUNNING | ORDERS_SUMMARY_BY_ITEM       | orders-summary-by-item
----------------------------------------------------------------------------------------------------------------------
```

Para la consulta pull por `ITEMID`:

```
+--------+------------+--------------+-------------------+---------------------+
|ITEMID  |ORDER_SIZE  |TOTAL_ORDERS  |TOTAL_REVENUE_USD  |AVG_ORDER_VALUE_USD  |
+--------+------------+--------------+-------------------+---------------------+
|Item_1  |SMALL       |8             |1679.44            |209.93               |
|Item_1  |MEDIUM      |11            |3298.9             |299.9                |
|Item_1  |LARGE       |6             |2099.3             |349.87               |
Query terminated
```

Para la lista de tópicos:

```
orders
orders-enriched
orders-by-state-per-minute
orders-summary-by-item
```

**Verificación:**

- `SHOW QUERIES` muestra tres consultas persistentes con estado `RUNNING`.
- Las consultas pull sobre `ORDERS_SUMMARY_BY_ITEM` retornan datos agrupados por `ITEMID` y `ORDER_SIZE`.
- Los tópicos `orders-enriched`, `orders-by-state-per-minute` y `orders-summary-by-item` existen en Kafka.
- Los valores numéricos son consistentes (sin valores nulos ni errores de tipo).

---

### Paso 7: Exploración desde Confluent Control Center

**Objetivo:** Usar la interfaz web de Confluent Control Center para visualizar y ejecutar consultas ksqlDB de forma gráfica, comprendiendo cómo la herramienta facilita la administración y el monitoreo de los flujos de procesamiento.

**Instrucciones:**

1. Abre un navegador web y navega a Confluent Control Center:

   ```
   http://localhost:9021
   ```

2. En el menú lateral, selecciona el clúster Kafka y navega a **ksqlDB** → **ksqlDB clusters**. Haz clic en el cluster `ksqldb-server`.

3. En la sección **Editor**, ejecuta la siguiente consulta para ver el resumen de streams activos:

   ```sql
   SHOW STREAMS;
   ```

4. Navega a la pestaña **Streams** en el panel de ksqlDB para ver visualmente los streams `ORDERS_STREAM` y `ORDERS_ENRICHED` con sus metadatos.

5. Haz clic en `ORDERS_ENRICHED` para ver el detalle del stream, incluyendo el esquema de columnas y el tópico Kafka subyacente.

6. En el **Editor** de Control Center, ejecuta una consulta push para observar el flujo en tiempo real desde la interfaz web:

   ```sql
   SELECT ORDERID, ITEMID, ORDER_SIZE, ESTIMATED_VALUE_USD
   FROM ORDERS_ENRICHED
   EMIT CHANGES
   LIMIT 20;
   ```

7. Navega a **Topics** en el menú principal y busca el tópico `orders-enriched`. Explora la pestaña **Messages** para ver los mensajes Avro deserializados visualmente.

8. Vuelve a la terminal con el ksqlDB CLI para verificar el estado de las queries desde la línea de comandos:

   ```sql
   DESCRIBE EXTENDED ORDERS_ENRICHED;
   ```

**Salida Esperada:**

En Control Center, la sección ksqlDB mostrará:

- Lista de streams: `ORDERS_STREAM`, `ORDERS_ENRICHED`
- Lista de tablas: `ORDERS_BY_STATE_PER_MINUTE`, `ORDERS_SUMMARY_BY_ITEM`
- Lista de queries activas con su estado `RUNNING`

Para `DESCRIBE EXTENDED ORDERS_ENRICHED` en el CLI:

```
Name                 : ORDERS_ENRICHED
Type                 : STREAM
Timestamp field      : Not set - using <ROWTIME>
Key format           : KAFKA
Value format         : AVRO
Kafka topic          : orders-enriched (partitions: 1, replication: 1)
Statement            : CREATE STREAM ORDERS_ENRICHED WITH (...)...

 Field                | Type
---------------------------------------------
 ORDERID              | BIGINT
 ITEMID               | VARCHAR(STRING)
 ORDERUNITS           | DOUBLE
 CITY                 | VARCHAR(STRING)
 STATE                | VARCHAR(STRING)
 ZIPCODE              | BIGINT
 ORDER_SIZE           | VARCHAR(STRING)
 ESTIMATED_VALUE_USD  | DOUBLE
 ORDER_TIMESTAMP      | VARCHAR(STRING)
---------------------------------------------

Queries that write from this STREAM
------------------------------------
CTAS_ORDERS_BY_STATE_PER_MINUTE_1 (RUNNING) : ...
CTAS_ORDERS_SUMMARY_BY_ITEM_2 (RUNNING) : ...

Local runtime statistics
------------------------
messages-per-sec:      1.23   last-message: 2024-01-15T10:30:45.123Z
```

**Verificación:**

- Control Center muestra los streams y tablas creados en los pasos anteriores.
- La consulta ejecutada desde el Editor de Control Center retorna resultados correctamente.
- El tópico `orders-enriched` en la sección Topics muestra mensajes con campos Avro deserializados.
- `DESCRIBE EXTENDED` muestra las queries que leen del stream.

---

### Paso 8: Construcción del Pipeline Completo y Validación End-to-End

**Objetivo:** Validar el pipeline completo de procesamiento de flujos construido durante el laboratorio, verificando que los datos fluyen correctamente desde el tópico fuente hasta los tópicos de destino, y crear un stream final de alertas para órdenes de alto valor.

**Instrucciones:**

1. En el ksqlDB CLI, crea un stream de alertas para órdenes de alto valor (más de 200 USD estimados) que fluya a un tópico dedicado:

   ```sql
   CREATE STREAM HIGH_VALUE_ORDERS_ALERTS
   WITH (
     KAFKA_TOPIC  = 'high-value-orders-alerts',
     VALUE_FORMAT = 'AVRO',
     PARTITIONS   = 1
   ) AS
   SELECT
     ORDERID,
     ITEMID,
     ORDERUNITS,
     ESTIMATED_VALUE_USD,
     CITY,
     STATE,
     ORDER_TIMESTAMP,
     'HIGH_VALUE_ALERT'  AS ALERT_TYPE
   FROM ORDERS_ENRICHED
   WHERE ESTIMATED_VALUE_USD > 200.0
   EMIT CHANGES;
   ```

2. Verifica el pipeline completo con `SHOW QUERIES`:

   ```sql
   SHOW QUERIES;
   ```

3. Prueba una consulta push sobre el stream de alertas:

   ```sql
   SELECT ORDERID, ITEMID, ESTIMATED_VALUE_USD, ALERT_TYPE
   FROM HIGH_VALUE_ORDERS_ALERTS
   EMIT CHANGES
   LIMIT 5;
   ```

4. Valida el pipeline completo desde la terminal (fuera del CLI), verificando que todos los tópicos Kafka de destino reciben mensajes:

   ```bash
   # En una segunda terminal - verificar mensajes en orders-enriched
   docker exec broker kafka-console-consumer \
     --bootstrap-server broker:29092 \
     --topic orders-enriched \
     --from-beginning \
     --max-messages 3 \
     --timeout-ms 10000
   ```

   ```bash
   # Verificar mensajes en high-value-orders-alerts
   docker exec broker kafka-console-consumer \
     --bootstrap-server broker:29092 \
     --topic high-value-orders-alerts \
     --from-beginning \
     --max-messages 3 \
     --timeout-ms 10000
   ```

5. Usa la API REST de ksqlDB para verificar el estado de todas las queries (desde fuera del CLI):

   ```bash
   # Listar todas las queries persistentes via REST API
   curl -s http://localhost:8088/ksql \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d '{"ksql": "SHOW QUERIES;", "streamsProperties": {}}' | jq '.[].queries[].queryString' 2>/dev/null || \
   curl -s http://localhost:8088/ksql \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d '{"ksql": "SHOW QUERIES;", "streamsProperties": {}}' | jq .
   ```

6. Finalmente, desde el ksqlDB CLI, muestra el resumen completo del entorno construido:

   ```sql
   SHOW STREAMS;
   SHOW TABLES;
   SHOW QUERIES;
   ```

**Salida Esperada:**

Para `SHOW STREAMS` al final del laboratorio:

```
 Stream Name                  | Kafka Topic              | Key Format | Value Format | Windowed
------------------------------------------------------------------------------------------------
 ORDERS_STREAM                | orders                   | KAFKA      | AVRO         | false
 ORDERS_ENRICHED              | orders-enriched          | KAFKA      | AVRO         | false
 HIGH_VALUE_ORDERS_ALERTS     | high-value-orders-alerts | KAFKA      | AVRO         | false
------------------------------------------------------------------------------------------------
```

Para `SHOW TABLES`:

```
 Table Name                    | Kafka Topic                  | Key Format | Value Format | Windowed
----------------------------------------------------------------------------------------------------
 ORDERS_BY_STATE_PER_MINUTE    | orders-by-state-per-minute   | KAFKA      | AVRO         | true
 ORDERS_SUMMARY_BY_ITEM        | orders-summary-by-item       | KAFKA      | AVRO         | false
----------------------------------------------------------------------------------------------------
```

Para `SHOW QUERIES` (4 queries persistentes activas):

```
 Query ID                                    | Query Type | Status  | Sink Name
------------------------------------------------------------------------------------
 CSAS_ORDERS_ENRICHED_0                      | PERSISTENT | RUNNING | ORDERS_ENRICHED
 CTAS_ORDERS_BY_STATE_PER_MINUTE_1           | PERSISTENT | RUNNING | ORDERS_BY_STATE_PER_MINUTE
 CTAS_ORDERS_SUMMARY_BY_ITEM_2               | PERSISTENT | RUNNING | ORDERS_SUMMARY_BY_ITEM
 CSAS_HIGH_VALUE_ORDERS_ALERTS_3             | PERSISTENT | RUNNING | HIGH_VALUE_ORDERS_ALERTS
------------------------------------------------------------------------------------
```

**Verificación:**

- Existen 3 streams y 2 tablas en ksqlDB.
- Las 4 consultas persistentes tienen estado `RUNNING`.
- Los 4 tópicos de destino (`orders-enriched`, `orders-by-state-per-minute`, `orders-summary-by-item`, `high-value-orders-alerts`) existen en Kafka y reciben mensajes.
- El stream `HIGH_VALUE_ORDERS_ALERTS` solo contiene órdenes con `ESTIMATED_VALUE_USD > 200.0`.

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El ksqlDB CLI se conecta exitosamente al servidor en `http://ksqldb-server:8088` y muestra `Server Status: RUNNING`.
- [ ] El STREAM `ORDERS_STREAM` fue creado sobre el tópico `orders` con el esquema correcto (5 campos incluyendo `ADDRESS` como STRUCT).
- [ ] Las consultas push con `EMIT CHANGES` emiten resultados continuos en tiempo real sin errores de deserialización.
- [ ] El STREAM derivado `ORDERS_ENRICHED` existe con los campos calculados `ORDER_SIZE`, `ESTIMATED_VALUE_USD` y `ORDER_TIMESTAMP`.
- [ ] La TABLE `ORDERS_BY_STATE_PER_MINUTE` usa ventana TUMBLING de 1 minuto y acepta consultas pull por `STATE`.
- [ ] La TABLE `ORDERS_SUMMARY_BY_ITEM` mantiene el estado actualizado de métricas por `ITEMID` y `ORDER_SIZE`.
- [ ] El STREAM `HIGH_VALUE_ORDERS_ALERTS` solo contiene órdenes con valor estimado mayor a 200 USD.
- [ ] Los 4 tópicos Kafka de destino existen y contienen mensajes verificables con `kafka-console-consumer`.
- [ ] Confluent Control Center muestra correctamente los streams, tablas y queries activas de ksqlDB.
- [ ] `SHOW QUERIES` lista exactamente 4 consultas persistentes con estado `RUNNING`.

### Procedimiento de Pruebas

1. Verifica la conectividad al servidor ksqlDB via REST API:

   ```bash
   curl -s http://localhost:8088/info | jq '{version: .KsqlServerInfo.version, status: .KsqlServerInfo.serverStatus}'
   ```

   **Resultado Esperado:**

   ```json
   {
     "version": "8.0.0-ce",
     "status": "RUNNING"
   }
   ```

2. Verifica que todos los tópicos de destino existen y tienen mensajes:

   ```bash
   for topic in orders-enriched orders-by-state-per-minute orders-summary-by-item high-value-orders-alerts; do
     count=$(docker exec broker kafka-run-class kafka.tools.GetOffsetShell \
       --broker-list broker:29092 \
       --topic $topic \
       --time -1 2>/dev/null | awk -F: '{sum += $3} END {print sum}')
     echo "Tópico: $topic | Mensajes aproximados: ${count:-0}"
   done
   ```

   **Resultado Esperado:**

   ```
   Tópico: orders-enriched | Mensajes aproximados: 150
   Tópico: orders-by-state-per-minute | Mensajes aproximados: 30
   Tópico: orders-summary-by-item | Mensajes aproximados: 45
   Tópico: high-value-orders-alerts | Mensajes aproximados: 50
   ```

3. Verifica que el filtro de alto valor funciona correctamente ejecutando una consulta pull de validación:

   ```bash
   curl -s -X POST http://localhost:8088/query \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d '{
       "ksql": "SELECT ORDERID, ESTIMATED_VALUE_USD FROM HIGH_VALUE_ORDERS_ALERTS LIMIT 3;",
       "streamsProperties": {"ksql.streams.auto.offset.reset": "earliest"}
     }' | jq '.[1:] | .[] | {orderid: .row.columns[0], value: .row.columns[1]}'
   ```

   **Resultado Esperado** (todos los valores deben ser > 200):

   ```json
   {"orderid": 2, "value": 209.93}
   {"orderid": 7, "value": 239.92}
   {"orderid": 12, "value": 269.91}
   ```

4. Verifica la tabla con ventana TUMBLING:

   ```bash
   curl -s -X POST http://localhost:8088/query \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d '{
       "ksql": "SELECT STATE, TOTAL_ORDERS, TOTAL_VALUE_USD FROM ORDERS_BY_STATE_PER_MINUTE WHERE STATE = '\''State_1'\'';",
       "streamsProperties": {}
     }' | jq .
   ```

   **Resultado Esperado:** Respuesta JSON con filas que contienen `STATE = "State_1"` y valores numéricos positivos.

## Solución de Problemas

### Problema 1: Error "Topic 'orders' does not exist" al crear ORDERS_STREAM

**Síntomas:**
- Al ejecutar `CREATE STREAM ORDERS_STREAM ...`, ksqlDB retorna un error indicando que el tópico `orders` no existe.
- `SHOW TOPICS` en ksqlDB no lista el tópico `orders`.

**Causa:**
El conector Datagen del laboratorio anterior no está activo o fue eliminado, por lo que el tópico `orders` nunca fue creado o no tiene mensajes. También puede ocurrir si el contenedor de Kafka fue reiniciado con `docker compose down -v` (que elimina volúmenes y tópicos).

**Solución:**

```bash
# Verificar que el tópico existe en Kafka
docker exec broker kafka-topics \
  --bootstrap-server broker:29092 \
  --list | grep orders

# Si no existe, recrear el tópico manualmente
docker exec broker kafka-topics \
  --bootstrap-server broker:29092 \
  --create \
  --topic orders \
  --partitions 1 \
  --replication-factor 1

# Recrear el conector Datagen para generar datos
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "datagen-orders",
    "config": {
      "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
      "kafka.topic": "orders",
      "quickstart": "orders",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter": "io.confluent.connect.avro.AvroConverter",
      "value.converter.schema.registry.url": "http://schema-registry:8081",
      "max.interval": 500,
      "iterations": 50000,
      "tasks.max": "1"
    }
  }'

# Esperar 10 segundos y verificar que hay mensajes
sleep 10
docker exec broker kafka-console-consumer \
  --bootstrap-server broker:29092 \
  --topic orders \
  --max-messages 3 \
  --timeout-ms 5000
```

---

### Problema 2: Error de deserialización Avro al ejecutar SELECT sobre ORDERS_STREAM

**Síntomas:**
- Las consultas `SELECT` sobre `ORDERS_STREAM` retornan filas con valores `null` en todos los campos.
- El log de ksqlDB muestra errores como `Could not deserialize Avro message`.
- `PRINT 'orders'` muestra mensajes pero con formato ilegible o binario.

**Causa:**
El Schema Registry no está accesible desde ksqlDB, o el esquema Avro registrado para el tópico `orders` no coincide con el esquema definido en el `CREATE STREAM`. Esto ocurre cuando el stream fue creado con `VALUE_FORMAT = 'AVRO'` pero el Schema Registry está caído o la URL es incorrecta.

**Solución:**

```bash
# Verificar que Schema Registry está disponible
curl -s http://localhost:8081/subjects | jq .

# Verificar que el esquema del tópico orders está registrado
curl -s http://localhost:8081/subjects/orders-value/versions/latest | jq .

# Verificar la configuración de ksqlDB (debe apuntar a Schema Registry)
curl -s http://localhost:8088/info | jq '.KsqlServerInfo'

# Si Schema Registry no responde, reiniciarlo
docker compose restart schema-registry

# Esperar 30 segundos y verificar nuevamente
sleep 30
curl -s http://localhost:8081/subjects | jq .

# Eliminar el stream con error y recrearlo
# (dentro del ksqlDB CLI)
# DROP STREAM IF EXISTS ORDERS_STREAM DELETE TOPIC;
# Luego recrear con el CREATE STREAM del Paso 2
```

---

### Problema 3: Las consultas persistentes aparecen en estado FAILED en SHOW QUERIES

**Síntomas:**
- `SHOW QUERIES` muestra una o más queries con estado `ERROR` o `FAILED`.
- Las tablas `ORDERS_BY_STATE_PER_MINUTE` o `ORDERS_SUMMARY_BY_ITEM` no se actualizan con nuevos datos.
- Los tópicos de destino no reciben mensajes nuevos.

**Causa:**
Las consultas persistentes pueden fallar por falta de memoria en el contenedor de ksqlDB (especialmente en máquinas con menos de 12 GB de RAM disponibles para Docker), por conflictos de nombres de tópicos ya existentes, o por incompatibilidades de esquema al intentar escribir en un tópico con un esquema Avro diferente.

**Solución:**

```bash
# Ver el error detallado de la query fallida
# (dentro del ksqlDB CLI)
# EXPLAIN <QUERY_ID>;

# Verificar el uso de memoria del contenedor ksqlDB
docker stats ksqldb-server --no-stream --format "table {{.Container}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Si hay problemas de memoria, aumentar el límite en docker-compose.yml:
# mem_limit: 2g
# environment:
#   KSQL_HEAP_OPTS: "-Xms512m -Xmx1536m"

# Reiniciar ksqlDB Server después de ajustar memoria
docker compose restart ksqldb-server
sleep 60

# Eliminar queries fallidas y recrear los objetos
# (dentro del ksqlDB CLI)
# TERMINATE <QUERY_ID>;
# DROP TABLE IF EXISTS ORDERS_BY_STATE_PER_MINUTE DELETE TOPIC;
# Luego recrear con el CREATE TABLE del Paso 5

# Verificar tópicos con nombres conflictivos
docker exec broker kafka-topics \
  --bootstrap-server broker:29092 \
  --list | grep orders
```

---

### Problema 4: ksqlDB CLI no puede conectarse al servidor

**Síntomas:**
- El comando `docker exec -it ksqldb-cli ksql http://ksqldb-server:8088` falla con `Connection refused` o `Could not connect to the ksqlDB server`.
- `curl http://localhost:8088/info` retorna error de conexión.

**Causa:**
El contenedor `ksqldb-server` no terminó de inicializar (puede tardar entre 30-60 segundos), está en estado de error, o el puerto 8088 no está mapeado correctamente en el `docker-compose.yml`.

**Solución:**

```bash
# Verificar el estado del contenedor ksqldb-server
docker compose ps ksqldb-server

# Ver los logs de inicialización
docker logs ksqldb-server --tail 50

# Esperar a que el servidor esté listo (buscar el mensaje "Server up and running")
docker logs -f ksqldb-server 2>&1 | grep -m 1 "Server up and running"

# Si el contenedor está en estado "Exiting", reiniciarlo
docker compose restart ksqldb-server

# Esperar 60 segundos y verificar
sleep 60
curl -s http://localhost:8088/info | jq '.KsqlServerInfo.serverStatus'

# Verificar que el puerto 8088 está mapeado
docker compose port ksqldb-server 8088
```

## Limpieza

Ejecuta los siguientes comandos para limpiar los recursos creados durante el laboratorio. **Elige la opción según si necesitas conservar los datos para laboratorios posteriores.**

```bash
# OPCIÓN A: Limpieza parcial - solo eliminar objetos ksqlDB (conserva datos en Kafka)
# Ejecutar dentro del ksqlDB CLI:

# Terminar todas las queries persistentes
# TERMINATE CSAS_ORDERS_ENRICHED_0;
# TERMINATE CTAS_ORDERS_BY_STATE_PER_MINUTE_1;
# TERMINATE CTAS_ORDERS_SUMMARY_BY_ITEM_2;
# TERMINATE CSAS_HIGH_VALUE_ORDERS_ALERTS_3;

# Eliminar tablas y streams (con DELETE TOPIC para limpiar también los tópicos Kafka)
# DROP TABLE IF EXISTS ORDERS_SUMMARY_BY_ITEM DELETE TOPIC;
# DROP TABLE IF EXISTS ORDERS_BY_STATE_PER_MINUTE DELETE TOPIC;
# DROP STREAM IF EXISTS HIGH_VALUE_ORDERS_ALERTS DELETE TOPIC;
# DROP STREAM IF EXISTS ORDERS_ENRICHED DELETE TOPIC;
# DROP STREAM IF EXISTS ORDERS_STREAM;

# OPCIÓN B: Detener el entorno completo SIN eliminar volúmenes (preserva datos)
docker compose down

# OPCIÓN C: Limpieza completa - eliminar todos los contenedores Y volúmenes
# ⚠️ ADVERTENCIA: Esto elimina TODOS los datos de Kafka permanentemente
# docker compose down -v
```

> ⚠️ **Advertencia:** Usa `docker compose down -v` **únicamente** si deseas reiniciar el entorno desde cero. Este comando elimina todos los volúmenes Docker nombrados, incluyendo los datos de Kafka, PostgreSQL y los tópicos internos de Kafka Connect. Los laboratorios 5 y 6 dependen de datos generados en sesiones anteriores. Si planeas continuar con el Laboratorio 7, usa `docker compose down` (sin `-v`) para preservar los volúmenes.

```bash
# Verificar que los contenedores fueron detenidos correctamente
docker compose ps

# Si necesitas limpiar solo los tópicos creados en este laboratorio sin bajar el entorno:
for topic in orders-enriched orders-by-state-per-minute orders-summary-by-item high-value-orders-alerts; do
  docker exec broker kafka-topics \
    --bootstrap-server broker:29092 \
    --delete \
    --topic $topic 2>/dev/null && echo "Tópico $topic eliminado" || echo "Tópico $topic no encontrado"
done
```

## Resumen

### Lo Que Lograste

- Conectaste el ksqlDB CLI al servidor y exploraste el entorno usando comandos `SHOW TOPICS`, `SHOW STREAMS`, `SHOW TABLES` y `SHOW QUERIES`.
- Creaste el STREAM `ORDERS_STREAM` sobre el tópico Kafka `orders` con esquema explícito incluyendo campos de tipo STRUCT anidado.
- Ejecutaste consultas push con `EMIT CHANGES` para observar eventos en tiempo real y consultas con filtros sobre campos simples y anidados.
- Construiste el STREAM derivado `ORDERS_ENRICHED` usando `CREATE STREAM AS SELECT` con transformaciones, clasificación condicional (`CASE WHEN`) y cálculos aritméticos.
- Creaste la TABLE `ORDERS_BY_STATE_PER_MINUTE` con ventanas temporales TUMBLING de 1 minuto para agregaciones por estado geográfico.
- Creaste la TABLE `ORDERS_SUMMARY_BY_ITEM` para mantener métricas actualizadas por ítem y categoría de tamaño.
- Construiste el STREAM `HIGH_VALUE_ORDERS_ALERTS` como pipeline de alertas para órdenes con valor estimado mayor a 200 USD.
- Verificaste el pipeline completo desde Confluent Control Center y la API REST de ksqlDB.

### Conceptos Clave Aprendidos

- **STREAM vs TABLE en ksqlDB**: Un STREAM representa una secuencia inmutable de eventos ordenados en el tiempo; una TABLE representa el estado más reciente para cada clave, actualizado continuamente por los eventos del stream.
- **Consultas Push vs Pull**: Las consultas push (`EMIT CHANGES`) son continuas y retornan resultados a medida que llegan nuevos eventos; las consultas pull son puntuales y retornan el estado actual de una tabla materializada.
- **CREATE STREAM AS SELECT (CSAS)**: Crea un nuevo stream derivado y una consulta persistente que procesa continuamente los eventos del stream fuente, generando automáticamente un nuevo tópico Kafka de destino.
- **CREATE TABLE AS SELECT (CTAS)**: Crea una tabla materializada con estado actualizado mediante una consulta persistente de agregación sobre un stream.
- **Ventanas Temporales TUMBLING**: Dividen el tiempo en intervalos fijos y no superpuestos (ej: cada minuto), permitiendo calcular métricas como conteos y sumas dentro de cada ventana de tiempo.
- **Consultas Persistentes**: Las queries creadas con `CREATE STREAM AS SELECT` y `CREATE TABLE AS SELECT` se ejecutan continuamente en el servidor ksqlDB y sobreviven a la desconexión del cliente CLI.
- **Pipeline Declarativo**: ksqlDB permite construir pipelines de procesamiento complejos usando únicamente SQL, sin necesidad de código Java o Python para las transformaciones más comunes.

### Próximos Pasos

- Explorar las ventanas temporales **HOPPING** (superpuestas) y **SESSION** (basadas en actividad) para casos de uso más avanzados como detección de sesiones de usuario.
- Implementar **JOINs entre streams y tablas** en ksqlDB para enriquecer eventos en tiempo real con datos de referencia (ej: enriquecer órdenes con información del catálogo de productos).
- Estudiar las capacidades de **Kafka Connect integrado en ksqlDB** para crear fuentes y sinks directamente desde sentencias SQL usando `CREATE SOURCE CONNECTOR` y `CREATE SINK CONNECTOR`.
- Revisar las métricas de rendimiento de las consultas persistentes en Confluent Control Center para identificar cuellos de botella y optimizar el paralelismo mediante `tasks.max`.

## Recursos Adicionales

- **Documentación oficial de ksqlDB** (https://docs.ksqldb.io) - Referencia completa de la sintaxis SQL, tipos de datos, funciones escalares y de agregación disponibles en ksqlDB.
- **ksqlDB Tutorials** (https://docs.ksqldb.io/en/latest/tutorials/) - Tutoriales oficiales de Confluent con casos de uso reales: detección de fraude, análisis de logs, pipelines de IoT.
- **Referencia de Ventanas Temporales en ksqlDB** (https://docs.ksqldb.io/en/latest/concepts/time-and-windows-in-ksqldb-queries/) - Explicación detallada de los tipos de ventanas TUMBLING, HOPPING y SESSION con ejemplos.
- **ksqlDB REST API Reference** (https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-rest-api/) - Documentación completa de todos los endpoints REST para integrar ksqlDB con aplicaciones externas.
- **Confluent Developer: ksqlDB Recipes** (https://developer.confluent.io/tutorials/#use-case=stream-processing) - Recetas prácticas de procesamiento de flujos organizadas por caso de uso, con código listo para usar.

---

# Laboratorio 7: Evaluación Final e Integración del Curso

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 50 minutos |
| **Complejidad** | Avanzado |
| **Nivel Bloom** | Crear |
| **Tecnologías** | Apache Kafka 4.0 (KRaft), Confluent Platform 8.0, Kafka Connect (JDBC Source + Sink), Schema Registry, ksqlDB, Confluent Control Center, Prometheus + Grafana, PostgreSQL 15, Docker Compose |

---

## Descripción General

Este laboratorio es la evaluación integradora final del curso. Los participantes recibirán un escenario de negocio complejo — un sistema de e-commerce con eventos de órdenes, inventario y notificaciones — y deberán diseñar e implementar de forma autónoma una solución Kafka completa que integre todos los componentes estudiados. La actividad demuestra dominio real del ecosistema Kafka al construir un pipeline end-to-end funcional que va desde la ingestión de datos en PostgreSQL hasta el procesamiento en tiempo real con ksqlDB y la visualización de métricas en Grafana.

Este ejercicio refleja directamente el trabajo que realizan los ingenieros de datos y arquitectos de plataformas en entornos de producción reales, donde la capacidad de integrar múltiples componentes y tomar decisiones de diseño justificadas es más valiosa que el conocimiento aislado de cada herramienta.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Diseñar una arquitectura Kafka completa para un caso de uso de negocio real, documentando decisiones sobre tópicos, particiones, retención y conectores.
- [ ] Implementar de forma autónoma un pipeline end-to-end que integre PostgreSQL, Kafka Connect JDBC Source/Sink, Schema Registry y ksqlDB.
- [ ] Aplicar buenas prácticas de producción incluyendo naming conventions, configuración de replicación, Dead Letter Queue y monitoreo con Prometheus y Grafana.
- [ ] Presentar y defender las decisiones de diseño arquitectónico tomadas durante la implementación del proyecto final.

---

## Prerrequisitos

### Conocimiento Requerido

- Haber completado todos los laboratorios anteriores del curso (Laboratorios 1 al 6).
- Dominio de la configuración de Kafka Connect con conectores JDBC Source y Sink.
- Capacidad de crear y ejecutar consultas ksqlDB para procesamiento de flujos en tiempo real.
- Familiaridad con Prometheus y Grafana para monitoreo básico del clúster Kafka.
- Comprensión de buenas prácticas de producción en Kafka: replicación, retención, naming conventions y seguridad básica.
- Manejo de la API REST de Kafka Connect para registrar y monitorear conectores.

### Acceso Requerido

- Docker Engine 24.0+ y Docker Compose 2.20+ instalados y funcionando.
- Al menos 12 GB de RAM disponibles para Docker (16 GB recomendados).
- Al menos 5 GB de espacio en disco libre para imágenes y volúmenes.
- Acceso a internet para descargar imágenes Docker si no están en caché local.
- Permisos de administrador (sudo en Linux/macOS) para ejecutar contenedores Docker.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | 4 núcleos físicos (64-bit, virtualización habilitada) |
| RAM | Mínimo 12 GB disponibles para Docker; 16 GB recomendados |
| Disco | 5 GB libres para imágenes, volúmenes y logs |
| Red | Conexión a internet de al menos 10 Mbps |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Docker Engine | 24.0+ | Motor de contenedores principal |
| Docker Compose | 2.20+ (plugin v2) | Orquestación del ecosistema completo |
| Confluent Platform | 8.0 | Kafka, Schema Registry, Connect, ksqlDB, Control Center |
| PostgreSQL | 15 | Base de datos fuente y destino del pipeline |
| Prometheus | 2.45+ | Recolección de métricas del clúster |
| Grafana | 10.0+ | Visualización de dashboards de monitoreo |
| curl | 7.x+ | Interacción con APIs REST |
| jq | 1.6+ | Procesamiento de respuestas JSON |

### Configuración Inicial

Crea el directorio de trabajo del laboratorio final y la estructura de archivos necesaria:

```bash
mkdir -p ~/kafka-final-lab/{config,connectors,prometheus,grafana/provisioning/{datasources,dashboards},sql,scripts}
cd ~/kafka-final-lab
```

Verifica que Docker y Docker Compose estén disponibles y con las versiones correctas:

```bash
docker --version
docker compose version
docker info | grep -E "Total Memory|CPUs"
```

---

## Instrucciones Paso a Paso

### Paso 1: Diseño de la Arquitectura (Fase 1 — 15 minutos)

**Objetivo:** Definir la arquitectura completa del sistema de e-commerce antes de comenzar la implementación, documentando cada decisión de diseño.

**Escenario de Negocio:**

Eres el arquitecto de datos de **ShopStream**, una plataforma de e-commerce en crecimiento. El equipo de ingeniería necesita una plataforma de eventos en tiempo real que:

1. Capture automáticamente todos los pedidos nuevos y cambios de inventario desde la base de datos PostgreSQL existente.
2. Procese los eventos en tiempo real para detectar pedidos de alto valor (> $500) y generar alertas.
3. Mantenga una vista agregada del inventario actual por categoría de producto.
4. Exporte las alertas de pedidos críticos de vuelta a una tabla PostgreSQL para el equipo de operaciones.
5. Proporcione visibilidad operacional mediante dashboards de monitoreo.

**Instrucciones:**

1. Documenta la arquitectura en un archivo de diseño. Crea el archivo de diseño:

   ```bash
   cat > ~/kafka-final-lab/ARQUITECTURA.md << 'EOF'
   # ShopStream - Arquitectura Kafka

   ## Escenario de Negocio
   Sistema de e-commerce con procesamiento de eventos en tiempo real.

   ## Tópicos Kafka

   | Tópico | Particiones | Replicación | Retención | Descripción |
   |--------|-------------|-------------|-----------|-------------|
   | shopstream.orders.raw | 3 | 1 | 7 días | Pedidos capturados desde PostgreSQL via JDBC Source |
   | shopstream.inventory.raw | 3 | 1 | 7 días | Cambios de inventario desde PostgreSQL |
   | shopstream.orders.highvalue | 1 | 1 | 30 días | Pedidos con valor > $500 (generado por ksqlDB) |
   | shopstream.alerts.operations | 1 | 1 | 30 días | Alertas exportadas a PostgreSQL via JDBC Sink |
   | shopstream.connect.dlq | 1 | 1 | 3 días | Dead Letter Queue para errores de conectores |

   ## Componentes

   - **JDBC Source Connector**: Lee tablas `orders` e `inventory` de PostgreSQL
   - **ksqlDB Stream**: Filtra pedidos de alto valor en tiempo real
   - **ksqlDB Table**: Agrega inventario por categoría
   - **JDBC Sink Connector**: Escribe alertas en tabla `critical_orders` de PostgreSQL
   - **Prometheus + Grafana**: Monitoreo de métricas del broker y conectores

   ## Decisiones de Diseño

   1. **3 particiones** en tópicos de ingestión para permitir paralelismo futuro
   2. **Retención diferenciada**: datos raw 7 días, alertas 30 días (requisito de auditoría)
   3. **DLQ habilitado** en conectores para tolerancia a errores sin detener el pipeline
   4. **Naming convention**: `{dominio}.{entidad}.{estado}` (ej: shopstream.orders.raw)
   5. **Schema Registry con Avro** para garantizar compatibilidad de esquemas

   EOF
   echo "Archivo de diseño creado."
   ```

2. Revisa y personaliza el archivo de diseño si deseas ajustar alguna decisión:

   ```bash
   cat ~/kafka-final-lab/ARQUITECTURA.md
   ```

**Salida Esperada:**

```
Archivo de diseño creado.
# ShopStream - Arquitectura Kafka
...
[Contenido completo del archivo de arquitectura]
```

**Verificación:**

- El archivo `ARQUITECTURA.md` existe y contiene la definición de al menos 5 tópicos.
- Cada tópico tiene justificación de particiones y retención.

---

### Paso 2: Creación del Docker Compose Completo (Fase 2 — Implementación)

**Objetivo:** Construir el archivo `docker-compose.yml` que levanta el ecosistema completo de ShopStream incluyendo todos los componentes necesarios.

**Instrucciones:**

1. Crea el archivo de configuración JMX Exporter para exponer métricas de Kafka:

   ```bash
   cat > ~/kafka-final-lab/config/jmx-exporter.yml << 'EOF'
   startDelaySeconds: 0
   ssl: false
   lowercaseOutputName: false
   lowercaseOutputLabelNames: false
   rules:
   - pattern: "kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value"
     name: "kafka_server_$1_$2"
     type: GAUGE
     labels:
       clientId: "$3"
       topic: "$4"
       partition: "$5"
   - pattern: "kafka.server<type=(.+), name=(.+), clientId=(.+), brokerHost=(.+), brokerPort=(.+)><>Value"
     name: "kafka_server_$1_$2"
     type: GAUGE
     labels:
       clientId: "$3"
       broker: "$4:$5"
   - pattern: "kafka.server<type=(.+), name=(.+)><>OneMinuteRate"
     name: "kafka_server_$1_$2_rate1m"
     type: GAUGE
   - pattern: "kafka.server<type=(.+), name=(.+)><>Value"
     name: "kafka_server_$1_$2"
     type: GAUGE
   - pattern: "kafka.network<type=(.+), name=(.+)><>OneMinuteRate"
     name: "kafka_network_$1_$2_rate1m"
     type: GAUGE
   - pattern: "kafka.network<type=(.+), name=(.+)><>Value"
     name: "kafka_network_$1_$2"
     type: GAUGE
   - pattern: "kafka.controller<type=(.+), name=(.+)><>Value"
     name: "kafka_controller_$1_$2"
     type: GAUGE
   EOF
   echo "JMX Exporter config creado."
   ```

2. Crea la configuración de Prometheus:

   ```bash
   cat > ~/kafka-final-lab/prometheus/prometheus.yml << 'EOF'
   global:
     scrape_interval: 15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: 'kafka-broker'
       static_configs:
         - targets: ['broker:9101']
       metrics_path: /metrics

     - job_name: 'kafka-connect'
       static_configs:
         - targets: ['kafka-connect:9102']
       metrics_path: /metrics

     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']
   EOF
   echo "Prometheus config creado."
   ```

3. Crea el datasource de Grafana para Prometheus:

   ```bash
   cat > ~/kafka-final-lab/grafana/provisioning/datasources/prometheus.yml << 'EOF'
   apiVersion: 1
   datasources:
     - name: Prometheus
       type: prometheus
       access: proxy
       url: http://prometheus:9090
       isDefault: true
       editable: true
   EOF
   echo "Grafana datasource creado."
   ```

4. Crea el archivo `docker-compose.yml` completo del ecosistema ShopStream:

   ```bash
   cat > ~/kafka-final-lab/docker-compose.yml << 'EOF'
   version: '3.8'

   networks:
     shopstream-net:
       driver: bridge

   volumes:
     postgres-data:
     kafka-data:
     prometheus-data:
     grafana-data:

   services:

     # ─── PostgreSQL ───────────────────────────────────────────────────
     postgres:
       image: postgres:15
       hostname: postgres
       container_name: shopstream-postgres
       networks:
         - shopstream-net
       ports:
         - "5432:5432"
       environment:
         POSTGRES_USER: shopstream
         POSTGRES_PASSWORD: shopstream123
         POSTGRES_DB: shopstream_db
       volumes:
         - postgres-data:/var/lib/postgresql/data
         - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U shopstream -d shopstream_db"]
         interval: 10s
         timeout: 5s
         retries: 5

     # ─── Kafka Broker (KRaft mode) ────────────────────────────────────
     broker:
       image: confluentinc/cp-kafka:8.0.0
       hostname: broker
       container_name: shopstream-broker
       networks:
         - shopstream-net
       ports:
         - "9092:9092"
         - "9101:9101"
       environment:
         KAFKA_NODE_ID: 1
         KAFKA_PROCESS_ROLES: broker,controller
         KAFKA_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://0.0.0.0:9092,CONTROLLER://broker:29093
         KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
         KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT,CONTROLLER:PLAINTEXT
         KAFKA_CONTROLLER_QUORUM_VOTERS: 1@broker:29093
         KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
         KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
         KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
         KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
         KAFKA_DEFAULT_REPLICATION_FACTOR: 1
         KAFKA_NUM_PARTITIONS: 3
         KAFKA_LOG_RETENTION_HOURS: 168
         KAFKA_LOG_SEGMENT_BYTES: 1073741824
         KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000
         KAFKA_JMX_PORT: 9101
         KAFKA_JMX_HOSTNAME: broker
         CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
         KAFKA_OPTS: "-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=9101:/opt/jmx-exporter/jmx-exporter.yml"
       volumes:
         - kafka-data:/var/lib/kafka/data
         - ./config/jmx-exporter.yml:/opt/jmx-exporter/jmx-exporter.yml
       healthcheck:
         test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
         interval: 15s
         timeout: 10s
         retries: 10

     # ─── Schema Registry ──────────────────────────────────────────────
     schema-registry:
       image: confluentinc/cp-schema-registry:8.0.0
       hostname: schema-registry
       container_name: shopstream-schema-registry
       networks:
         - shopstream-net
       depends_on:
         broker:
           condition: service_healthy
       ports:
         - "8081:8081"
       environment:
         SCHEMA_REGISTRY_HOST_NAME: schema-registry
         SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: broker:29092
         SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
         SCHEMA_REGISTRY_KAFKASTORE_TOPIC: _schemas
         SCHEMA_REGISTRY_SCHEMA_COMPATIBILITY_LEVEL: BACKWARD

     # ─── Kafka Connect ────────────────────────────────────────────────
     kafka-connect:
       image: confluentinc/cp-kafka-connect:8.0.0
       hostname: kafka-connect
       container_name: shopstream-kafka-connect
       networks:
         - shopstream-net
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_started
         postgres:
           condition: service_healthy
       ports:
         - "8083:8083"
         - "9102:9102"
       environment:
         CONNECT_BOOTSTRAP_SERVERS: broker:29092
         CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
         CONNECT_REST_PORT: 8083
         CONNECT_GROUP_ID: shopstream-connect-group
         CONNECT_CONFIG_STORAGE_TOPIC: shopstream.connect.configs
         CONNECT_OFFSET_STORAGE_TOPIC: shopstream.connect.offsets
         CONNECT_STATUS_STORAGE_TOPIC: shopstream.connect.status
         CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
         CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
         CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
         CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
         CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
         CONNECT_LOG4J_ROOT_LOGLEVEL: INFO
         KAFKA_OPTS: "-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=9102:/opt/jmx-exporter/jmx-exporter.yml"
       volumes:
         - ./config/jmx-exporter.yml:/opt/jmx-exporter/jmx-exporter.yml
       command:
         - bash
         - -c
         - |
           echo "Instalando conector JDBC..."
           confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.7.4
           echo "Instalando conector Datagen..."
           confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.6.3
           echo "Iniciando Kafka Connect..."
           /etc/confluent/docker/run

     # ─── ksqlDB Server ────────────────────────────────────────────────
     ksqldb-server:
       image: confluentinc/cp-ksqldb-server:8.0.0
       hostname: ksqldb-server
       container_name: shopstream-ksqldb-server
       networks:
         - shopstream-net
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_started
       ports:
         - "8088:8088"
       environment:
         KSQL_KSQL_SERVICE_ID: shopstream_ksqldb
         KSQL_BOOTSTRAP_SERVERS: broker:29092
         KSQL_LISTENERS: http://0.0.0.0:8088
         KSQL_KSQL_SCHEMA_REGISTRY_URL: http://schema-registry:8081
         KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
         KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"
         KSQL_KSQL_STREAMS_AUTO_OFFSET_RESET: earliest

     # ─── ksqlDB CLI ───────────────────────────────────────────────────
     ksqldb-cli:
       image: confluentinc/cp-ksqldb-cli:8.0.0
       container_name: shopstream-ksqldb-cli
       networks:
         - shopstream-net
       depends_on:
         - ksqldb-server
       entrypoint: /bin/sh
       tty: true

     # ─── Control Center ───────────────────────────────────────────────
     control-center:
       image: confluentinc/cp-enterprise-control-center:8.0.0
       hostname: control-center
       container_name: shopstream-control-center
       networks:
         - shopstream-net
       depends_on:
         broker:
           condition: service_healthy
         schema-registry:
           condition: service_started
         kafka-connect:
           condition: service_started
         ksqldb-server:
           condition: service_started
       ports:
         - "9021:9021"
       environment:
         CONTROL_CENTER_BOOTSTRAP_SERVERS: broker:29092
         CONTROL_CENTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
         CONTROL_CENTER_CONNECT_SHOPSTREAM_CLUSTER: http://kafka-connect:8083
         CONTROL_CENTER_KSQL_SHOPSTREAM_URL: http://ksqldb-server:8088
         CONTROL_CENTER_KSQL_SHOPSTREAM_ADVERTISED_URL: http://localhost:8088
         CONTROL_CENTER_REPLICATION_FACTOR: 1
         CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
         CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
         CONFLUENT_METRICS_TOPIC_REPLICATION: 1
         PORT: 9021

     # ─── Prometheus ───────────────────────────────────────────────────
     prometheus:
       image: prom/prometheus:v2.45.0
       hostname: prometheus
       container_name: shopstream-prometheus
       networks:
         - shopstream-net
       ports:
         - "9090:9090"
       volumes:
         - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
         - prometheus-data:/prometheus
       command:
         - '--config.file=/etc/prometheus/prometheus.yml'
         - '--storage.tsdb.path=/prometheus'
         - '--web.console.libraries=/etc/prometheus/console_libraries'
         - '--web.console.templates=/etc/prometheus/consoles'
         - '--storage.tsdb.retention.time=7d'

     # ─── Grafana ──────────────────────────────────────────────────────
     grafana:
       image: grafana/grafana:10.0.0
       hostname: grafana
       container_name: shopstream-grafana
       networks:
         - shopstream-net
       depends_on:
         - prometheus
       ports:
         - "3000:3000"
       environment:
         GF_SECURITY_ADMIN_USER: admin
         GF_SECURITY_ADMIN_PASSWORD: shopstream123
         GF_USERS_ALLOW_SIGN_UP: "false"
       volumes:
         - grafana-data:/var/lib/grafana
         - ./grafana/provisioning:/etc/grafana/provisioning
   EOF
   echo "docker-compose.yml creado exitosamente."
   ```

**Salida Esperada:**

```
JMX Exporter config creado.
Prometheus config creado.
Grafana datasource creado.
docker-compose.yml creado exitosamente.
```

**Verificación:**

- Confirma que todos los archivos de configuración existen:

  ```bash
  find ~/kafka-final-lab -type f | sort
  ```

- Verifica la sintaxis del docker-compose.yml:

  ```bash
  cd ~/kafka-final-lab && docker compose config --quiet && echo "Sintaxis válida"
  ```

---

### Paso 3: Inicialización de la Base de Datos PostgreSQL

**Objetivo:** Crear el esquema de la base de datos de ShopStream con las tablas de órdenes, inventario y alertas que serán integradas con Kafka.

**Instrucciones:**

1. Crea el script SQL de inicialización de la base de datos:

   ```bash
   cat > ~/kafka-final-lab/sql/init.sql << 'EOF'
   -- ============================================================
   -- ShopStream Database Schema
   -- ============================================================

   -- Tabla de productos
   CREATE TABLE IF NOT EXISTS products (
       product_id    SERIAL PRIMARY KEY,
       name          VARCHAR(200) NOT NULL,
       category      VARCHAR(100) NOT NULL,
       price         DECIMAL(10,2) NOT NULL,
       created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- Tabla de órdenes (fuente para JDBC Source Connector)
   CREATE TABLE IF NOT EXISTS orders (
       order_id      SERIAL PRIMARY KEY,
       customer_id   INTEGER NOT NULL,
       product_id    INTEGER NOT NULL,
       quantity      INTEGER NOT NULL DEFAULT 1,
       total_amount  DECIMAL(10,2) NOT NULL,
       status        VARCHAR(50) DEFAULT 'PENDING',
       created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- Tabla de inventario (fuente para JDBC Source Connector)
   CREATE TABLE IF NOT EXISTS inventory (
       inventory_id  SERIAL PRIMARY KEY,
       product_id    INTEGER NOT NULL,
       category      VARCHAR(100) NOT NULL,
       stock_level   INTEGER NOT NULL DEFAULT 0,
       warehouse     VARCHAR(100) DEFAULT 'MAIN',
       updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- Tabla de destino para alertas de pedidos críticos (JDBC Sink Connector)
   CREATE TABLE IF NOT EXISTS critical_orders (
       order_id      INTEGER PRIMARY KEY,
       customer_id   INTEGER NOT NULL,
       total_amount  DECIMAL(10,2) NOT NULL,
       alert_reason  VARCHAR(200),
       processed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- ─── Datos de prueba: Productos ───────────────────────────────
   INSERT INTO products (name, category, price) VALUES
       ('Laptop Pro 15"', 'Electronics', 1299.99),
       ('Wireless Headphones', 'Electronics', 199.99),
       ('Running Shoes', 'Sports', 89.99),
       ('Coffee Maker Deluxe', 'Appliances', 149.99),
       ('Programming Book', 'Books', 49.99),
       ('Gaming Chair', 'Furniture', 399.99),
       ('Smart Watch', 'Electronics', 299.99),
       ('Yoga Mat', 'Sports', 35.99);

   -- ─── Datos de prueba: Inventario ─────────────────────────────
   INSERT INTO inventory (product_id, category, stock_level, warehouse) VALUES
       (1, 'Electronics', 50, 'MAIN'),
       (2, 'Electronics', 120, 'MAIN'),
       (3, 'Sports', 200, 'WAREHOUSE-A'),
       (4, 'Appliances', 75, 'MAIN'),
       (5, 'Books', 500, 'WAREHOUSE-B'),
       (6, 'Furniture', 30, 'WAREHOUSE-A'),
       (7, 'Electronics', 80, 'MAIN'),
       (8, 'Sports', 300, 'WAREHOUSE-B');

   -- ─── Datos de prueba: Órdenes iniciales ──────────────────────
   INSERT INTO orders (customer_id, product_id, quantity, total_amount, status) VALUES
       (1001, 1, 1, 1299.99, 'COMPLETED'),
       (1002, 3, 2, 179.98, 'PENDING'),
       (1003, 7, 1, 299.99, 'PROCESSING'),
       (1004, 6, 2, 799.98, 'COMPLETED'),
       (1005, 2, 1, 199.99, 'PENDING'),
       (1006, 1, 2, 2599.98, 'COMPLETED'),
       (1007, 4, 3, 449.97, 'PROCESSING'),
       (1008, 5, 5, 249.95, 'COMPLETED');

   -- ─── Función para actualizar updated_at automáticamente ──────
   CREATE OR REPLACE FUNCTION update_updated_at_column()
   RETURNS TRIGGER AS $$
   BEGIN
       NEW.updated_at = CURRENT_TIMESTAMP;
       RETURN NEW;
   END;
   $$ language 'plpgsql';

   CREATE TRIGGER update_orders_updated_at
       BEFORE UPDATE ON orders
       FOR EACH ROW
       EXECUTE FUNCTION update_updated_at_column();

   CREATE TRIGGER update_inventory_updated_at
       BEFORE UPDATE ON inventory
       FOR EACH ROW
       EXECUTE FUNCTION update_updated_at_column();

   -- ─── Permisos para usuario shopstream ────────────────────────
   GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO shopstream;
   GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO shopstream;

   SELECT 'ShopStream database initialized successfully' AS status;
   EOF
   echo "Script SQL de inicialización creado."
   ```

**Salida Esperada:**

```
Script SQL de inicialización creado.
```

**Verificación:**

- Confirma que el archivo SQL tiene contenido válido:

  ```bash
  wc -l ~/kafka-final-lab/sql/init.sql
  grep -c "CREATE TABLE" ~/kafka-final-lab/sql/init.sql
  ```

  Deberías ver al menos 4 tablas definidas.

---

### Paso 4: Levantamiento del Ecosistema Completo

**Objetivo:** Iniciar todos los servicios del ecosistema ShopStream y verificar que cada componente está saludable antes de continuar con la implementación del pipeline.

**Instrucciones:**

1. Levanta el ecosistema completo. Este proceso puede tomar 3-5 minutos la primera vez:

   ```bash
   cd ~/kafka-final-lab
   docker compose up -d
   ```

2. Monitorea el proceso de arranque de los servicios críticos:

   ```bash
   watch -n 5 'docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"'
   ```

   Presiona `Ctrl+C` cuando todos los servicios muestren estado `running` o `healthy`.

3. Verifica específicamente el estado de salud de los servicios con healthcheck:

   ```bash
   docker compose ps | grep -E "healthy|unhealthy|starting"
   ```

4. Espera a que Kafka Connect esté completamente listo (puede tardar 3-4 minutos adicionales mientras instala los plugins JDBC y Datagen):

   ```bash
   echo "Esperando a que Kafka Connect esté listo..."
   until curl -s http://localhost:8083/ > /dev/null 2>&1; do
     echo "  Kafka Connect aún no disponible, esperando 10 segundos..."
     sleep 10
   done
   echo "Kafka Connect está listo."
   ```

5. Verifica que los plugins JDBC y Datagen están instalados:

   ```bash
   curl -s http://localhost:8083/connector-plugins | \
     jq -r '.[].class' | \
     grep -E "jdbc|datagen|FileStream" | sort
   ```

6. Verifica la base de datos PostgreSQL con los datos iniciales:

   ```bash
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "SELECT 'orders' as tabla, COUNT(*) FROM orders
         UNION ALL
         SELECT 'inventory', COUNT(*) FROM inventory
         UNION ALL
         SELECT 'products', COUNT(*) FROM products;"
   ```

**Salida Esperada:**

```
  tabla    | count
-----------+-------
 orders    |     8
 inventory |     8
 products  |     8
(3 rows)
```

Para los plugins de Kafka Connect:

```
io.confluent.connect.jdbc.JdbcSinkConnector
io.confluent.connect.jdbc.JdbcSourceConnector
io.confluent.kafka.connect.datagen.DatagenConnector
org.apache.kafka.connect.file.FileStreamSinkConnector
org.apache.kafka.connect.file.FileStreamSourceConnector
```

**Verificación:**

- Todos los contenedores están en estado `running`:

  ```bash
  docker compose ps --format "table {{.Name}}\t{{.Status}}" | grep -v "Up\|running" || echo "Todos los servicios están activos."
  ```

---

### Paso 5: Creación de Tópicos Kafka con Configuraciones de Producción

**Objetivo:** Crear los tópicos de Kafka del sistema ShopStream aplicando las configuraciones de particiones, replicación y retención definidas en el diseño arquitectónico.

**Instrucciones:**

1. Crea el script de creación de tópicos con las configuraciones del diseño:

   ```bash
   cat > ~/kafka-final-lab/scripts/create-topics.sh << 'EOF'
   #!/bin/bash
   # Script de creación de tópicos ShopStream
   # Aplica naming convention: {dominio}.{entidad}.{estado}

   BROKER="broker:29092"
   CONTAINER="shopstream-broker"

   echo "=============================================="
   echo "  Creando tópicos ShopStream"
   echo "=============================================="

   # Función helper para crear tópicos
   create_topic() {
     local name=$1
     local partitions=$2
     local replication=$3
     local retention_ms=$4
     local description=$5

     echo ""
     echo "Creando: $name"
     echo "  Particiones: $partitions | Replicación: $replication | Retención: $retention_ms ms"
     echo "  Descripción: $description"

     docker exec $CONTAINER kafka-topics \
       --bootstrap-server $BROKER \
       --create \
       --topic "$name" \
       --partitions $partitions \
       --replication-factor $replication \
       --config retention.ms=$retention_ms \
       --if-not-exists

     echo "  ✓ Tópico creado."
   }

   # Tópicos de ingestión (raw data - 7 días = 604800000 ms)
   create_topic "shopstream.orders.raw"      3 1 604800000 "Pedidos capturados desde PostgreSQL via JDBC Source"
   create_topic "shopstream.inventory.raw"   3 1 604800000 "Cambios de inventario desde PostgreSQL"

   # Tópicos de procesamiento (alertas - 30 días = 2592000000 ms)
   create_topic "shopstream.orders.highvalue" 1 1 2592000000 "Pedidos con valor > 500 USD (generado por ksqlDB)"
   create_topic "shopstream.alerts.operations" 1 1 2592000000 "Alertas exportadas a PostgreSQL via JDBC Sink"

   # Dead Letter Queue (3 días = 259200000 ms)
   create_topic "shopstream.connect.dlq"     1 1 259200000 "Dead Letter Queue para errores de conectores"

   echo ""
   echo "=============================================="
   echo "  Verificando tópicos creados"
   echo "=============================================="
   docker exec $CONTAINER kafka-topics \
     --bootstrap-server $BROKER \
     --list | grep "shopstream\." | sort

   echo ""
   echo "✓ Todos los tópicos ShopStream han sido creados."
   EOF

   chmod +x ~/kafka-final-lab/scripts/create-topics.sh
   echo "Script de tópicos creado."
   ```

2. Ejecuta el script de creación de tópicos:

   ```bash
   cd ~/kafka-final-lab
   bash scripts/create-topics.sh
   ```

3. Verifica la configuración detallada del tópico de órdenes:

   ```bash
   docker exec shopstream-broker kafka-topics \
     --bootstrap-server broker:29092 \
     --describe \
     --topic shopstream.orders.raw
   ```

**Salida Esperada:**

```
==============================================
  Creando tópicos ShopStream
==============================================

Creando: shopstream.orders.raw
  Particiones: 3 | Replicación: 1 | Retención: 604800000 ms
  Descripción: Pedidos capturados desde PostgreSQL via JDBC Source
  ✓ Tópico creado.
[... más tópicos ...]

==============================================
  Verificando tópicos creados
==============================================
shopstream.alerts.operations
shopstream.connect.dlq
shopstream.inventory.raw
shopstream.orders.highvalue
shopstream.orders.raw

✓ Todos los tópicos ShopStream han sido creados.
```

Para el describe del tópico:

```
Topic: shopstream.orders.raw	TopicId: [id]	PartitionCount: 3	ReplicationFactor: 1
	Topic: shopstream.orders.raw	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
	Topic: shopstream.orders.raw	Partition: 1	Leader: 1	Replicas: 1	Isr: 1
	Topic: shopstream.orders.raw	Partition: 2	Leader: 1	Replicas: 1	Isr: 1
```

**Verificación:**

- Confirma que los 5 tópicos de ShopStream existen:

  ```bash
  docker exec shopstream-broker kafka-topics \
    --bootstrap-server broker:29092 \
    --list | grep "shopstream\." | wc -l
  ```

  Debe retornar `5`.

---

### Paso 6: Configuración del JDBC Source Connector

**Objetivo:** Desplegar los conectores JDBC Source para capturar automáticamente los datos de las tablas `orders` e `inventory` de PostgreSQL y publicarlos en los tópicos Kafka correspondientes.

**Instrucciones:**

1. Crea el script de despliegue de conectores Source:

   ```bash
   cat > ~/kafka-final-lab/scripts/deploy-source-connectors.sh << 'EOF'
   #!/bin/bash
   # Despliega los JDBC Source Connectors para ShopStream
   CONNECT_URL="http://localhost:8083"

   echo "=============================================="
   echo "  Desplegando JDBC Source Connectors"
   echo "=============================================="

   # ─── Source Connector: orders ─────────────────────────────────
   echo ""
   echo "Registrando conector: shopstream-jdbc-source-orders"
   curl -s -X POST $CONNECT_URL/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "shopstream-jdbc-source-orders",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "tasks.max": "1",
         "connection.url": "jdbc:postgresql://postgres:5432/shopstream_db",
         "connection.user": "shopstream",
         "connection.password": "shopstream123",
         "mode": "timestamp+incrementing",
         "incrementing.column.name": "order_id",
         "timestamp.column.name": "updated_at",
         "table.whitelist": "orders",
         "topic.prefix": "shopstream.",
         "poll.interval.ms": "5000",
         "batch.max.rows": "100",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "errors.tolerance": "all",
         "errors.deadletterqueue.topic.name": "shopstream.connect.dlq",
         "errors.deadletterqueue.topic.replication.factor": "1",
         "errors.log.enable": "true",
         "errors.log.include.messages": "true",
         "transforms": "addSuffix",
         "transforms.addSuffix.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
         "transforms.addSuffix.renames": "order_id:orderId,customer_id:customerId,product_id:productId,total_amount:totalAmount,created_at:createdAt,updated_at:updatedAt"
       }
     }' | jq -r '.name // .message'

   # ─── Source Connector: inventory ──────────────────────────────
   echo ""
   echo "Registrando conector: shopstream-jdbc-source-inventory"
   curl -s -X POST $CONNECT_URL/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "shopstream-jdbc-source-inventory",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "tasks.max": "1",
         "connection.url": "jdbc:postgresql://postgres:5432/shopstream_db",
         "connection.user": "shopstream",
         "connection.password": "shopstream123",
         "mode": "timestamp+incrementing",
         "incrementing.column.name": "inventory_id",
         "timestamp.column.name": "updated_at",
         "table.whitelist": "inventory",
         "topic.prefix": "shopstream.",
         "poll.interval.ms": "5000",
         "batch.max.rows": "100",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "errors.tolerance": "all",
         "errors.deadletterqueue.topic.name": "shopstream.connect.dlq",
         "errors.deadletterqueue.topic.replication.factor": "1",
         "errors.log.enable": "true",
         "errors.log.include.messages": "true"
       }
     }' | jq -r '.name // .message'

   echo ""
   echo "Esperando 10 segundos para que los conectores inicialicen..."
   sleep 10

   echo ""
   echo "=============================================="
   echo "  Estado de los conectores Source"
   echo "=============================================="
   for connector in shopstream-jdbc-source-orders shopstream-jdbc-source-inventory; do
     echo ""
     echo "Conector: $connector"
     curl -s $CONNECT_URL/connectors/$connector/status | \
       jq -r '"  Estado conector: " + .connector.state + "\n  Estado tarea 0: " + .tasks[0].state'
   done
   EOF

   chmod +x ~/kafka-final-lab/scripts/deploy-source-connectors.sh
   echo "Script de conectores Source creado."
   ```

2. Ejecuta el script de despliegue:

   ```bash
   bash ~/kafka-final-lab/scripts/deploy-source-connectors.sh
   ```

3. Verifica que los datos están llegando al tópico de órdenes:

   ```bash
   docker exec shopstream-broker kafka-console-consumer \
     --bootstrap-server broker:29092 \
     --topic shopstream.orders \
     --from-beginning \
     --max-messages 3 \
     --timeout-ms 15000 2>/dev/null | head -20
   ```

   > **Nota:** El tópico se llama `shopstream.orders` porque el conector JDBC Source crea el tópico concatenando el `topic.prefix` con el nombre de la tabla. El tópico `shopstream.orders.raw` que creamos manualmente se usará en pasos posteriores con una transformación.

4. Verifica el número de mensajes en el tópico:

   ```bash
   sleep 5
   docker exec shopstream-broker kafka-run-class kafka.tools.GetOffsetShell \
     --broker-list broker:29092 \
     --topic shopstream.orders \
     --time -1 2>/dev/null | awk -F: '{sum += $3} END {print "Total mensajes en shopstream.orders: " sum}'
   ```

**Salida Esperada:**

```
==============================================
  Desplegando JDBC Source Connectors
==============================================

Registrando conector: shopstream-jdbc-source-orders
shopstream-jdbc-source-orders

Registrando conector: shopstream-jdbc-source-inventory
shopstream-jdbc-source-inventory

Esperando 10 segundos para que los conectores inicialicen...

==============================================
  Estado de los conectores Source
==============================================

Conector: shopstream-jdbc-source-orders
  Estado conector: RUNNING
  Estado tarea 0: RUNNING

Conector: shopstream-jdbc-source-inventory
  Estado conector: RUNNING
  Estado tarea 0: RUNNING
```

**Verificación:**

- Ambos conectores muestran estado `RUNNING`.
- El tópico `shopstream.orders` tiene al menos 8 mensajes (los registros iniciales de la BD).

---

### Paso 7: Procesamiento en Tiempo Real con ksqlDB

**Objetivo:** Crear streams y tablas ksqlDB para procesar los eventos de órdenes en tiempo real, filtrando pedidos de alto valor y agregando el inventario por categoría.

**Instrucciones:**

1. Crea el script de consultas ksqlDB:

   ```bash
   cat > ~/kafka-final-lab/scripts/ksqldb-setup.sql << 'EOF'
   -- ============================================================
   -- ShopStream ksqlDB Setup
   -- Procesamiento de flujos en tiempo real
   -- ============================================================

   -- Configurar lectura desde el inicio del tópico
   SET 'auto.offset.reset' = 'earliest';

   -- ─── STREAM: Órdenes raw desde PostgreSQL ─────────────────────
   CREATE STREAM IF NOT EXISTS orders_stream (
     orderId       INT,
     customerId    INT,
     productId     INT,
     quantity      INT,
     totalAmount   DOUBLE,
     status        VARCHAR,
     createdAt     VARCHAR,
     updatedAt     VARCHAR
   ) WITH (
     KAFKA_TOPIC   = 'shopstream.orders',
     VALUE_FORMAT  = 'AVRO',
     PARTITIONS    = 3
   );

   -- ─── STREAM: Inventario raw desde PostgreSQL ──────────────────
   CREATE STREAM IF NOT EXISTS inventory_stream (
     inventory_id  INT,
     product_id    INT,
     category      VARCHAR,
     stock_level   INT,
     warehouse     VARCHAR,
     updated_at    VARCHAR
   ) WITH (
     KAFKA_TOPIC   = 'shopstream.inventory',
     VALUE_FORMAT  = 'AVRO',
     PARTITIONS    = 3
   );

   -- ─── STREAM: Pedidos de alto valor (> $500) ───────────────────
   CREATE STREAM IF NOT EXISTS high_value_orders_stream
   WITH (
     KAFKA_TOPIC   = 'shopstream.orders.highvalue',
     VALUE_FORMAT  = 'AVRO',
     PARTITIONS    = 1
   ) AS
   SELECT
     orderId,
     customerId,
     productId,
     quantity,
     totalAmount,
     status,
     'HIGH_VALUE_ORDER' AS alertReason,
     CASE
       WHEN totalAmount >= 1000 THEN 'CRITICAL'
       WHEN totalAmount >= 500  THEN 'HIGH'
       ELSE 'NORMAL'
     END AS priorityLevel
   FROM orders_stream
   WHERE totalAmount > 500.00
   EMIT CHANGES;

   -- ─── TABLE: Agregación de inventario por categoría ────────────
   CREATE TABLE IF NOT EXISTS inventory_by_category_table
   WITH (
     KAFKA_TOPIC   = 'shopstream.inventory.by.category',
     VALUE_FORMAT  = 'AVRO',
     PARTITIONS    = 1
   ) AS
   SELECT
     category,
     COUNT(*)            AS total_products,
     SUM(stock_level)    AS total_stock,
     AVG(stock_level)    AS avg_stock,
     MIN(stock_level)    AS min_stock,
     MAX(stock_level)    AS max_stock
   FROM inventory_stream
   GROUP BY category
   EMIT CHANGES;

   -- ─── STREAM: Alertas para exportar a PostgreSQL ───────────────
   CREATE STREAM IF NOT EXISTS operations_alerts_stream
   WITH (
     KAFKA_TOPIC   = 'shopstream.alerts.operations',
     VALUE_FORMAT  = 'AVRO',
     PARTITIONS    = 1
   ) AS
   SELECT
     orderId       AS order_id,
     customerId    AS customer_id,
     totalAmount   AS total_amount,
     alertReason   AS alert_reason,
     priorityLevel AS priority_level
   FROM high_value_orders_stream
   EMIT CHANGES;

   EOF
   echo "Script ksqlDB creado."
   ```

2. Ejecuta el script de configuración en ksqlDB:

   ```bash
   docker exec -i shopstream-ksqldb-cli \
     ksql http://ksqldb-server:8088 \
     --execute "$(cat ~/kafka-final-lab/scripts/ksqldb-setup.sql)"
   ```

3. Verifica los streams y tablas creados:

   ```bash
   docker exec shopstream-ksqldb-cli \
     ksql http://ksqldb-server:8088 \
     --execute "SHOW STREAMS; SHOW TABLES;"
   ```

4. Consulta los pedidos de alto valor procesados hasta ahora:

   ```bash
   docker exec shopstream-ksqldb-cli \
     ksql http://ksqldb-server:8088 \
     --execute "SELECT orderId, customerId, totalAmount, priorityLevel FROM high_value_orders_stream LIMIT 5;"
   ```

5. Consulta la tabla de inventario por categoría:

   ```bash
   docker exec shopstream-ksqldb-cli \
     ksql http://ksqldb-server:8088 \
     --execute "SELECT * FROM inventory_by_category_table EMIT CHANGES LIMIT 5;"
   ```

**Salida Esperada:**

Para SHOW STREAMS:

```
 Stream Name                    | Kafka Topic                       | Key Format | Value Format | Windowed
----------------------------------------------------------------------------------------------------
 HIGH_VALUE_ORDERS_STREAM       | shopstream.orders.highvalue       | KAFKA      | AVRO         | false
 INVENTORY_STREAM               | shopstream.inventory              | KAFKA      | AVRO         | false
 OPERATIONS_ALERTS_STREAM       | shopstream.alerts.operations      | KAFKA      | AVRO         | false
 ORDERS_STREAM                  | shopstream.orders                 | KAFKA      | AVRO         | false
```

Para los pedidos de alto valor:

```
+----------+------------+-------------+---------------+
| ORDERID  | CUSTOMERID | TOTALAMOUNT | PRIORITYLEVEL |
+----------+------------+-------------+---------------+
| 1        | 1001       | 1299.99     | CRITICAL      |
| 4        | 1004       | 799.98      | HIGH          |
| 6        | 1006       | 2599.98     | CRITICAL      |
```

**Verificación:**

- Al menos 3 pedidos de alto valor deben aparecer en `high_value_orders_stream`.
- La tabla de inventario debe mostrar al menos 4 categorías distintas.

---

### Paso 8: Configuración del JDBC Sink Connector

**Objetivo:** Desplegar el conector JDBC Sink para exportar automáticamente las alertas de pedidos críticos del tópico Kafka a la tabla `critical_orders` en PostgreSQL.

**Instrucciones:**

1. Registra el JDBC Sink Connector mediante la API REST:

   ```bash
   curl -s -X POST http://localhost:8083/connectors \
     -H "Content-Type: application/json" \
     -d '{
       "name": "shopstream-jdbc-sink-alerts",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
         "tasks.max": "1",
         "connection.url": "jdbc:postgresql://postgres:5432/shopstream_db",
         "connection.user": "shopstream",
         "connection.password": "shopstream123",
         "topics": "shopstream.alerts.operations",
         "table.name.format": "critical_orders",
         "insert.mode": "upsert",
         "pk.mode": "record_value",
         "pk.fields": "order_id",
         "auto.create": "false",
         "auto.evolve": "false",
         "key.converter": "org.apache.kafka.connect.storage.StringConverter",
         "value.converter": "io.confluent.connect.avro.AvroConverter",
         "value.converter.schema.registry.url": "http://schema-registry:8081",
         "errors.tolerance": "all",
         "errors.deadletterqueue.topic.name": "shopstream.connect.dlq",
         "errors.deadletterqueue.topic.replication.factor": "1",
         "errors.log.enable": "true",
         "errors.log.include.messages": "true"
       }
     }' | jq .
   ```

2. Verifica el estado del conector Sink:

   ```bash
   sleep 10
   curl -s http://localhost:8083/connectors/shopstream-jdbc-sink-alerts/status | \
     jq -r '"Estado conector: " + .connector.state + "\nEstado tarea 0: " + .tasks[0].state'
   ```

3. Lista todos los conectores activos del sistema ShopStream:

   ```bash
   curl -s http://localhost:8083/connectors | jq -r '.[]' | sort
   ```

4. Verifica que las alertas llegaron a la tabla `critical_orders` en PostgreSQL:

   ```bash
   sleep 15
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "SELECT order_id, customer_id, total_amount, alert_reason, priority_level FROM critical_orders ORDER BY total_amount DESC;"
   ```

**Salida Esperada:**

Para el estado del conector:

```
Estado conector: RUNNING
Estado tarea 0: RUNNING
```

Para la lista de conectores:

```
shopstream-jdbc-sink-alerts
shopstream-jdbc-source-inventory
shopstream-jdbc-source-orders
```

Para la tabla `critical_orders`:

```
 order_id | customer_id | total_amount |   alert_reason    | priority_level
----------+-------------+--------------+-------------------+----------------
        6 |        1006 |      2599.98 | HIGH_VALUE_ORDER  | CRITICAL
        1 |        1001 |      1299.99 | HIGH_VALUE_ORDER  | CRITICAL
        4 |        1004 |       799.98 | HIGH_VALUE_ORDER  | HIGH
(3 rows)
```

**Verificación:**

- El conector Sink está en estado `RUNNING`.
- La tabla `critical_orders` contiene los pedidos con `total_amount > 500`.

---

### Paso 9: Prueba del Pipeline End-to-End con Datos Nuevos

**Objetivo:** Validar que el pipeline completo funciona correctamente insertando nuevos registros en PostgreSQL y verificando que fluyen a través de todos los componentes hasta llegar de vuelta a la base de datos.

**Instrucciones:**

1. Inserta nuevos pedidos en PostgreSQL para simular actividad de negocio en tiempo real:

   ```bash
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "
   INSERT INTO orders (customer_id, product_id, quantity, total_amount, status) VALUES
     (2001, 1, 3, 3899.97, 'PENDING'),
     (2002, 7, 1, 299.99, 'PROCESSING'),
     (2003, 6, 4, 1599.96, 'PENDING'),
     (2004, 2, 2, 399.98, 'COMPLETED'),
     (2005, 1, 5, 6499.95, 'PENDING');
   SELECT 'Insertados 5 nuevos pedidos' AS resultado;
   "
   ```

2. Actualiza el inventario para activar el flujo del conector de inventario:

   ```bash
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "
   UPDATE inventory SET stock_level = stock_level - 3 WHERE product_id = 1;
   UPDATE inventory SET stock_level = stock_level - 1 WHERE product_id = 7;
   UPDATE inventory SET stock_level = stock_level - 4 WHERE product_id = 6;
   SELECT product_id, category, stock_level FROM inventory WHERE product_id IN (1, 6, 7);
   "
   ```

3. Espera 15 segundos para que el pipeline procese los nuevos eventos:

   ```bash
   echo "Esperando procesamiento del pipeline (15 segundos)..."
   sleep 15
   echo "Verificando resultados..."
   ```

4. Verifica el conteo total de mensajes en el tópico de órdenes:

   ```bash
   docker exec shopstream-broker kafka-run-class kafka.tools.GetOffsetShell \
     --broker-list broker:29092 \
     --topic shopstream.orders \
     --time -1 2>/dev/null | awk -F: '{sum += $3} END {print "Total mensajes en shopstream.orders: " sum}'
   ```

5. Verifica los nuevos pedidos de alto valor procesados por ksqlDB:

   ```bash
   docker exec shopstream-ksqldb-cli \
     ksql http://ksqldb-server:8088 \
     --execute "SELECT orderId, customerId, totalAmount, priorityLevel FROM high_value_orders_stream LIMIT 10;"
   ```

6. Confirma que los nuevos pedidos críticos llegaron a PostgreSQL:

   ```bash
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "SELECT order_id, customer_id, total_amount, priority_level, processed_at FROM critical_orders ORDER BY total_amount DESC;"
   ```

**Salida Esperada:**

Para el conteo de mensajes (debe mostrar 13 = 8 iniciales + 5 nuevos):

```
Total mensajes en shopstream.orders: 13
```

Para los pedidos de alto valor nuevos:

```
+----------+------------+-------------+---------------+
| ORDERID  | CUSTOMERID | TOTALAMOUNT | PRIORITYLEVEL |
+----------+------------+-------------+---------------+
| 9        | 2001       | 3899.97     | CRITICAL      |
| 11       | 2003       | 1599.96     | CRITICAL      |
| 13       | 2005       | 6499.95     | CRITICAL      |
```

Para la tabla `critical_orders` actualizada:

```
 order_id | customer_id | total_amount | priority_level
----------+-------------+--------------+----------------
       13 |        2005 |      6499.95 | CRITICAL
        9 |        2001 |      3899.97 | CRITICAL
       11 |        2003 |      1599.96 | CRITICAL
        6 |        1006 |      2599.98 | CRITICAL
        1 |        1001 |      1299.99 | CRITICAL
        4 |        1004 |       799.98 | HIGH
(6 rows)
```

**Verificación:**

- El tópico `shopstream.orders` tiene 13 mensajes totales.
- Los 3 nuevos pedidos de alto valor aparecen en `critical_orders`.

---

### Paso 10: Configuración de Monitoreo con Prometheus y Grafana

**Objetivo:** Configurar un dashboard funcional en Grafana que muestre las métricas clave del clúster Kafka y los conectores del sistema ShopStream.

**Instrucciones:**

1. Verifica que Prometheus está recolectando métricas del broker:

   ```bash
   curl -s "http://localhost:9090/api/v1/targets" | \
     jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"'
   ```

2. Verifica que hay métricas de Kafka disponibles en Prometheus:

   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=kafka_server_BrokerTopicMetrics_MessagesInPerSec_rate1m" | \
     jq -r '.data.result | length | "Métricas de mensajes entrantes disponibles: " + tostring + " series"'
   ```

3. Crea un dashboard de Grafana mediante la API:

   ```bash
   cat > ~/kafka-final-lab/grafana/shopstream-dashboard.json << 'EOF'
   {
     "dashboard": {
       "id": null,
       "uid": "shopstream-kafka",
       "title": "ShopStream - Kafka Monitoring",
       "tags": ["kafka", "shopstream"],
       "timezone": "browser",
       "schemaVersion": 38,
       "version": 1,
       "refresh": "30s",
       "panels": [
         {
           "id": 1,
           "title": "Mensajes por Segundo (Broker)",
           "type": "stat",
           "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
           "targets": [
             {
               "expr": "sum(kafka_server_BrokerTopicMetrics_MessagesInPerSec_rate1m)",
               "legendFormat": "msg/s",
               "refId": "A"
             }
           ],
           "options": {
             "colorMode": "value",
             "graphMode": "area",
             "justifyMode": "center"
           }
         },
         {
           "id": 2,
           "title": "Bytes Entrantes por Segundo",
           "type": "stat",
           "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
           "targets": [
             {
               "expr": "sum(kafka_server_BrokerTopicMetrics_BytesInPerSec_rate1m)",
               "legendFormat": "bytes/s",
               "refId": "A"
             }
           ]
         },
         {
           "id": 3,
           "title": "Bytes Salientes por Segundo",
           "type": "stat",
           "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
           "targets": [
             {
               "expr": "sum(kafka_server_BrokerTopicMetrics_BytesOutPerSec_rate1m)",
               "legendFormat": "bytes/s",
               "refId": "A"
             }
           ]
         },
         {
           "id": 4,
           "title": "Particiones Activas",
           "type": "stat",
           "gridPos": {"h": 4, "w": 6, "x": 18, "y": 0},
           "targets": [
             {
               "expr": "kafka_server_ReplicaManager_PartitionCount_Value",
               "legendFormat": "particiones",
               "refId": "A"
             }
           ]
         },
         {
           "id": 5,
           "title": "Tasa de Mensajes Entrantes - Histórico",
           "type": "timeseries",
           "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4},
           "targets": [
             {
               "expr": "kafka_server_BrokerTopicMetrics_MessagesInPerSec_rate1m",
               "legendFormat": "{{topic}}",
               "refId": "A"
             }
           ]
         }
       ]
     },
     "overwrite": true,
     "folderId": 0
   }
   EOF

   curl -s -X POST http://localhost:3000/api/dashboards/db \
     -H "Content-Type: application/json" \
     -H "Authorization: Basic $(echo -n 'admin:shopstream123' | base64)" \
     -d @~/kafka-final-lab/grafana/shopstream-dashboard.json | \
     jq -r '"Dashboard: " + .url + " | Status: " + .status'
   ```

4. Verifica que el dashboard fue creado exitosamente:

   ```bash
   curl -s "http://localhost:3000/api/dashboards/uid/shopstream-kafka" \
     -H "Authorization: Basic $(echo -n 'admin:shopstream123' | base64)" | \
     jq -r '"Título: " + .dashboard.title + "\nUID: " + .dashboard.uid + "\nPaneles: " + (.dashboard.panels | length | tostring)'
   ```

5. Accede a los servicios web para verificación visual:

   ```bash
   echo ""
   echo "=============================================="
   echo "  URLs de acceso a los servicios ShopStream"
   echo "=============================================="
   echo "  Control Center:    http://localhost:9021"
   echo "  Grafana:           http://localhost:3000 (admin/shopstream123)"
   echo "  Prometheus:        http://localhost:9090"
   echo "  Kafka Connect API: http://localhost:8083"
   echo "  Schema Registry:   http://localhost:8081"
   echo "  ksqlDB REST API:   http://localhost:8088"
   echo "=============================================="
   ```

**Salida Esperada:**

Para los targets de Prometheus:

```
kafka-broker: up
kafka-connect: up
prometheus: up
```

Para el dashboard creado:

```
Dashboard: /d/shopstream-kafka/shopstream-kafka-monitoring | Status: success
```

Para la verificación del dashboard:

```
Título: ShopStream - Kafka Monitoring
UID: shopstream-kafka
Paneles: 5
```

**Verificación:**

- Accede a `http://localhost:3000` con credenciales `admin/shopstream123`.
- El dashboard "ShopStream - Kafka Monitoring" debe aparecer en la lista de dashboards.
- Los paneles de métricas deben mostrar datos (puede requerir esperar 30 segundos para el primer scrape).

---

### Paso 11: Verificación Final del Estado del Sistema

**Objetivo:** Realizar una verificación completa del estado de todos los componentes del sistema ShopStream para confirmar que el pipeline end-to-end está funcionando correctamente antes de la presentación.

**Instrucciones:**

1. Ejecuta el script de verificación completa del sistema:

   ```bash
   cat > ~/kafka-final-lab/scripts/verify-system.sh << 'EOF'
   #!/bin/bash
   # Script de verificación completa del sistema ShopStream

   PASS=0
   FAIL=0
   WARN=0

   check() {
     local description=$1
     local command=$2
     local expected=$3

     result=$(eval "$command" 2>/dev/null)
     if echo "$result" | grep -q "$expected"; then
       echo "  ✓ $description"
       ((PASS++))
     else
       echo "  ✗ $description (esperado: '$expected', obtenido: '$result')"
       ((FAIL++))
     fi
   }

   echo "=============================================="
   echo "  Verificación del Sistema ShopStream"
   echo "=============================================="

   echo ""
   echo "── Servicios Docker ──────────────────────────"
   check "Broker Kafka activo" "docker inspect --format='{{.State.Status}}' shopstream-broker" "running"
   check "PostgreSQL activo" "docker inspect --format='{{.State.Status}}' shopstream-postgres" "running"
   check "Schema Registry activo" "docker inspect --format='{{.State.Status}}' shopstream-schema-registry" "running"
   check "Kafka Connect activo" "docker inspect --format='{{.State.Status}}' shopstream-kafka-connect" "running"
   check "ksqlDB Server activo" "docker inspect --format='{{.State.Status}}' shopstream-ksqldb-server" "running"
   check "Prometheus activo" "docker inspect --format='{{.State.Status}}' shopstream-prometheus" "running"
   check "Grafana activo" "docker inspect --format='{{.State.Status}}' shopstream-grafana" "running"

   echo ""
   echo "── Tópicos Kafka ─────────────────────────────"
   check "Tópico orders.raw existe" \
     "docker exec shopstream-broker kafka-topics --bootstrap-server broker:29092 --list 2>/dev/null" \
     "shopstream.orders.raw"
   check "Tópico orders.highvalue existe" \
     "docker exec shopstream-broker kafka-topics --bootstrap-server broker:29092 --list 2>/dev/null" \
     "shopstream.orders.highvalue"
   check "Tópico alerts.operations existe" \
     "docker exec shopstream-broker kafka-topics --bootstrap-server broker:29092 --list 2>/dev/null" \
     "shopstream.alerts.operations"
   check "Tópico connect.dlq existe" \
     "docker exec shopstream-broker kafka-topics --bootstrap-server broker:29092 --list 2>/dev/null" \
     "shopstream.connect.dlq"

   echo ""
   echo "── Conectores Kafka Connect ──────────────────"
   check "Source connector orders RUNNING" \
     "curl -s http://localhost:8083/connectors/shopstream-jdbc-source-orders/status" \
     "RUNNING"
   check "Source connector inventory RUNNING" \
     "curl -s http://localhost:8083/connectors/shopstream-jdbc-source-inventory/status" \
     "RUNNING"
   check "Sink connector alerts RUNNING" \
     "curl -s http://localhost:8083/connectors/shopstream-jdbc-sink-alerts/status" \
     "RUNNING"

   echo ""
   echo "── ksqlDB Streams y Tablas ───────────────────"
   check "Stream orders_stream existe" \
     "curl -s http://localhost:8088/ksql -H 'Content-Type: application/vnd.ksql.v1+json' -d '{\"ksql\": \"SHOW STREAMS;\"}'" \
     "ORDERS_STREAM"
   check "Stream high_value_orders_stream existe" \
     "curl -s http://localhost:8088/ksql -H 'Content-Type: application/vnd.ksql.v1+json' -d '{\"ksql\": \"SHOW STREAMS;\"}'" \
     "HIGH_VALUE_ORDERS_STREAM"
   check "Table inventory_by_category_table existe" \
     "curl -s http://localhost:8088/ksql -H 'Content-Type: application/vnd.ksql.v1+json' -d '{\"ksql\": \"SHOW TABLES;\"}'" \
     "INVENTORY_BY_CATEGORY_TABLE"

   echo ""
   echo "── Base de Datos PostgreSQL ──────────────────"
   check "Tabla critical_orders tiene datos" \
     "docker exec shopstream-postgres psql -U shopstream -d shopstream_db -t -c 'SELECT COUNT(*) FROM critical_orders;'" \
     "[1-9]"
   check "Schema Registry responde" \
     "curl -s http://localhost:8081/subjects" \
     "\["

   echo ""
   echo "── Monitoreo ─────────────────────────────────"
   check "Prometheus responde" \
     "curl -s http://localhost:9090/-/ready" \
     "Prometheus"
   check "Grafana responde" \
     "curl -s http://localhost:3000/api/health" \
     "ok"

   echo ""
   echo "=============================================="
   echo "  RESULTADO FINAL"
   echo "  ✓ Verificaciones exitosas: $PASS"
   echo "  ✗ Verificaciones fallidas: $FAIL"
   if [ $FAIL -eq 0 ]; then
     echo ""
     echo "  🎉 ¡SISTEMA SHOPSTREAM COMPLETAMENTE OPERACIONAL!"
   else
     echo ""
     echo "  ⚠️  Hay $FAIL componentes que requieren atención."
   fi
   echo "=============================================="
   EOF

   chmod +x ~/kafka-final-lab/scripts/verify-system.sh
   bash ~/kafka-final-lab/scripts/verify-system.sh
   ```

**Salida Esperada:**

```
==============================================
  Verificación del Sistema ShopStream
==============================================

── Servicios Docker ──────────────────────────
  ✓ Broker Kafka activo
  ✓ PostgreSQL activo
  ✓ Schema Registry activo
  ✓ Kafka Connect activo
  ✓ ksqlDB Server activo
  ✓ Prometheus activo
  ✓ Grafana activo

── Tópicos Kafka ─────────────────────────────
  ✓ Tópico orders.raw existe
  ✓ Tópico orders.highvalue existe
  ✓ Tópico alerts.operations existe
  ✓ Tópico connect.dlq existe

── Conectores Kafka Connect ──────────────────
  ✓ Source connector orders RUNNING
  ✓ Source connector inventory RUNNING
  ✓ Sink connector alerts RUNNING

── ksqlDB Streams y Tablas ───────────────────
  ✓ Stream orders_stream existe
  ✓ Stream high_value_orders_stream existe
  ✓ Table inventory_by_category_table existe

── Base de Datos PostgreSQL ──────────────────
  ✓ Tabla critical_orders tiene datos
  ✓ Schema Registry responde

── Monitoreo ─────────────────────────────────
  ✓ Prometheus responde
  ✓ Grafana responde

==============================================
  RESULTADO FINAL
  ✓ Verificaciones exitosas: 18
  ✗ Verificaciones fallidas: 0

  🎉 ¡SISTEMA SHOPSTREAM COMPLETAMENTE OPERACIONAL!
==============================================
```

**Verificación:**

- El script debe reportar 0 verificaciones fallidas.
- Todos los componentes deben mostrar marca de verificación ✓.

---

### Paso 12: Presentación y Defensa del Pipeline (Fase 4 — 5 minutos)

**Objetivo:** Preparar la demostración en vivo del pipeline funcionando end-to-end y articular las decisiones de diseño tomadas durante la implementación.

**Instrucciones:**

1. Genera una carga de datos en tiempo real para la demostración en vivo:

   ```bash
   cat > ~/kafka-final-lab/scripts/demo-live.sh << 'EOF'
   #!/bin/bash
   # Script de demostración en vivo - genera pedidos continuos
   echo "Iniciando generación de pedidos en tiempo real..."
   echo "Presiona Ctrl+C para detener."
   echo ""

   ORDER_NUM=100
   while true; do
     CUSTOMER_ID=$((RANDOM % 9000 + 1000))
     PRODUCT_ID=$((RANDOM % 8 + 1))
     QUANTITY=$((RANDOM % 5 + 1))

     # Precios por producto (simulados)
     declare -A PRICES=([1]=1299.99 [2]=199.99 [3]=89.99 [4]=149.99 [5]=49.99 [6]=399.99 [7]=299.99 [8]=35.99)
     PRICE=${PRICES[$PRODUCT_ID]}
     TOTAL=$(echo "$PRICE * $QUANTITY" | bc)

     docker exec shopstream-postgres psql \
       -U shopstream \
       -d shopstream_db \
       -c "INSERT INTO orders (customer_id, product_id, quantity, total_amount, status)
           VALUES ($CUSTOMER_ID, $PRODUCT_ID, $QUANTITY, $TOTAL, 'PENDING');" \
       -q

     echo "  [$(date +%H:%M:%S)] Orden #$ORDER_NUM - Cliente $CUSTOMER_ID - Producto $PRODUCT_ID - Total: \$$TOTAL"
     ((ORDER_NUM++))
     sleep 2
   done
   EOF

   chmod +x ~/kafka-final-lab/scripts/demo-live.sh
   echo "Script de demo en vivo creado."
   echo ""
   echo "Para iniciar la demostración en vivo, ejecuta:"
   echo "  bash ~/kafka-final-lab/scripts/demo-live.sh"
   ```

2. Prepara el resumen de decisiones de diseño para la presentación:

   ```bash
   cat > ~/kafka-final-lab/PRESENTACION.md << 'EOF'
   # ShopStream - Guía de Presentación

   ## Flujo del Pipeline (5 minutos)

   ### 1. Ingestión (JDBC Source → Kafka)
   - PostgreSQL tabla `orders` → tópico `shopstream.orders`
   - PostgreSQL tabla `inventory` → tópico `shopstream.inventory`
   - Modo: timestamp+incrementing para capturar cambios
   - Polling cada 5 segundos

   ### 2. Procesamiento (ksqlDB)
   - `orders_stream` → filtra pedidos > $500 → `high_value_orders_stream`
   - `inventory_stream` → agrega por categoría → `inventory_by_category_table`
   - `high_value_orders_stream` → renombra campos → `operations_alerts_stream`

   ### 3. Exportación (JDBC Sink ← Kafka)
   - Tópico `shopstream.alerts.operations` → tabla `critical_orders` en PostgreSQL
   - Modo UPSERT con PK en `order_id`

   ### 4. Monitoreo
   - JMX Exporter expone métricas del broker en puerto 9101
   - Prometheus scrape cada 15 segundos
   - Grafana dashboard con 5 paneles de métricas clave

   ## Decisiones de Diseño a Defender

   1. **¿Por qué 3 particiones en tópicos de ingestión?**
      → Permite paralelismo futuro con múltiples consumidores sin reconfiguración.

   2. **¿Por qué retención diferenciada (7 días vs 30 días)?**
      → Datos raw son voluminosos; alertas tienen requisito de auditoría de 30 días.

   3. **¿Por qué Dead Letter Queue en todos los conectores?**
      → Tolerancia a errores sin detener el pipeline; los mensajes problemáticos
        se analizan en el DLQ sin impactar el flujo principal.

   4. **¿Por qué AvroConverter + Schema Registry?**
      → Garantiza compatibilidad de esquemas entre productores y consumidores;
        evolución controlada de schemas sin romper downstream.

   5. **¿Por qué modo timestamp+incrementing en JDBC Source?**
      → Captura tanto inserts nuevos (incrementing) como updates (timestamp),
        garantizando que ningún cambio se pierda.

   EOF
   echo "Guía de presentación creada."
   cat ~/kafka-final-lab/PRESENTACION.md
   ```

3. Muestra el estado final completo del sistema para la demostración:

   ```bash
   echo ""
   echo "=============================================="
   echo "  RESUMEN FINAL DEL SISTEMA SHOPSTREAM"
   echo "=============================================="
   echo ""
   echo "── Contenedores activos ──"
   docker compose ps --format "table {{.Name}}\t{{.Status}}" 2>/dev/null | grep -v "^NAME"
   echo ""
   echo "── Tópicos Kafka (ShopStream) ──"
   docker exec shopstream-broker kafka-topics \
     --bootstrap-server broker:29092 \
     --list 2>/dev/null | grep "shopstream\."
   echo ""
   echo "── Conectores activos ──"
   curl -s http://localhost:8083/connectors | jq -r '.[]'
   echo ""
   echo "── Pedidos críticos en PostgreSQL ──"
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "SELECT COUNT(*) AS total_critical_orders FROM critical_orders;" \
     -t | tr -d ' '
   echo " pedidos críticos registrados en PostgreSQL"
   ```

**Salida Esperada:**

```
Guía de presentación creada.

# ShopStream - Guía de Presentación
[... contenido del archivo ...]

==============================================
  RESUMEN FINAL DEL SISTEMA SHOPSTREAM
==============================================

── Contenedores activos ──
shopstream-broker          Up X minutes (healthy)
shopstream-control-center  Up X minutes
shopstream-grafana         Up X minutes
shopstream-kafka-connect   Up X minutes
shopstream-ksqldb-cli      Up X minutes
shopstream-ksqldb-server   Up X minutes
shopstream-postgres        Up X minutes (healthy)
shopstream-prometheus      Up X minutes
shopstream-schema-registry Up X minutes

── Tópicos Kafka (ShopStream) ──
shopstream.alerts.operations
shopstream.connect.dlq
shopstream.inventory.raw
shopstream.orders.highvalue
shopstream.orders.raw

── Conectores activos ──
shopstream-jdbc-sink-alerts
shopstream-jdbc-source-inventory
shopstream-jdbc-source-orders

── Pedidos críticos en PostgreSQL ──
6 pedidos críticos registrados en PostgreSQL
```

**Verificación:**

- Todos los 9 contenedores están activos.
- Los 3 conectores están listados.
- La tabla `critical_orders` tiene al menos 6 registros.

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El archivo `ARQUITECTURA.md` documenta todos los tópicos con justificación de particiones y retención.
- [ ] Los 5 tópicos de ShopStream existen en el broker con la configuración correcta.
- [ ] Los 2 conectores JDBC Source están en estado `RUNNING` y capturan datos de PostgreSQL.
- [ ] El conector JDBC Sink está en estado `RUNNING` y exporta alertas a PostgreSQL.
- [ ] Los 4 objetos ksqlDB (streams y tabla) están creados y procesando eventos.
- [ ] La tabla `critical_orders` en PostgreSQL contiene los pedidos con `total_amount > 500`.
- [ ] El dashboard de Grafana "ShopStream - Kafka Monitoring" está creado y muestra métricas.
- [ ] El script de verificación `verify-system.sh` reporta 0 fallos.
- [ ] El participante puede explicar al menos 3 decisiones de diseño arquitectónico.

### Procedimiento de Prueba

1. Ejecuta la verificación completa del sistema:

   ```bash
   bash ~/kafka-final-lab/scripts/verify-system.sh
   ```

   **Resultado Esperado:** `✓ Verificaciones exitosas: 18` y `✗ Verificaciones fallidas: 0`

2. Verifica el flujo completo insertando un pedido de prueba y rastreándolo:

   ```bash
   # Insertar pedido de prueba con valor conocido
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "INSERT INTO orders (customer_id, product_id, quantity, total_amount, status)
         VALUES (9999, 1, 2, 2599.98, 'PENDING') RETURNING order_id;"
   ```

   **Resultado Esperado:** Retorna el `order_id` del nuevo pedido.

3. Espera 20 segundos y verifica que el pedido llegó a la tabla de alertas:

   ```bash
   sleep 20
   docker exec shopstream-postgres psql \
     -U shopstream \
     -d shopstream_db \
     -c "SELECT * FROM critical_orders WHERE customer_id = 9999;"
   ```

   **Resultado Esperado:** El pedido del cliente 9999 aparece en `critical_orders` con `priority_level = 'CRITICAL'`.

4. Verifica los esquemas registrados en Schema Registry:

   ```bash
   curl -s http://localhost:8081/subjects | jq -r '.[]' | grep "shopstream" | sort
   ```

   **Resultado Esperado:** Al menos 4 schemas registrados para los tópicos de ShopStream.

5. Verifica la API de ksqlDB directamente:

   ```bash
   curl -s -X POST http://localhost:8088/ksql \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d '{"ksql": "SHOW STREAMS EXTENDED;"}' | \
     jq -r '.[] | .sourceDescription | .name + " -> " + .topic'
   ```

   **Resultado Esperado:** Lista de streams con sus tópicos asociados.

---

## Solución de Problemas

### Problema 1: Kafka Connect tarda demasiado en iniciar o falla al instalar plugins

**Síntomas:**

- `curl http://localhost:8083/` retorna `Connection refused` después de 5 minutos.
- Los logs muestran errores al ejecutar `confluent-hub install`.
- El contenedor `shopstream-kafka-connect` se reinicia repetidamente.

**Causa:**

La instalación de plugins mediante `confluent-hub install` en el comando de arranque requiere conectividad a internet. En entornos con conectividad limitada o si el servidor de Confluent Hub está temporalmente inaccesible, la descarga falla.

**Solución:**

```bash
# Verificar los logs del contenedor para diagnosticar el error
docker logs shopstream-kafka-connect --tail 50

# Si el problema es de red, intentar instalar manualmente dentro del contenedor
docker exec -it shopstream-kafka-connect bash -c "
  confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.7.4 && \
  confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.6.3
"

# Reiniciar el contenedor después de la instalación manual
docker restart shopstream-kafka-connect

# Esperar a que esté listo
sleep 60
curl -s http://localhost:8083/connector-plugins | jq '.[].class' | grep -E "jdbc|datagen"
```

---

### Problema 2: Los conectores JDBC Source están en estado FAILED

**Síntomas:**

- `curl -s http://localhost:8083/connectors/shopstream-jdbc-source-orders/status` muestra `"state": "FAILED"`.
- Los logs del conector muestran `Connection refused` o `Authentication failed`.

**Causa:**

El conector JDBC no puede conectarse a PostgreSQL. Puede ser porque PostgreSQL aún no está completamente iniciado, las credenciales son incorrectas, o el driver JDBC no puede resolver el hostname `postgres`.

**Solución:**

```bash
# Verificar que PostgreSQL está saludable
docker inspect --format='{{.State.Health.Status}}' shopstream-postgres

# Verificar conectividad desde el contenedor de Kafka Connect a PostgreSQL
docker exec shopstream-kafka-connect \
  curl -v telnet://postgres:5432 2>&1 | head -5

# Ver el trace del error en el estado de la tarea
curl -s http://localhost:8083/connectors/shopstream-jdbc-source-orders/status | \
  jq -r '.tasks[0].trace // "Sin trace disponible"'

# Eliminar y recrear el conector
curl -X DELETE http://localhost:8083/connectors/shopstream-jdbc-source-orders
sleep 5
bash ~/kafka-final-lab/scripts/deploy-source-connectors.sh
```

---

### Problema 3: ksqlDB no puede crear el stream porque el tópico no existe

**Síntomas:**

- Al ejecutar `CREATE STREAM orders_stream` aparece el error: `Topic 'shopstream.orders' does not exist`.
- El conector JDBC Source está en `RUNNING` pero el tópico no fue creado.

**Causa:**

El conector JDBC Source crea el tópico automáticamente cuando procesa el primer batch de datos. Si no hay registros en la tabla de PostgreSQL o el conector aún no ha hecho el primer poll, el tópico puede no existir aún.

**Solución:**

```bash
# Verificar si el tópico existe
docker exec shopstream-broker kafka-topics \
  --bootstrap-server broker:29092 \
  --list | grep "shopstream\."

# Verificar el offset del conector (si está en 0, aún no procesó datos)
curl -s http://localhost:8083/connectors/shopstream-jdbc-source-orders/offsets 2>/dev/null | jq .

# Forzar que el conector haga un poll reiniciando la tarea
curl -X POST http://localhost:8083/connectors/shopstream-jdbc-source-orders/tasks/0/restart

# Esperar 15 segundos y verificar de nuevo
sleep 15
docker exec shopstream-broker kafka-topics \
  --bootstrap-server broker:29092 \
  --list | grep "shopstream\."

# Si el tópico sigue sin existir, verificar que PostgreSQL tiene datos
docker exec shopstream-postgres psql \
  -U shopstream \
  -d shopstream_db \
  -c "SELECT COUNT(*) FROM orders;"
```

---

### Problema 4: La tabla critical_orders no recibe datos del Sink Connector

**Síntomas:**

- El conector Sink está en `RUNNING` pero `SELECT COUNT(*) FROM critical_orders` retorna 0.
- No hay errores en el estado del conector.

**Causa:**

El tópico `shopstream.alerts.operations` puede estar vacío porque ksqlDB aún no ha procesado ningún pedido de alto valor, o porque el stream de ksqlDB no está configurado con `auto.offset.reset = earliest`.

**Solución:**

```bash
# Verificar si hay mensajes en el tópico de alertas
docker exec shopstream-broker kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list broker:29092 \
  --topic shopstream.alerts.operations \
  --time -1 2>/dev/null

# Verificar si ksqlDB está procesando desde el inicio
docker exec shopstream-ksqldb-cli \
  ksql http://ksqldb-server:8088 \
  --execute "DESCRIBE EXTENDED high_value_orders_stream;"

# Forzar reprocesamiento eliminando y recreando el stream con offset desde el inicio
docker exec shopstream-ksqldb-cli \
  ksql http://ksqldb-server:8088 \
  --execute "
    SET 'auto.offset.reset' = 'earliest';
    DROP STREAM IF EXISTS operations_alerts_stream DELETE TOPIC;
    DROP STREAM IF EXISTS high_value_orders_stream DELETE TOPIC;
  "

# Volver a ejecutar el script de ksqlDB
docker exec -i shopstream-ksqldb-cli \
  ksql http://ksqldb-server:8088 \
  --execute "$(cat ~/kafka-final-lab/scripts/ksqldb-setup.sql)"
```

---

### Problema 5: El broker Kafka no inicia en modo KRaft

**Síntomas:**

- El contenedor `shopstream-broker` se reinicia repetidamente.
- Los logs muestran: `ERROR Fatal error during KafkaServer startup` o problemas con el `CLUSTER_ID`.

**Causa:**

El `CLUSTER_ID` debe ser un UUID válido en formato Base64. Si el volumen de datos ya contiene un cluster ID diferente al especificado en la variable de entorno, el broker rechazará el inicio.

**Solución:**

```bash
# Ver los logs del broker para identificar el error exacto
docker logs shopstream-broker --tail 30

# Si el error es de CLUSTER_ID incompatible, eliminar el volumen de datos
docker compose down
docker volume rm kafka-final-lab_kafka-data

# Generar un nuevo CLUSTER_ID válido
docker run --rm confluentinc/cp-kafka:8.0.0 kafka-storage random-uuid

# Actualizar el docker-compose.yml con el nuevo CLUSTER_ID generado
# y volver a levantar el ecosistema
docker compose up -d broker
sleep 30
docker logs shopstream-broker --tail 10
```

---

## Limpieza

```bash
# Detener todos los servicios preservando los volúmenes (para revisión posterior)
cd ~/kafka-final-lab
docker compose down

# Para eliminar COMPLETAMENTE todos los datos y volúmenes (limpieza total)
docker compose down -v

# Eliminar el directorio del laboratorio
rm -rf ~/kafka-final-lab

# Verificar que no quedan contenedores de ShopStream
docker ps -a | grep shopstream

# Limpiar imágenes no utilizadas (opcional, libera espacio en disco)
docker image prune -f
```

> ⚠️ **Advertencia:** Ejecutar `docker compose down -v` eliminará PERMANENTEMENTE todos los datos de PostgreSQL, Kafka, Prometheus y Grafana. Usa `docker compose down` (sin `-v`) si deseas conservar los datos para revisión o demostración posterior. Asegúrate de haber completado la presentación y capturado evidencias antes de ejecutar la limpieza total.

> ⚠️ **Nota para Windows con WSL2:** Si los archivos del laboratorio están dentro del filesystem de WSL2 (`~/`), el comando `rm -rf` los eliminará permanentemente. Considera hacer un backup de `ARQUITECTURA.md` y `PRESENTACION.md` si deseas conservar la documentación.

---

## Resumen

### Lo que Lograste

- Diseñaste una arquitectura Kafka completa para un sistema de e-commerce real, documentando todas las decisiones de diseño sobre tópicos, particiones, retención y conectores.
- Implementaste un pipeline end-to-end funcional que integra PostgreSQL, Kafka Connect (JDBC Source y Sink), Schema Registry, ksqlDB y monitoreo con Prometheus y Grafana.
- Configuraste 3 conectores Kafka Connect (2 Source, 1 Sink) con Dead Letter Queue y tolerancia a errores habilitada.
- Creaste 4 objetos ksqlDB (streams y tabla) para procesamiento de flujos en tiempo real, incluyendo filtrado de pedidos de alto valor y agregación de inventario por categoría.
- Aplicaste naming conventions consistentes, configuraciones de retención diferenciadas y buenas prácticas de producción en todos los componentes del sistema.
- Construiste un dashboard de Grafana con métricas clave del broker Kafka integrado con Prometheus.

### Conceptos Clave Demostrados

- **Integración end-to-end**: El pipeline completo demuestra cómo datos originados en PostgreSQL fluyen automáticamente a través de Kafka, son procesados en tiempo real por ksqlDB, y regresan a PostgreSQL — todo sin código personalizado de producción/consumo.
- **Kafka Connect como framework de integración**: Los conectores JDBC Source y Sink eliminan la necesidad de escribir código para integrar sistemas externos, usando configuración declarativa via API REST.
- **Procesamiento de flujos con ksqlDB**: Las consultas continuas `EMIT CHANGES` permiten reaccionar a eventos en tiempo real con latencia de segundos, habilitando casos de uso como alertas de negocio y vistas materializadas.
- **Dead Letter Queue para resiliencia**: La configuración de DLQ en todos los conectores garantiza que errores individuales no detengan el pipeline, siguiendo el principio de tolerancia a fallos en producción.
- **Observabilidad como práctica de producción**: La integración de JMX Exporter, Prometheus y Grafana proporciona visibilidad operacional indispensable para operar Kafka en entornos de producción reales.

### Próximos Pasos

- Explorar **Debezium** como alternativa a JDBC Source para implementar Change Data Capture (CDC) real, que captura cambios a nivel de WAL de PostgreSQL con menor latencia.
- Investigar **Kafka Streams** como alternativa a ksqlDB para procesamiento de flujos con lógica de negocio más compleja que requiere código Java/Scala.
- Profundizar en **configuración de seguridad** (SASL/SCRAM, TLS mutual, ACLs) para llevar el ecosistema ShopStream a un entorno de producción real.
- Explorar **Confluent Cloud** como alternativa managed que elimina la operación del clúster y permite enfocarse en la lógica de negocio.
- Estudiar patrones avanzados de **Schema Evolution** con Schema Registry para gestionar cambios en esquemas de datos sin interrumpir consumidores existentes.

---

## Recursos Adicionales

- **Confluent Hub** (https://www.confluent.io/hub) — Repositorio oficial de conectores con más de 200 integraciones. Busca conectores para los sistemas que necesitas integrar en tu entorno real.
- **Documentación oficial de Kafka Connect** (https://kafka.apache.org/documentation/#connect) — Referencia completa de la API REST, configuración de workers, transformaciones (SMT) y modelo de ejecución de tareas.
- **ksqlDB Documentation** (https://docs.ksqldb.io) — Guía completa de sintaxis ksqlDB, tipos de datos, funciones de ventana temporal y casos de uso avanzados de procesamiento de flujos.
- **Debezium Documentation** (https://debezium.io/documentation) — Implementación de Change Data Capture para PostgreSQL, MySQL, MongoDB y otros sistemas con captura de cambios a nivel de log de transacciones.
- **Prometheus Kafka Exporter** (https://github.com/danielqsj/kafka_exporter) — Alternativa al JMX Exporter para exponer métricas de Kafka en formato Prometheus con mayor facilidad de configuración.
- **Grafana Kafka Dashboards** (https://grafana.com/grafana/dashboards/?search=kafka) — Dashboards pre-construidos de la comunidad para monitoreo de Kafka, listos para importar en Grafana.
- **Confluent Platform Documentation** (https://docs.confluent.io/platform/current/) — Documentación completa de todos los componentes de Confluent Platform 8.0 incluyendo guías de configuración de producción.
