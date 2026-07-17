# Día 2 — Spring for Apache Kafka

## Objetivos

Publicar y consumir eventos con retry, DLT e idempotencia. Estudiar primero [Apache Kafka](../../COMPLEMENTOS/README_Apache_Kafka.md).

## Evento

`EnrollmentCreatedV1` incluye `eventId`, `occurredAt`, `aggregateId`, versión y payload mínimo. Se publica con `KafkaTemplate` usando `enrollmentId` como clave.

## Consumidor

`@KafkaListener` delega a un caso de uso de notificaciones. Una restricción única sobre `processed_events.event_id` evita repetir efectos.

## Manejo de errores

- transitorio: retry limitado con backoff;
- contrato inválido: dead-letter topic;
- bug: alerta, no retry infinito;
- orden solo dentro de una partición;
- tópicos explícitos, sin auto-create como dependencia.

## Laboratorio

Levantar Kafka KRaft, producir inscripción, consumir notificación y reenviar el mismo record para demostrar idempotencia.

## Pruebas

Kafka Testcontainers: éxito, duplicado, caída temporal, poison message, DLT y recuperación.

## Antipatrones

Kafka como RPC, entidades JPA en payload, clave aleatoria o promesa de exactly-once para efectos externos.
