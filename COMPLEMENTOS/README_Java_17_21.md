# Java 17 y Java 21 para la ruta Spring

## Propósito

Todos los ejercicios obligatorios compilan con Java 17. Java 21 es el runtime recomendado y se usa para comparar mejoras sin romper la base.

## Política Maven

```xml
<properties>
  <java.version>17</java.version>
  <maven.compiler.release>17</maven.compiler.release>
</properties>
```

Validación:

```bash
java -version
./mvnw -version
./mvnw clean verify
```

## Características disponibles en la base

| Característica | Disponible | Uso recomendado |
| --- | --- | --- |
| Records | Java 16+ | DTO y eventos inmutables; no entidades JPA |
| Text blocks | Java 15+ | JSON, SQL y fixtures de prueba |
| Switch expressions | Java 14+ | Decisiones sencillas y exhaustivas |
| Sealed classes | Java 17 | Jerarquías cerradas cuando el dominio lo justifique |
| Pattern matching `instanceof` | Java 16+ | Evitar casts repetitivos |

## Características específicas de Java 21

- Pattern matching para `switch`: actividad opcional con alternativa Java 17.
- Virtual threads: útiles para mucha concurrencia con I/O bloqueante; no aceleran CPU ni eliminan límites de conexiones.
- Mejoras de runtime y garbage collectors: medir antes de afirmar una ganancia.

No se usan APIs preview en actividades obligatorias.

## Laboratorio comparativo

1. Ejecutar la misma suite con JDK 17 y 21.
2. Verificar que `release=17` rechace una API exclusiva de 21.
3. Comparar un endpoint MVC bloqueante con y sin virtual threads bajo carga local.
4. Documentar threads, latencia, conexiones y límites; no quedarse solo con requests/segundo.

## Errores frecuentes

- Confundir el JDK del IDE con el de la terminal.
- Cambiar `source/target` sin usar `release`.
- Usar virtual threads sobre un pool de base de datos sin controlar concurrencia.
- Convertir todas las clases en records, incluidas entidades con ciclo de vida mutable.

## Recursos oficiales

- [Java 17](https://docs.oracle.com/en/java/javase/17/)
- [Java 21](https://docs.oracle.com/en/java/javase/21/)

