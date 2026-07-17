# Apache Kafka en entorno local

## Modelo mental

Kafka es un log distribuido. Un productor escribe records en tópicos particionados; consumidores de un grupo avanzan offsets. No es una cola mágica ni garantiza exactamente una vez para efectos externos.

## Conceptos obligatorios

- evento frente a comando;
- broker, tópico, partición y réplica;
- clave y orden dentro de una partición;
- offset y consumer group;
- at-least-once, duplicados e idempotencia;
- retención, compaction, retry y dead-letter topic;
- evolución de esquema y compatibilidad.

## Kafka local con KRaft

```yaml
services:
  kafka:
    image: apache/kafka:4.2.1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: CONTROLLER://:9093,PLAINTEXT://:29092,PLAINTEXT_HOST://:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    ports: ["9092:9092"]
```

Un broker y réplica 1 sirven para aprender, no representan alta disponibilidad.

## Eventos de Riwi Learning Platform

```text
EnrollmentCreated.v1
SubmissionCreated.v1
GradeAssigned.v1
ModuleCompleted.v1
RouteCompleted.v1
```

Todo evento incluye `eventId`, `eventType`, `version`, `occurredAt`, `aggregateId` y payload mínimo.

## Prácticas

- crear tópicos explícitamente;
- elegir clave por requisito de orden;
- probar mensajes duplicados;
- medir consumer lag;
- limitar retries y enviar poison messages a DLT;
- no incluir entidades JPA o secretos en payloads.

## Recursos oficiales

- [Apache Kafka](https://kafka.apache.org/documentation/)

