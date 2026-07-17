# Día 3 — Rendimiento y observabilidad de Redis

## Objetivos
Observar hits, misses, latencia, errores y riesgo de stampede.

## Integración Spring
- Actuator y Micrometer;
- métricas del cliente y timers de casos de uso;
- tags de baja cardinalidad;
- health indicator sin convertir una caché opcional en dependencia crítica.

## Laboratorio
Generar carga sobre catálogo, vaciar una clave, observar miss/reconstrucción y detener Redis para comprobar degradación.

## Retos
1. Evitar stampede con sincronización local o estrategia documentada.
2. Cambiar Redis por Valkey y ejecutar las mismas pruebas.
3. Establecer presupuesto de rendimiento, no solo porcentaje de hits.

## Criterios
Estrategia 30%, consistencia 30%, pruebas 25%, observabilidad 15%.

