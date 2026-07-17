# Día 2 — TTL, invalidación y consistencia de caché

## Objetivos
Diseñar claves, TTL e invalidación alrededor de cambios de negocio.

## Problema
Al editar o despublicar una ruta, el catálogo cacheado no puede seguir mostrándola indefinidamente.

## Implementación
- `@CacheEvict` después de una transacción exitosa;
- TTL diferente para catálogo y resumen de progreso;
- claves `learning-route:v1:*` y `progress-summary:v1:*`;
- no usar `@CachePut` si obliga a duplicar lógica compleja.

## Laboratorio
Implementar actualización de ruta, invalidación y reconstrucción. Simular dos instancias de aplicación compartiendo Redis.

## Pruebas
Validar dato anterior, escritura, evicción y dato nuevo. Controlar el tiempo sin `Thread.sleep` frágil.

## Discusión
Comparar consistencia fuerte, eventual y “stale while revalidate”. Explicar el costo de cada opción.

