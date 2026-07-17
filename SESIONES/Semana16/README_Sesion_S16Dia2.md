# Día 2 — Calidad, seguridad e imagen

## Objetivos
Producir un artefacto trazable sin desplegar automáticamente.

## Contenido
- análisis estático y dependencias;
- escaneo de secretos;
- validación de migraciones;
- build de imagen una vez;
- SBOM, tags inmutables y digest;
- estrategia de actualización.

## Laboratorio
Construir imagen solo después de matriz verde, generar reporte y demostrar que un secreto de prueba controlado es detectado sin subirlo al repositorio.

## Regla
No publicar imágenes desde pull requests no confiables ni usar credenciales de larga duración.

