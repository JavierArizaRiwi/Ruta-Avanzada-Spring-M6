# Día 2 — RestClient, timeouts y Resilience4j

## Objetivos
Hacer llamadas HTTP remotas con fallos controlados.

## Contenido
- `RestClient` para flujo MVC imperativo;
- connect/read timeout;
- retry solo para operación idempotente/transitoria;
- circuit breaker, bulkhead y rate limiter;
- presupuesto de latencia y fallback honesto.

## Laboratorio
Consultar un resumen de progreso remoto. Introducir latencia/errores, abrir el circuito y demostrar recuperación.

## Regla
`WebClient` se usa si hay modelo reactivo justificado; no por ser más nuevo. Retry sin timeout multiplica una caída.

## Pruebas
WireMock, reloj controlado y assertions sobre estados del circuito.

