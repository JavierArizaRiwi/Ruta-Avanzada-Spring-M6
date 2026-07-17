# Día 3 — Testing profesional y calidad

## Objetivos

Construir una estrategia de pruebas rápida y confiable sobre los módulos de rutas e inscripciones.

## Pirámide aplicada

1. Dominio: JUnit 5, sin Spring.
2. Caso de uso: Mockito solo para puertos.
3. Web: `@WebMvcTest`, Security y contrato RFC 9457.
4. Persistencia: PostgreSQL Testcontainers y migraciones Flyway.
5. Integración crítica: contexto completo únicamente cuando aporta confianza adicional.

## Reglas a probar

- una ruta sin módulos no se publica;
- un coder no se inscribe dos veces en la misma ruta activa;
- una nota respeta su rango;
- una entrega posterior a la fecha queda tardía;
- dos solicitudes concurrentes no rompen la unicidad.

## Calidad

- JaCoCo mide líneas/ramas, pero no reemplaza escenarios;
- nombres Given/When/Then o conductuales;
- reloj inyectable para fechas;
- fixtures pequeños y builders de test;
- ArchUnit o Spring Modulith opcional para dependencias entre módulos;
- análisis estático local sin convertir una herramienta en objetivo pedagógico.

## Laboratorio

Crear una prueba por nivel, introducir un defecto real y comprobar qué prueba lo detecta. Ejecutar:

```bash
./mvnw clean verify
```

## Evidencias

Reporte JaCoCo, tiempos por suite, matriz de escenarios y justificación de exclusiones. La cobertura se evalúa especialmente sobre dominio y casos de uso.

## Antipatrones

Mockear entidades, levantar contexto para toda prueba, `Thread.sleep`, depender del orden o perseguir 100% global.

## Criterios

Escenarios 40%, integración real 25%, mantenibilidad 20%, arquitectura/cobertura razonada 15%.
