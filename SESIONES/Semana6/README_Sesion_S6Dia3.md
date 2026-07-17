# Día 3 — Outbox, observabilidad y cierre profesional

## Objetivos

Cerrar el módulo sin ocultar el problema de escritura dual y dejar una aplicación operable.

## Transactional Outbox

Guardar inscripción y `outbox_event` en la misma transacción PostgreSQL. Un relay publica lotes en Kafka y marca el resultado. El consumidor sigue siendo idempotente porque una caída después de publicar puede duplicar.

Implementación mínima:

- migración Flyway `outbox_events`;
- `event_id`, tipo, versión, agregado, payload, fecha, estado e intentos;
- relay con lote, locking, backoff y métrica de edad;
- prueba: Kafka caído durante inscripción y recuperación posterior.

## Observabilidad

Usa como apoyo [Monitoreo con Prometheus y Grafana](../../COMPLEMENTOS/README_Monitoreo.md).

- logs estructurados y correlation ID;
- Actuator con health, liveness y readiness;
- Micrometer/Prometheus;
- métricas HTTP, PostgreSQL, Redis, Kafka, DLT y outbox;
- dashboard Grafana básico.

## Demo final de seis semanas

1. Registro/login y permisos.
2. CRUD contextual de rutas/módulos.
3. Inscripción única en PostgreSQL/Flyway.
4. Caché observable e invalidación.
5. Outbox → Kafka → consumidor idempotente.
6. Fallo de Redis/Kafka y recuperación.
7. `./mvnw clean verify` con Java 17 y 21.
8. Arranque mediante Docker Compose.

## Alcance

El producto final es un monolito modular profesional preparado para microservicios. La extracción comienza en Semana 13; Gateway, discovery y resiliencia distribuida se estudian en Semana 14, y CI/CD se profundiza en Semana 16.

## Evaluación

Funcionalidad 25%, arquitectura 20%, pruebas 20%, seguridad 15%, operación/observabilidad 20%.
