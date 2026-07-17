# Día 3 — Grafana y diagnóstico

## Objetivos
Construir un dashboard que responda preguntas operativas.

## Dashboard obligatorio
- throughput, errores y p95 HTTP;
- pool PostgreSQL;
- Redis hits/misses;
- Kafka consumer lag/DLT;
- pendientes y edad máxima de outbox;
- inscripciones por resultado.

## Laboratorio
Provocar latencia, apagar Redis, detener un consumidor y diagnosticar cada caso usando métricas y logs.

## Regla
Prometheus identifica cada aplicación/instancia; no se ocultan todos los targets detrás de Gateway.

## Criterios
Señales 30%, dashboard 25%, diagnóstico 30%, seguridad 15%.

