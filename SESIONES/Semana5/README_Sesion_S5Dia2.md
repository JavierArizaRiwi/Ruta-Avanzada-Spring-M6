# Semana 5 – Día 2  
## Validación con Jakarta Validation y Manejo Global de Errores en Spring Boot **(Arquitectura Hexagonal + REST Adapter)**

En esta sesión reforzarás la calidad del contrato HTTP aplicando **validación automática** con `@Valid` y un **manejador global de excepciones** que devuelva errores consistentes, manteniendo el **dominio y los casos de uso libres de framework**.  
Continuamos con el dominio educativo de **Estudiantes** y reusamos la base del Día 1, ahora organizada en **puertos/adaptadores**. El tráfico externo en plataforma se hará por **API Gateway** en semanas siguientes; hoy trabajamos el **servicio en local**.

---

## 1) Objetivos del día

- Diseñar **DTOs** con anotaciones de **Jakarta Validation** y mensajes claros.  
- Activar validaciones en controladores con `@Valid` (cuerpo) y `@Validated` (query/path).  
- Implementar un **`@RestControllerAdvice`** para centralizar errores (400/404/409/500).  
- Definir un **formato estándar de error** para toda la API (adapter).  
- Probar errores comunes con **MockMvc** (opcional).

---

## 2) Dependencias necesarias (`pom.xml`)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## 3) Estructura de proyecto (hexagonal)

```
com.riwi.academico
 ├─ domain/                                   # Entidades y reglas puras
 ├─ application/
 │   ├─ ports/                                # Puertos (repos, cache, etc.)
 │   └─ usecase/                              # Casos de uso (servicios de aplicación)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/EstudianteController.java # REST (usa @Valid/@Validated)
 │   │   └─ out/persistence/...               # Repos/JPA/in-memory
 │   └─ config/                               # GlobalExceptionHandler, OpenAPI
 ├─ dto/                                      # DTOs request/response + ErrorResponse
 └─ AcademicoApplication.java
```

---

## 4) DTOs con validación y mensajes personalizados

```java
// dto/EstudianteRequest.java
package com.riwi.academico.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class EstudianteRequest {
  @NotBlank(message = "El nombre es obligatorio")
  @Size(min = 3, max = 50, message = "El nombre debe tener entre 3 y 50 caracteres")
  private String nombre;

  public String getNombre() { return nombre; }
  public void setNombre(String nombre) { this.nombre = nombre; }
}
```

> Opcional: centraliza mensajes en `src/main/resources/messages.properties` y define un `MessageSource` para i18n.

---

## 5) Caso de uso y puerto (application)

```java
// application/ports/EstudianteRepositoryPort.java
package com.riwi.academico.application.ports;

import com.riwi.academico.domain.Estudiante;
import java.util.List;
import java.util.Optional;

public interface EstudianteRepositoryPort {
  boolean existsByNombre(String nombre);
  Long nextId();
  Estudiante save(Estudiante e);
  Optional<Estudiante> findById(Long id);
  List<Estudiante> findAll();
}
```

```java
// application/usecase/EstudianteUseCase.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.application.ports.EstudianteRepositoryPort;
import com.riwi.academico.domain.Estudiante;

import java.util.List;
import java.util.NoSuchElementException;

public class EstudianteUseCase {

  private final EstudianteRepositoryPort repo;
  public EstudianteUseCase(EstudianteRepositoryPort repo) { this.repo = repo; }

  public Estudiante crear(String nombre) {
    if (repo.existsByNombre(nombre)) throw new IllegalArgumentException("El nombre ya existe");
    var e = new Estudiante(repo.nextId(), nombre);
    return repo.save(e);
  }

  public Estudiante obtener(Long id) {
    return repo.findById(id).orElseThrow(() -> new NoSuchElementException("Estudiante no encontrado"));
  }

  public List<Estudiante> listar() { return repo.findAll(); }
}
```

---

## 6) Dominio y DTOs de respuesta/errores

```java
// domain/Estudiante.java
package com.riwi.academico.domain;

public class Estudiante {
  private Long id;
  private String nombre;
  public Estudiante(Long id, String nombre){ this.id = id; this.nombre = nombre; }
  public Long getId(){ return id; }
  public String getNombre(){ return nombre; }
}
```

```java
// dto/EstudianteResponse.java
package com.riwi.academico.dto;

public class EstudianteResponse {
  private Long id; private String nombre;
  public EstudianteResponse(Long id, String nombre){ this.id=id; this.nombre=nombre; }
  public Long getId(){ return id; } public String getNombre(){ return nombre; }
}
```

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
  private Map<String, String> fields; // opcional para errores de validación

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

## 7) Adaptador de entrada (REST) con `@Valid`, `@Validated`, `@RequestParam`

```java
// infrastructure/adapters/in/rest/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in.rest;

import com.riwi.academico.application.usecase.EstudianteUseCase;
import com.riwi.academico.dto.EstudianteRequest;
import com.riwi.academico.dto.EstudianteResponse;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import org.springframework.http.*;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/estudiantes")
@Validated  // habilita validación en params/path variables
public class EstudianteController {

  private final EstudianteUseCase useCase;
  public EstudianteController(EstudianteUseCase useCase) { this.useCase = useCase; }

  @PostMapping
  public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest req) {
    var e = useCase.crear(req.getNombre());
    return ResponseEntity.status(HttpStatus.CREATED).body(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @GetMapping("/{id}")
  public ResponseEntity<EstudianteResponse> obtener(@PathVariable @Min(1) Long id) {
    var e = useCase.obtener(id);
    return ResponseEntity.ok(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @GetMapping
  public ResponseEntity<List<EstudianteResponse>> listar(
      @RequestParam(defaultValue = "0") @Min(value = 0, message = "page debe ser >= 0") int page,
      @RequestParam(defaultValue = "10") @Min(value = 1, message = "size debe ser >= 1") int size) {
    var res = useCase.listar().stream().map(e -> new EstudianteResponse(e.getId(), e.getNombre())).toList();
    return ResponseEntity.ok(res);
  }
}
```

---

## 8) Manejador global de excepciones (adapter)

```java
// infrastructure/config/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.dto.ErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.ConstraintViolationException;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.NoSuchElementException;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ErrorResponse> handleBodyValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
    Map<String, String> fields = ex.getBindingResult().getFieldErrors().stream()
        .collect(Collectors.toMap(
            fe -> fe.getField(),
            fe -> fe.getDefaultMessage(),
            (a,b) -> a,
            LinkedHashMap::new));

    var body = new ErrorResponse(400, "Bad Request", "Validación fallida", req.getRequestURI(), fields);
    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(ConstraintViolationException.class)
  public ResponseEntity<ErrorResponse> handleParamValidation(ConstraintViolationException ex, HttpServletRequest req) {
    Map<String, String> fields = ex.getConstraintViolations().stream()
        .collect(Collectors.toMap(
            v -> v.getPropertyPath().toString(),
            v -> v.getMessage(),
            (a,b) -> a,
            LinkedHashMap::new));

    var body = new ErrorResponse(400, "Bad Request", "Parámetros inválidos", req.getRequestURI(), fields);
    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<ErrorResponse> handleConflict(IllegalArgumentException ex, HttpServletRequest req) {
    var body = new ErrorResponse(409, "Conflict", ex.getMessage(), req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
  }

  @ExceptionHandler(NoSuchElementException.class)
  public ResponseEntity<ErrorResponse> handleNotFound(NoSuchElementException ex, HttpServletRequest req) {
    var body = new ErrorResponse(404, "Not Found", ex.getMessage(), req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest req) {
    var body = new ErrorResponse(500, "Internal Server Error", "Ocurrió un error inesperado", req.getRequestURI(), null);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
  }
}
```

---

## 9) Ejemplos de respuestas de error

Validación fallida (400):
```json
{
  "timestamp": "2025-10-21T15:00:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validación fallida",
  "path": "/api/v1/estudiantes",
  "fields": {
    "nombre": "El nombre es obligatorio"
  }
}
```

No encontrado (404):
```json
{
  "timestamp": "2025-10-21T15:00:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Estudiante no encontrado",
  "path": "/api/v1/estudiantes/99"
}
```

Conflicto (409):
```json
{
  "timestamp": "2025-10-21T15:00:00Z",
  "status": 409,
  "error": "Conflict",
  "message": "El nombre ya existe",
  "path": "/api/v1/estudiantes"
}
```

---

## 10) Pruebas rápidas con MockMvc (opcional)

```java
// src/test/java/com/riwi/academico/infrastructure/adapters/in/rest/EstudianteControllerValidationTest.java
package com.riwi.academico.infrastructure.adapters.in.rest;

import com.riwi.academico.application.usecase.EstudianteUseCase;
import com.riwi.academico.domain.Estudiante;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(EstudianteController.class)
class EstudianteControllerValidationTest {

  @Autowired private MockMvc mvc;
  @MockBean private EstudianteUseCase useCase;

  @Test
  void debeRetornar400SiNombreInvalido() throws Exception {
    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"nombre\":\"\"}"))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.error").value("Bad Request"))
      .andExpect(jsonPath("$.fields.nombre").exists());
  }

  @Test
  void debeRetornar409SiDuplicado() throws Exception {
    Mockito.when(useCase.crear("Ana")).thenThrow(new IllegalArgumentException("El nombre ya existe"));

    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"nombre\":\"Ana\"}"))
      .andExpect(status().isConflict())
      .andExpect(jsonPath("$.message").value("El nombre ya existe"));
  }
}
```

---

## 11) Checklist del día

- `@Valid` en cuerpos y `@Validated` en parámetros.  
- `@RestControllerAdvice` con respuestas homogéneas.  
- Mensajes de validación claros (posible i18n).  
- Pruebas de error básicas con MockMvc.

---

## 12) Resultado esperado

- Contrato HTTP robusto: validaciones automáticas y errores consistentes.  
- Controladores simples, sin `try/catch` repetitivos.  
- Base sólida para documentar con OpenAPI/Swagger (Día 3) y para integrar seguridad/persistencia más adelante.