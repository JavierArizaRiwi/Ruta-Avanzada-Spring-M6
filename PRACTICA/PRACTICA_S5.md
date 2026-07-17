# Práctica Semana 5 — Seguridad y entrega reproducible

## Objetivo

Proteger Riwi Learning Platform y entregarla con Docker Compose sin secretos en el repositorio.

## Historias

- Un coder inicia sesión y consulta únicamente su progreso.
- Un trainer califica actividades asignadas.
- Un admin publica rutas.
- Un refresh token rotado no puede reutilizarse.
- La aplicación y PostgreSQL arrancan desde cero con Compose y Flyway.

## Entregables

1. `SecurityFilterChain`, BCrypt y autorización por método/recurso.
2. Access token y refresh token revocable.
3. Pruebas 401, 403, expiración, audience, rol y propietario.
4. Dockerfile multi-stage/no-root.
5. Compose de aplicación + PostgreSQL con healthchecks.
6. `.env.example`, matriz de variables y README de operación.

## Restricciones

- Sin secretos, tokens o contraseñas reales en Git.
- Sin `ddl-auto=update` ni imágenes `latest`.
- Sin lógica de permisos únicamente en frontend o Gateway.
- Cambiar configuración no debe requerir reconstruir imagen.

## Evaluación

Seguridad 40%, pruebas 25%, Docker/Compose 25%, documentación 10%.
