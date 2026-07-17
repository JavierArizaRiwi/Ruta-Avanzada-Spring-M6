# Día 1 — Spring Cache y Spring Data Redis

## Objetivos

Aplicar caché a un caso medido, con TTL e invalidación. Antes de integrar Spring, estudiar [Redis y Valkey](../../COMPLEMENTOS/README_Redis_Valkey.md).

## Problema

El catálogo de rutas y los resúmenes de progreso se consultan con frecuencia. PostgreSQL continúa siendo fuente de verdad.

## Implementación

- starter de Spring Data Redis gestionado por Boot;
- Spring Cache y `CacheManager` con TTL por caché;
- `@Cacheable` para lectura y `@CacheEvict` después de escritura confirmada;
- claves versionadas y serialización explícita;
- métricas de hit/miss/latencia;
- degradación cuando Redis es opcional.

## Laboratorio

Activar Redis en Compose, medir consulta sin caché, cachear catálogo, modificar una ruta e invalidar. Simular dos instancias compartiendo Redis.

## Pruebas

Redis Testcontainers: miss, hit, evicción, expiración y caída. Evitar `Thread.sleep` frágil y entidades JPA cacheadas.

## Criterios

Estrategia 30%, consistencia 30%, pruebas 25%, métricas 15%.
