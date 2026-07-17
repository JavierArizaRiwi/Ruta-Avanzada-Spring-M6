# Día 2 — Docker Compose por perfiles

## Objetivos
Orquestar aplicación, PostgreSQL, Redis y Kafka sin exigir todo desde el inicio.

## Contenido
Redes, DNS por nombre, volúmenes, healthchecks, `depends_on` limitado, variables y perfiles. Consultar [Infraestructura local](../../COMPLEMENTOS/README_Infraestructura_Local.md).

## Laboratorio
Crear perfiles `base`, `events` y `observability`. La aplicación usa `postgres`, `redis` y `kafka:29092` dentro de la red.

## Pruebas
Arranque desde cero, reinicio, persistencia, caída de dependencia y `docker compose config`.

## Regla
Compose es entorno local, no una descripción automática de producción.

