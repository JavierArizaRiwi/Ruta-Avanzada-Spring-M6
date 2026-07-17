# Día 2 — JWT profesional, refresh token y seguridad web

## Objetivos

Completar la seguridad iniciada el Día 1: access token corto, refresh token revocable, autorización por recurso y pruebas de amenazas comunes.

## Contenido

- claims mínimos: `sub`, `iss`, `aud`, `iat`, `exp`, roles/scopes;
- access token frente a refresh token;
- rotación y revocación persistida;
- CORS según clientes permitidos;
- CSRF: cuándo deshabilitarlo en API stateless y cuándo conservarlo;
- 401 frente a 403;
- seguridad a nivel de método y propiedad del recurso;
- sesiones como alternativa válida a JWT.

## Laboratorio

Implementar login, refresh rotatorio y logout. `TRAINER` califica únicamente actividades asignadas; `CODER` consulta solo su progreso; `ADMIN` administra rutas.

## Pruebas

- credencial inválida y usuario bloqueado;
- token expirado, firma/audience inválida;
- refresh reutilizado después de rotación;
- acceso sin rol y acceso a recurso ajeno;
- CORS permitido/rechazado.

## Seguridad

El secreto o clave privada se obtiene del entorno. No registrar tokens ni almacenar contraseñas sin BCrypt. Consultar OWASP API Security y documentar limitaciones de JWT.

## Criterios

Autenticación 30%, autorización 30%, ciclo de tokens 20%, pruebas/amenazas 20%.
