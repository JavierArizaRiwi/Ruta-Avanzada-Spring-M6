# Práctica Semana 4 — De capas a monolito modular

> Práctica alineada con las sesiones de [Semana 4](../SESIONES/Semana4/).

## Contexto

Riwi Learning Platform tiene controladores, servicios y repositorios globales. Inscripciones conoce directamente detalles internos de rutas y usuarios. Cada cambio obliga a modificar varias carpetas técnicas.

## Objetivo

Refactorizar el flujo de inscripción hacia módulos por funcionalidad sin cambiar el contrato HTTP ni dividir el sistema en microservicios.

## Reglas obligatorias

- Un coder no puede tener dos inscripciones activas en la misma ruta.
- La ruta debe estar publicada y tener cupo.
- Una inscripción confirmada genera un evento interno.
- El controlador no accede a repositorios.
- La restricción crítica también existe en PostgreSQL.

## Entregables

1. Diagrama antes/después.
2. Paquetes `learningroutes` y `enrollments` con API interna explícita.
3. Caso de uso probado sin levantar todo el contexto.
4. Adaptador JPA con PostgreSQL Testcontainers.
5. ADR que compare capas, monolito modular y hexagonal.
6. Regla ArchUnit o Spring Modulith opcional para impedir dependencia inversa.

## Restricciones

- No crear un microservicio.
- No compartir entidades JPA como contrato entre módulos.
- No crear interfaces sin explicar la alternativa que habilitan.
- No introducir Kafka todavía; el evento interno prepara la evolución posterior.

## Pruebas mínimas

- inscripción válida;
- duplicado activo rechazado;
- ruta sin cupo rechazada;
- fallo concurrente protegido por restricción/locking;
- contrato HTTP sin regresión.

## Evaluación

| Criterio | Peso |
| --- | ---: |
| Reglas y consistencia | 35% |
| Límites modulares | 25% |
| Pruebas | 25% |
| ADR y defensa técnica | 15% |
