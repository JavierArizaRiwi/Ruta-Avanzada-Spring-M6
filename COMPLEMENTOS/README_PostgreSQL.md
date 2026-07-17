# PostgreSQL para desarrolladores Java

## Qué es y por qué se usa

PostgreSQL es la base relacional principal de la ruta. Esta guía explica la tecnología; Spring Data JPA y Flyway se aplican en `SESIONES/Semana3`.

## Conceptos esenciales

- base, esquema, tabla, constraint e índice;
- clave primaria, foránea y unicidad;
- ACID, aislamiento, MVCC y locks;
- plan de ejecución con `EXPLAIN (ANALYZE, BUFFERS)`;
- tipos `uuid`, `timestamp with time zone`, `jsonb` y arrays, evitando usarlos sin necesidad.

## Ejecución local

```yaml
services:
  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: riwi_learning
      POSTGRES_USER: riwi
      POSTGRES_PASSWORD: riwi_local_only
    ports: ["5432:5432"]
    volumes: ["postgres_data:/var/lib/postgresql"]
volumes:
  postgres_data:
```

La contraseña es solo para red local. En un proyecto real se inyecta desde el entorno.

## Reglas del caso conductor

- índice para búsquedas de rutas publicadas;
- constraint para rango de nota;
- unicidad de inscripción activa, modelada de forma compatible con el negocio;
- locking optimista para actualizaciones concurrentes;
- auditoría temporal en UTC.

## Cuándo no confiar solo en JPA

Usar SQL explícito, JDBC o una herramienta especializada cuando una consulta masiva, reporte, CTE o función de ventana sea más clara. ORM no reemplaza conocimientos de SQL.

## Recursos oficiales

- [PostgreSQL](https://www.postgresql.org/docs/)

