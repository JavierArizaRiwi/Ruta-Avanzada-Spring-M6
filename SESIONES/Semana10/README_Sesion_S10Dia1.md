# Día 1 — El problema de escritura dual

## Objetivos
Comprender por qué guardar en PostgreSQL y publicar en Kafka son dos operaciones independientes.

## Escenarios
1. La base confirma y Kafka falla.
2. Kafka confirma y la transacción de base se revierte.
3. El productor reintenta y duplica.

## Laboratorio
Implementar deliberadamente dual write, inyectar fallos y registrar inconsistencias. No “resolver” con un `try/catch` genérico.

## Resultado
ADR que compare publicación directa, evento después de commit, transacción distribuida y transactional outbox.

## Preguntas
¿Qué garantía necesita el negocio? ¿Por qué `@Transactional` no incluye Kafka automáticamente?

