# Día 3 — Relay, reintentos y recuperación

## Objetivos
Publicar lotes del outbox de forma segura y observable.

## Implementación Spring
Tarea programada o componente dedicado; claim de lotes con locking; backoff; estado publicado/fallido; métrica de edad del evento más antiguo.

## Laboratorio
Procesar un lote, simular caída después de publicar y antes de marcar, y demostrar que el consumidor idempotente tolera el duplicado.

## Profundización
Comparar polling publisher con CDC/Debezium. Debezium no es obligatorio para el laboratorio local.

## Evidencias
Pruebas de recuperación, dashboard de pendientes/fallos y runbook para reprocesar DLT/outbox.

## Criterios
Atomicidad 35%, relay 30%, idempotencia 20%, operación 15%.

