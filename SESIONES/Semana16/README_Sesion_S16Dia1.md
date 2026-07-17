# Día 1 — CI con Java 17 y Java 21

## Objetivos
Automatizar compilación, pruebas y cobertura en ambos JDK.

## Pipeline
- Maven Wrapper y cache local del runner;
- matriz 17/21;
- `./mvnw clean verify`;
- Testcontainers para PostgreSQL, Redis y Kafka;
- JaCoCo y reportes;
- permisos mínimos.

## Laboratorio
Crear GitHub Actions, romper una regla y comprobar que CI bloquee. Java 17 valida compatibilidad; Java 21 valida runtime recomendado.

## Antipatrones
Comandos distintos a local, tests ignorados o secretos impresos.

