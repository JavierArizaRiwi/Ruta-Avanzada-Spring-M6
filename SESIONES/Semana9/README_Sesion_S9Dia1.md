# Día 1 — Productores con Spring for Apache Kafka

## Objetivos
Publicar eventos versionados desde Spring Boot. Estudiar primero [Apache Kafka](../../COMPLEMENTOS/README_Apache_Kafka.md).

## Conceptos Spring
- dependencia gestionada por el BOM de Spring Boot;
- `KafkaTemplate` y `NewTopic`;
- serialización JSON y headers;
- claves de partición;
- callbacks, timeouts y errores de envío.

## Laboratorio
Publicar `EnrollmentCreatedV1` con clave `enrollmentId`. El record incluye metadatos mínimos y no expone entidades JPA.

## Pruebas
Kafka Testcontainers, verificación de tópico/clave/payload y caso de broker no disponible.

## Antipatrones
Auto-crear tópicos silenciosamente, payload gigante, dato sensible o `.send()` ignorado sin observabilidad.

