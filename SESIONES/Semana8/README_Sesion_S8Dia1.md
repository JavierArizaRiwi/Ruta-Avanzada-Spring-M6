# Día 1 — Spring Cache y Redis

## Objetivos
Integrar Redis con Spring Boot sin convertirlo en fuente de verdad. Antes de iniciar, estudiar [Redis y Valkey](../../COMPLEMENTOS/README_Redis_Valkey.md).

## Conceptos Spring
- `spring-boot-starter-data-redis` y Spring Cache;
- `@EnableCaching`, `@Cacheable` y `CacheManager`;
- configuración tipada de TTL y serialización;
- fallback cuando Redis no está disponible.

## Laboratorio
Cachear el catálogo de rutas publicadas. Medir primero PostgreSQL, luego activar cache-aside y demostrar hit/miss sin cambiar el contrato REST.

## Regla
La caché solo contiene proyecciones reconstruibles. Una caída de Redis no debe perder inscripciones.

## Evidencias
Pruebas unitarias del caso de uso, integración con Redis Testcontainers y comparación de latencia.

## Errores frecuentes
Cachear entidades JPA, usar claves implícitas difíciles de migrar o afirmar una mejora sin medir.

