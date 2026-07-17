# Día 2 — Extracción de notifications-service

## Objetivos
Aplicar branch-by-abstraction y datos por servicio.

## Arquitectura
La API principal registra inscripciones/outbox. `notifications-service` consume Kafka, posee su registro de envíos y no consulta tablas del monolito.

## Laboratorio
Crear servicio Spring Boot independiente con Wrapper, health, métricas, listener idempotente, PostgreSQL propio y Dockerfile.

## Contratos
Compartir especificación/event schema, no entidades ni lógica de dominio mutable.

## Pruebas
Contrato, integración Kafka/PostgreSQL y caída del servicio sin rollback de inscripción.

## Antipatrones
Base compartida, librería `common` gigante o despliegue siempre coordinado.

