# Día 3 — Validación, Problem Details RFC 9457 y OpenAPI

> **Estado curricular:** cierre de API en Semana 2. Incluye validación, RFC 9457, documentación OpenAPI y pruebas del contrato.

## Entregable adicional: OpenAPI

Documenta operaciones, estados, DTO y Problem Details mediante springdoc-openapi compatible con Spring Boot 4. La documentación no reemplaza las pruebas: añade `@WebMvcTest` para 201, 400, 404 y 409 y verifica `application/problem+json`.

Endpoints esperados:

```text
/v3/api-docs
/swagger-ui/index.html
```

Al finalizar la semana deben existir una colección `.http` y un contrato OpenAPI reproducible.

En esta sesión aprenderás a construir un **sistema robusto de validaciones y manejo de errores** en APIs REST usando **Spring Boot**, bajo principios de **arquitectura limpia/hexagonal**.
El objetivo es que tus adaptadores de entrada (controladores) sean **limpios**, sin lógica de validación o control de errores repetida, y que el **dominio** permanezca independiente del framework.

> Nota: en semanas siguientes, estos endpoints se expondrán **a través del API Gateway**; el formato unificado de errores ayuda a clientes y observabilidad.

---

## 1) Objetivos del día

- Implementar **validaciones personalizadas** con `jakarta.validation`.
- Centralizar el manejo de errores mediante `@ControllerAdvice` / `@RestControllerAdvice`.
- Definir una **estructura uniforme** de respuesta de error.
- Diferenciar entre errores de **dominio**, **infraestructura** y **validación**.
- Mantener el **desacoplamiento** entre dominio y framework (hexagonal).

---

## 2) Por qué manejar errores de forma uniforme

En proyectos empresariales es vital que todas las respuestas de error tengan un formato predecible y homogéneo, por ejemplo:

```json
{
  "timestamp": "2025-10-20T14:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "El campo 'nombre' es obligatorio",
  "path": "/api/v1/estudiantes",
  "fields": { "nombre": "El nombre no puede estar vacío" }
}
```

Esto facilita:
- Integración con frontends y sistemas externos.
- Depuración centralizada y logging.
- Mantenibilidad, consistencia y pruebas automatizadas (contratos).

---

## 3) Estructura de proyecto (hexagonal)

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/
 │   ├─ exception/            # Excepciones de dominio (p.ej., NegocioException)
 │   ├─ ports/
 │   └─ service/
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/          # Controladores REST (sin lógica de validación de negocio)
 │   │   └─ out/jpa/          # Persistencia (implementaciones de puertos)
 │   └─ exception/            # Manejador global + excepciones técnicas
 ├─ dto/                      # DTOs request/response + ErrorResponse
 ├─ application/              # (opcional) orquestación
 └─ AcademicoApplication.java
```

---

## 4) Validaciones con `jakarta.validation`

Spring Boot integra automáticamente **Bean Validation**.
Anota tus **DTOs** y usa `@Valid` (cuerpo) y `@Validated` (params/path).

### 4.1 DTO con validaciones
```java
// dto/EstudianteRequest.java
package com.riwi.academico.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class EstudianteRequest {
  @NotBlank(message = "El nombre no puede estar vacío")
  @Size(min = 3, max = 50, message = "El nombre debe tener entre 3 y 50 caracteres")
  private String nombre;

  public String getNombre() { return nombre; }
  public void setNombre(String nombre) { this.nombre = nombre; }
}
```

### 4.2 Controlador con validación automática
```java
// infrastructure/adapters/in/rest/EstudianteController.java (fragmento)
@PostMapping
public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest request) {
  var e = service.ejecutar(request.getNombre());           // caso de uso en dominio
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(new EstudianteResponse(e.getId(), e.getNombre()));
}
```

Si la validación falla, Spring lanza `MethodArgumentNotValidException` (cuerpo).
Para **params/path**, anota la clase con `@Validated` y usa restricciones como `@Min`, `@Max`, etc.

---

## 5) Contrato de error: `ProblemDetail` RFC 9457

La implementación obligatoria usa el soporte nativo de Spring. El DTO `ErrorResponse` mostrado después se conserva únicamente para comparar un contrato propietario; no debe coexistir con `ProblemDetail` en la misma API.

```java
@RestControllerAdvice
class ApiExceptionHandler {

  @ExceptionHandler(NegocioException.class)
  ProblemDetail business(NegocioException exception, HttpServletRequest request) {
    var problem = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
    problem.setType(URI.create("https://errors.riwi.io/business-rule"));
    problem.setTitle("No se puede procesar la operación");
    problem.setDetail(exception.getMessage());
    problem.setInstance(URI.create(request.getRequestURI()));
    problem.setProperty("code", "BUSINESS_RULE_VIOLATION");
    return problem;
  }
}
```

Para validación añade una propiedad `errors` con campo y mensaje, sin devolver valores sensibles. El media type esperado es `application/problem+json`.

### Comparación: contrato propietario anterior

```java
// dto/ErrorResponse.java
package com.riwi.academico.dto;

import java.time.Instant;
import java.util.Map;

public class ErrorResponse {
  private String timestamp;
  private int status;
  private String error;
  private String message;
  private String path;
  private Map<String, String> fields; // opcional para errores de validación de campos

  public ErrorResponse() {}
  public ErrorResponse(int status, String error, String message, String path, Map<String,String> fields) {
    this.timestamp = Instant.now().toString();
    this.status = status;
    this.error = error;
    this.message = message;
    this.path = path;
    this.fields = fields;
  }
  public String getTimestamp() { return timestamp; }
  public int getStatus() { return status; }
  public String getError() { return error; }
  public String getMessage() { return message; }
  public String getPath() { return path; }
  public Map<String, String> getFields() { return fields; }
}
```

---

## 6) Manejador global de errores (`@RestControllerAdvice`)

```java
// infrastructure/exception/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.exception;

import com.riwi.academico.dto.ErrorResponse;
import com.riwi.academico.domain.exception.NegocioException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.ConstraintViolationException;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class) // body inválido
  public ResponseEntity<ErrorResponse> handleBodyValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
    Map<String, String> fields = ex.getBindingResult().getFieldErrors().stream()
        .collect(Collectors.toMap(
            fe -> fe.getField(),
            fe -> fe.getDefaultMessage(),
            (a,b) -> a,
            LinkedHashMap::new
        ));
    var body = new ErrorResponse(400, "Bad Request", "Validación fallida", req.getRequestURI(), fields);
    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(ConstraintViolationException.class) // params/path inválidos
  public ResponseEntity<ErrorResponse> handleParamValidation(ConstraintViolationException ex, HttpServletRequest req) {
    Map<String, String> fields = ex.getConstraintViolations().stream()
        .collect(Collectors.toMap(
            v -> v.getPropertyPath().toString(),
            v -> v.getMessage(),
            (a,b) -> a,
            LinkedHashMap::new
        ));
    var body = new ErrorResponse(400, "Bad Request", "Parámetros inválidos", req.getRequestURI(), fields);
    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(NegocioException.class) // dominio
  public ResponseEntity<ErrorResponse> handleBusiness(NegocioException ex, HttpServletRequest req) {
    var body = new ErrorResponse(422, "Unprocessable Entity", ex.getMessage(), req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(body);
  }

  @ExceptionHandler(IllegalArgumentException.class) // conflictos de reglas
  public ResponseEntity<ErrorResponse> handleConflict(IllegalArgumentException ex, HttpServletRequest req) {
    var body = new ErrorResponse(409, "Conflict", ex.getMessage(), req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
  }

  @ExceptionHandler(java.util.NoSuchElementException.class) // recurso inexistente
  public ResponseEntity<ErrorResponse> handleNotFound(java.util.NoSuchElementException ex, HttpServletRequest req) {
    var body = new ErrorResponse(404, "Not Found", ex.getMessage(), req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
  }

  @ExceptionHandler(Exception.class) // fallback
  public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest req) {
    var body = new ErrorResponse(500, "Internal Server Error", "Ocurrió un error inesperado", req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
  }
}
```

---

## 7) Excepciones personalizadas del dominio

```java
// domain/exception/NegocioException.java
package com.riwi.academico.domain.exception;

public class NegocioException extends RuntimeException {
  public NegocioException(String message) { super(message); }
}
```

Puedes añadir otras (p.ej., `DuplicadoException`, `ReglaDominioException`) manteniendo el **dominio** libre de dependencias de Spring.

---

## 8) Validaciones personalizadas (`@Constraint`)

A veces se requieren reglas propias, por ejemplo validar que un nombre **no contenga números**.

### 8.1 Anotación
```java
// infrastructure/validation/NoNumeros.java
package com.riwi.academico.infrastructure.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = NoNumerosValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NoNumeros {
  String message() default "El texto no debe contener números";
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};
}
```

### 8.2 Validador
```java
// infrastructure/validation/NoNumerosValidator.java
package com.riwi.academico.infrastructure.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class NoNumerosValidator implements ConstraintValidator<NoNumeros, String> {
  @Override
  public boolean isValid(String value, ConstraintValidatorContext context) {
    return value != null && value.matches("^[^0-9]*$");
  }
}
```

### 8.3 Uso en DTO
```java
// dto/EstudianteRequest.java (fragmento)
@NoNumeros
private String nombre;
```

---

## 9) Testing del manejo de errores con MockMvc

```java
// src/test/java/com/riwi/academico/infrastructure/adapters/in/rest/EstudianteControllerValidationTest.java
@WebMvcTest(EstudianteController.class)
class EstudianteControllerValidationTest {

  @Autowired private MockMvc mvc;
  @MockitoBean private RegistrarEstudianteService service;

  @Test
  void debeRetornarErrorDeValidacion() throws Exception {
    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"nombre":"12"}"))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.fields.nombre").value("El texto no debe contener números"));
  }
}
```

---

## 10) Buenas prácticas

| Práctica | Beneficio |
|-----------|-----------|
| DTOs validados (`@Valid`) | Evita lógica de validación en el dominio |
| `@RestControllerAdvice` global | Código limpio y reutilizable |
| Excepciones por capa | Depuración y responsabilidades claras |
| Mensajes sin detalles internos | Seguridad del API |
| Formato homogéneo de error | Mejor DX y pruebas automatizadas |

---

## 11) IntelliJ / Ejecución rápida

1. **Dependencias:** `spring-boot-starter-validation`, `spring-boot-starter-web`, `spring-boot-starter-test`.
2. **Annotation Processors:** habilitar en *Settings → Build → Compiler → Annotation Processors*.
3. **Ejecutar pruebas:** `./mvnw -q -Dtest=*Validation* test` o desde el IDE.
4. **Probar con curl:**
   ```bash
   curl -X POST http://localhost:8080/api/v1/estudiantes      -H "Content-Type: application/json"      -d '{"nombre": ""}'
   ```

---

## 12) Resultado esperado

- Validaciones automáticas en todos los endpoints.
- Manejador global centralizado y extensible.
- Respuestas RFC 9457 con `type`, `title`, `status`, `detail`, `instance`, `code` y errores de campo cuando corresponda.
- Validaciones personalizadas funcionales.
- Pruebas verificando el flujo de errores.
