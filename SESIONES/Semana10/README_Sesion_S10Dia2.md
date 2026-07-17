# Día 2 — Transactional Outbox con Spring y Flyway

## Objetivos
Persistir cambio de negocio y evento en una transacción PostgreSQL.

## Implementación
- migración `outbox_events`;
- `event_id`, tipo, versión, agregado, payload, fecha, estado e intentos;
- inserción desde el caso de uso transaccional;
- payload estable separado de la entidad.

## Laboratorio
Al crear una inscripción, guardar `Enrollment` y `EnrollmentCreatedV1` en outbox. Detener Kafka y comprobar que el evento permanece pendiente.

## Pruebas
Transacción exitosa, rollback completo, restricción de `event_id` y migración desde base vacía.

## Antipatrones
Serializar entidades, marcar publicado antes del envío o borrar inmediatamente sin auditoría.

