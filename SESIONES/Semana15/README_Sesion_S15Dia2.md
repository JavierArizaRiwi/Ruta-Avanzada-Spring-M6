# Día 2 — Trazabilidad distribuida

## Objetivos
Relacionar HTTP, base, Kafka y consumidor.

## Contenido
- W3C Trace Context;
- Micrometer Tracing/OpenTelemetry;
- propagación en headers Kafka;
- sampling y baggage limitado;
- diferencia entre correlation ID y trace completo.

## Laboratorio
Seguir `POST enrollment` → outbox → Kafka → notificación. Visualizar spans localmente como reto opcional y conservar logs correlacionados como base.

## Regla
No incluir coderId, email o token como baggage/tag de alta cardinalidad.

