# Día 2 — Consumidores idempotentes, retry y DLT

## Objetivos
Procesar eventos al menos una vez sin repetir efectos.

## Arquitectura
`@KafkaListener` delega a un caso de uso. Una tabla `processed_events` registra `eventId` bajo una restricción única.

## Laboratorio
Crear consumidor de notificaciones, reenviar el mismo evento y demostrar una única notificación. Configurar retry con backoff y dead-letter topic para errores no recuperables.

## Decisiones
- excepción transitoria: retry limitado;
- contrato inválido: DLT;
- bug de programación: alerta, no retry infinito;
- commit de offset coherente con la transacción del consumidor.

## Pruebas
Éxito, duplicado, fallo temporal, poison message y recuperación.

## Preguntas
¿Por qué Kafka puede duplicar? ¿Qué orden existe entre particiones? ¿Cuándo no reintentar?

