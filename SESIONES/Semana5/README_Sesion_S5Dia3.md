# Día 3 — Dockerfile, Docker Compose y configuración segura

## Objetivos

Entregar la aplicación de las primeras cinco semanas de forma reproducible. Estudiar [Dockerfile](../../COMPLEMENTOS/README_DockerFile.md) e [Infraestructura local](../../COMPLEMENTOS/README_Infraestructura_Local.md).

## Dockerfile

- build multi-stage usando `./mvnw clean package`;
- runtime JRE compatible;
- usuario no-root;
- capas que aprovechen cache;
- `.dockerignore` sin `.git`, `.env`, logs ni `target` local;
- healthcheck y graceful shutdown.

## Docker Compose

Levantar aplicación y PostgreSQL con healthchecks, red, volumen y variables. Redis/Kafka permanecen en perfiles desactivados hasta Semana 6.

```bash
docker compose config
docker compose up --build -d
docker compose ps
```

## Configuración

Perfiles `dev`, `test`, `prod`; `@ConfigurationProperties` validada; `.env.example` sin secretos reales. Cambiar credenciales no debe requerir reconstruir la imagen.

## Pruebas

Arranque desde equipo limpio, migraciones Flyway, health, persistencia tras reinicio y SIGTERM. No usar `latest` ni ejecutar como root.

## Entregable

Imagen, Compose base, guía de ejecución/parada/limpieza y matriz de variables/puertos.
