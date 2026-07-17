# Día 1 — Dockerfile para Spring Boot

## Objetivos
Crear una imagen reproducible, pequeña y no-root.

## Contenido
Usar [Dockerfile](../../COMPLEMENTOS/README_DockerFile.md): build multi-stage con Maven Wrapper, capas, `.dockerignore`, runtime JRE, usuario sin privilegios y graceful shutdown.

## Laboratorio
Construir la API con Java 17, ejecutarla sobre runtime 21 solo como comparación y verificar healthcheck.

## Pruebas
Inspeccionar usuario, tamaño, variables y señal SIGTERM. Confirmar que la imagen no contiene `.git`, `.env` ni cache Maven.

## Antipatrones
Tag `latest`, root, secreto como `ARG/ENV`, descargar dependencias en cada capa o copiar todo el workspace.

