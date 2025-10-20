
# Semana 4 – Día 2  
# Validaciones Avanzadas y Manejo Global de Excepciones en Spring Boot

En esta sesión aprenderás a construir un **sistema robusto de validaciones y manejo de errores** en APIs REST usando **Spring Boot**, bajo principios de **arquitectura limpia**.  
El objetivo es que tus controladores sean limpios, sin lógica de validación o control de errores repetida.

---

## 1) Objetivos del día

- Implementar validaciones personalizadas con `jakarta.validation`.  
- Centralizar el manejo de errores mediante `@ControllerAdvice`.  
- Crear una estructura uniforme de respuesta de error.  
- Diferenciar entre errores de dominio, infraestructura y validación.  
- Comprender la importancia del desacoplamiento entre dominio y framework.

---

## 2) Por qué manejar errores de forma uniforme

En proyectos empresariales es vital que todas las respuestas de error tengan un formato predecible y homogéneo, por ejemplo:

```json
{
  "timestamp": "2025-10-20T14:30:00",
  "status": 400,
  "error": "Bad Request",
  "message": "El campo 'nombre' es obligatorio",
  "path": "/api/estudiantes"
}
```

Esto facilita:
- Integración con frontends y sistemas externos.  
- Depuración centralizada.  
- Mantenibilidad y consistencia.  

---

## 3) Estructura de proyecto

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/
 │   ├─ ports/
 │   └─ service/
 ├─ infrastructure/
 │   ├─ adapters/in/        # Controladores REST
 │   ├─ adapters/out/       # Persistencia
 │   └─ exception/          # Excepciones y manejador global
 ├─ dto/
 ├─ application/
 └─ AcademicoApplication.java
```

---

## 4) Validaciones con `jakarta.validation`

Spring Boot integra automáticamente el **Bean Validation API**.  
Solo necesitas anotar tus DTOs con restricciones y usar `@Valid` en los controladores.

### Ejemplo de DTO con validaciones
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

### Controlador con validación automática
```java
@PostMapping
public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest request) {
    Estudiante e = service.ejecutar(request.getNombre());
    return ResponseEntity.ok(new EstudianteResponse(e.getId(), e.getNombre()));
}
```

Si la validación falla, Spring lanza una excepción `MethodArgumentNotValidException`.

---

## 5) Manejador global de errores (`@ControllerAdvice`)

El manejador global evita escribir `try-catch` en cada controlador.

```java
// infrastructure/exception/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.context.request.WebRequest;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidationException(MethodArgumentNotValidException ex, WebRequest request) {
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.BAD_REQUEST.value());
        body.put("error", "Bad Request");

        Map<String, String> errores = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err -> errores.put(err.getField(), err.getDefaultMessage()));
        body.put("errors", errores);

        body.put("path", request.getDescription(false).replace("uri=", ""));
        return new ResponseEntity<>(body, HttpStatus.BAD_REQUEST);
    }
}
```

---

## 6) Excepciones personalizadas del dominio

En una arquitectura limpia, las excepciones se agrupan por **nivel** o **responsabilidad**:

| Tipo | Ejemplo | Nivel |
|-------|----------|--------|
| `NegocioException` | Error de lógica de negocio (dominio) | Dominio |
| `RecursoNoEncontradoException` | Entidad no existente | Infraestructura |
| `ValidacionException` | Datos inválidos | Capa de entrada |

### Ejemplo
```java
// domain/exception/NegocioException.java
package com.riwi.academico.domain.exception;

public class NegocioException extends RuntimeException {
    public NegocioException(String message) {
        super(message);
    }
}
```

```java
// infrastructure/exception/GlobalExceptionHandler.java
@ExceptionHandler(NegocioException.class)
public ResponseEntity<Map<String, Object>> handleBusinessException(NegocioException ex, WebRequest request) {
    Map<String, Object> body = new HashMap<>();
    body.put("timestamp", LocalDateTime.now());
    body.put("status", HttpStatus.UNPROCESSABLE_ENTITY.value());
    body.put("error", "Unprocessable Entity");
    body.put("message", ex.getMessage());
    body.put("path", request.getDescription(false).replace("uri=", ""));
    return new ResponseEntity<>(body, HttpStatus.UNPROCESSABLE_ENTITY);
}
```

---

## 7) Validaciones personalizadas (`@Constraint`)

A veces se necesitan reglas propias, como validar que un nombre **no contenga números**.

### 7.1 Crear la anotación
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

### 7.2 Crear el validador
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

### 7.3 Usarla en el DTO
```java
public class EstudianteRequest {
    @NotBlank(message = "El nombre no puede estar vacío")
    @NoNumeros
    private String nombre;
}
```

---

## 8) Testing del manejo de errores con MockMvc

```java
@WebMvcTest(EstudianteController.class)
class EstudianteControllerValidationTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private RegistrarEstudianteService service;

    @Test
    void debeRetornarErrorDeValidacion() throws Exception {
        mvc.perform(post("/api/estudiantes")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"nombre\": \"12\"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors.nombre").value("El texto no debe contener números"));
    }
}
```

---

## 9) Buenas prácticas

| Práctica | Beneficio |
|-----------|------------|
| Usar DTOs validados | Evita lógica de validación en el dominio |
| Centralizar errores en `@ControllerAdvice` | Código limpio y reutilizable |
| Crear excepciones por capa | Facilita la depuración |
| No exponer mensajes internos | Aumenta la seguridad del API |
| Mantener formato de respuesta consistente | Mejora la experiencia de consumo |

---

## 10) Configuración en IntelliJ IDEA

1. **Dependencias:** `spring-boot-starter-validation`, `spring-boot-starter-web`, `spring-boot-starter-test`.  
2. **Activar procesamiento de anotaciones:**  
   `Settings → Build → Compiler → Annotation Processors → Enable`.  
3. **Ejecutar pruebas:**  
   `Ctrl+Shift+F10` (Windows/Linux) o `⌘⇧R` (Mac).  
4. **Probar con Postman o curl:**  
   ```bash
   curl -X POST http://localhost:8080/api/estudiantes -H "Content-Type: application/json" -d '{"nombre": ""}'
   ```

---

## 11) Resultado esperado

- Validaciones automáticas en todos los endpoints.  
- Manejador global centralizado y extensible.  
- Respuestas uniformes de error con timestamp y ruta.  
- Validaciones personalizadas funcionales.  
- Pruebas unitarias verificando el flujo de errores.  