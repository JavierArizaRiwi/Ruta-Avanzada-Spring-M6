# Día 1 — Identidad y autorización distribuidas

## Objetivos
Separar autenticación de usuario, autorización de recurso e identidad de servicio.

## Contenido
- issuer, audience, scopes y expiración;
- Gateway valida inicialmente, servicio vuelve a autorizar;
- propagación mínima de identidad;
- refresh token solo en componente de identidad;
- OAuth2/OIDC y Authorization Server como módulo avanzado.

## Laboratorio
Proteger endpoint administrativo de notificaciones y rechazar token con audience incorrecta incluso al acceder directamente.

## Antipatrones
Token en evento Kafka, JWT eterno, secreto compartido sin rotación o “red interna confiable”.

