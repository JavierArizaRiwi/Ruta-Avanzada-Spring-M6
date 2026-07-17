# Práctica Semana 6 — Caché, eventos y operación

## Objetivo

Integrar Redis y Kafka sin perder consistencia ni observabilidad.

## Flujo

1. Consultar catálogo cacheado.
2. Crear inscripción válida en PostgreSQL.
3. Registrar `EnrollmentCreatedV1` en outbox.
4. Publicar con relay a Kafka.
5. Consumir idempotentemente una notificación.
6. Observar métricas y logs de toda la operación.

## Pruebas obligatorias

- hit, miss, TTL e invalidación Redis;
- mensaje Kafka duplicado;
- poison message y DLT;
- Kafka caído durante inscripción y recuperación del outbox;
- Redis caído sin pérdida de datos;
- correlation ID y métricas sin datos sensibles.

## Demo

Levantar con Docker Compose, ejecutar el flujo, provocar fallos y recuperar. Validar con Java 17 y 21 mediante Maven Wrapper.

## Evaluación

Consistencia 30%, mensajería/idempotencia 25%, caché 15%, pruebas 15%, observabilidad/operación 15%.

