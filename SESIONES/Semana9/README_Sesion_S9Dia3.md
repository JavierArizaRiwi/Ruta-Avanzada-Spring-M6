# Día 3 — Contratos, particiones y operación de Kafka

## Objetivos
Evolucionar eventos y observar consumer groups.

## Laboratorio
Agregar `ModuleCompletedV1`, elegir clave por inscripción y ejecutar dos consumidores del mismo grupo. Comparar con grupos distintos.

## Contratos
- cambios aditivos antes que rupturas;
- nombre/version explícitos;
- consumidor tolerante a campos desconocidos;
- no reutilizar el mismo tipo para comando y evento.

## Operación
Observar particiones, offsets y lag desde CLI/UI. Explicar por qué un broker y réplica 1 son solo educativos.

## Reto
Diseñar V2 incompatible, plan de convivencia V1/V2 y prueba de contrato.

## Criterios
Contrato 30%, particionado 25%, consumidor 25%, operación/pruebas 20%.

