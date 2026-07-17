# Día 2 — Micrometer y Prometheus

## Objetivos
Publicar métricas técnicas y de negocio sin alta cardinalidad.

## Contenido
- Counter, Timer y Gauge;
- métricas JVM, HTTP, Hikari, Redis y Kafka;
- tags permitidos (`usecase`, `result`) y prohibidos (`userId`, `eventId`);
- histogramas, percentiles y SLI.

## Laboratorio
Medir inscripciones creadas/rechazadas, latencia del caso de uso, hits de caché y edad del outbox. Configurar Prometheus local según [Monitoreo](../../COMPLEMENTOS/README_Monitoreo.md).

## Pruebas
Generar tráfico conocido y comprobar incremento, unidades y ausencia de datos sensibles.

