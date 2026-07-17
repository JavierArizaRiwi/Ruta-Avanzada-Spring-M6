# Respuestas profesionales en APIs Spring Boot

> **Estado curricular:** guía consolidada de Semanas 2–3. Sustituye las dos versiones duplicadas anteriores y usa RFC 9457.

## Principios

1. HTTP ya ofrece estados, headers y negociación de contenido.
2. DTO de API y modelo interno son contratos diferentes.
3. Los errores usan `application/problem+json` y `ProblemDetail`.
4. La respuesta no revela stack traces, SQL ni datos sensibles.
5. Un correlation ID ayuda a buscar la operación, pero no se usa como tag de métrica de alta cardinalidad.

## Respuestas de éxito

No se impone un envelope universal. Para un recurso simple se devuelve el DTO; para una colección paginada se incluye metadata estable.

```java
record LearningRouteResponse(UUID id, String name, String status) {}

record PageMetadata(int number, int size, long totalElements, int totalPages) {}

record PageResponse<T>(List<T> content, PageMetadata page) {}
```

Reglas orientativas:

| Operación | Estado | Respuesta |
| --- | ---: | --- |
| Consulta encontrada | 200 | DTO |
| Creación | 201 | DTO + `Location` |
| Actualización completa | 200/204 | DTO o sin cuerpo, de forma consistente |
| Eliminación sin cuerpo | 204 | Sin body |
| Solicitud asíncrona aceptada | 202 | Estado/URL de seguimiento |

```java
@PostMapping
ResponseEntity<LearningRouteResponse> create(@Valid @RequestBody CreateRouteRequest request) {
    var created = createRoute.execute(request);
    var location = URI.create("/api/v1/learning-routes/" + created.id());
    return ResponseEntity.created(location).body(created);
}
```

No devolver `200 OK` para cualquier resultado ni incluir body en una respuesta 204.

## Paginación y filtros

- Limitar `size` para proteger base de datos y memoria.
- Permitir solo campos de ordenamiento conocidos.
- Documentar si la página comienza en cero.
- Para datasets que cambian rápidamente, evaluar cursor pagination.
- Evitar ejecutar una consulta N+1 por cada elemento del listado.

## Problem Details RFC 9457

Campos estándar:

- `type`: URI estable que identifica la categoría.
- `title`: resumen legible.
- `status`: estado HTTP.
- `detail`: explicación segura de esta ocurrencia.
- `instance`: URI de la solicitud/ocurrencia.

Se pueden añadir `code`, `correlationId` y `errors`, manteniendo nombres documentados.

```java
@RestControllerAdvice
final class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(DuplicateActiveEnrollmentException.class)
    ResponseEntity<ProblemDetail> duplicate(
            DuplicateActiveEnrollmentException exception,
            HttpServletRequest request) {

        var problem = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        problem.setType(URI.create("https://errors.riwi.io/duplicate-active-enrollment"));
        problem.setTitle("La inscripción activa ya existe");
        problem.setDetail(exception.getMessage());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "ENROLLMENT_ALREADY_ACTIVE");
        return ResponseEntity.status(problem.getStatus()).body(problem);
    }
}
```

Configurar el soporte nativo para excepciones de Spring según la versión de Boot y personalizar mensajes para no filtrar implementación.

## Mapeo de errores

| Situación | Estado recomendado |
| --- | ---: |
| JSON/constraint inválido | 400 |
| Credencial ausente o inválida | 401 |
| Identidad sin permiso | 403 |
| Recurso inexistente | 404 |
| Duplicado/conflicto de estado | 409 |
| Media type no soportado | 415 |
| Regla semántica no procesable | 422, si el equipo adopta esta convención |
| Rate limit | 429 |
| Fallo inesperado | 500 con detalle genérico |

No usar 404 para ocultar indiscriminadamente errores internos ni 500 para reglas esperadas.

## Validación

```java
record CreateActivityRequest(
    @NotBlank @Size(max = 120) String title,
    @NotNull UUID moduleId,
    @Future Instant dueAt
) {}
```

Bean Validation protege el borde. La regla “una actividad pertenece a un módulo existente” se resuelve en el caso de uso/dominio porque requiere estado.

Para errores por campo se puede agregar:

```json
{
  "type": "https://errors.riwi.io/validation",
  "title": "La solicitud contiene errores",
  "status": 400,
  "code": "VALIDATION_FAILED",
  "errors": [
    {"field": "title", "message": "no debe estar vacío"}
  ]
}
```

No devolver el valor rechazado si puede contener una contraseña, token o dato personal.

## Trazabilidad

Aceptar o generar `X-Correlation-Id`, validando longitud y caracteres antes de incorporarlo a logs/respuestas. En sistemas distribuidos, propagarlo en HTTP y metadata de Kafka. No sustituye OpenTelemetry ni W3C Trace Context.

## Versionado

Versionar solo cuando exista una ruptura que no pueda evolucionar compatiblemente. `/api/v1` es válido, pero no reemplaza una política de deprecación. Cambios aditivos suelen ser preferibles; consumidores no deben fallar por campos JSON desconocidos.

## Pruebas de contrato

```java
@WebMvcTest(LearningRouteController.class)
class LearningRouteControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean CreateLearningRouteUseCase useCase;

    @Test
    void createsWithLocation() throws Exception {
        // given: configurar caso de uso
        // when/then: validar 201, Location y JSON público
    }
}
```

Escenarios mínimos:

- 201 y header `Location`;
- 400 con lista de campos;
- 401 y 403 diferenciados;
- 404 y 409 con `type`/`code` estable;
- paginación y límite máximo;
- media type `application/problem+json`.

## Antipatrones

- `ApiResponse<T>` obligatorio incluso para 204 o Problem Details.
- Devolver `Map<String,Object>` desde cada controlador.
- Exponer entidades JPA o excepciones crudas.
- `try/catch` genérico por endpoint.
- Mensajes de error como único identificador de máquina.
- Inventar un status HTTP distinto para cada regla.

## Checklist

- [ ] DTO separados del modelo persistente.
- [ ] Estados/headers correctos.
- [ ] RFC 9457 consistente.
- [ ] Códigos de error documentados.
- [ ] Paginación limitada.
- [ ] Trazabilidad segura.
- [ ] OpenAPI y pruebas actualizadas.

## Recursos oficiales

- [Problem Details en Spring MVC](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
- [Spring MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html)
