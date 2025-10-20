
# Día 1 –  Creación de Endpoints REST con Spring MVC y DTOs  

En esta sesión aprenderás a exponer servicios REST profesionales usando **Spring MVC**, aplicando principios de **arquitectura limpia**, separación de capas y validaciones automáticas.  
Se usará el dominio académico como ejemplo (Estudiantes y Cursos).

---

## 1) Objetivos del día

- Comprender cómo funciona **Spring MVC** dentro del ecosistema Spring Boot.  
- Crear controladores REST que comuniquen adaptadores de entrada con el dominio.  
- Implementar **DTOs (Data Transfer Objects)** para desacoplar el modelo del dominio.  
- Aplicar **validaciones automáticas** con `@Valid` y `jakarta.validation`.  
- Producir y consumir JSON correctamente.  
- Mantener la independencia del dominio siguiendo el patrón **hexagonal**.

---

## 2) Spring MVC: concepto base

**Spring MVC (Model-View-Controller)** permite estructurar las aplicaciones web y REST bajo el patrón controlador.  
En un contexto de servicios, se usa principalmente para crear **controladores REST**, sin vistas ni HTML.

Flujo simplificado:

```
Cliente HTTP → Controlador (Controller) → Caso de uso (Service) → Puerto (Repositorio) → Adaptador (Base de datos)
```

**El controlador es solo un mediador** entre la capa externa (peticiones) y el dominio.  

---

## 3) Dependencias necesarias (`pom.xml`)

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
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

---

## 4) Estructura recomendada del proyecto

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/
 │   ├─ ports/
 │   └─ service/
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/   # Controladores REST (Spring MVC)
 │   │   └─ out/  # Adaptadores de salida (JPA)
 │   ├─ jpa/
 │   └─ config/
 ├─ application/
 │   └─ AcademicoApplication.java
 └─ dto/
     ├─ EstudianteRequest.java
     └─ EstudianteResponse.java
```

---

## 5) DTOs: Data Transfer Objects

Los **DTOs** permiten separar la representación de los datos (entrada/salida HTTP) de las entidades del dominio.  
Esto evita exponer directamente las clases internas del modelo.

### Ejemplo de DTO de entrada
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

### Ejemplo de DTO de salida
```java
// dto/EstudianteResponse.java
package com.riwi.academico.dto;

public class EstudianteResponse {
    private Long id;
    private String nombre;

    public EstudianteResponse(Long id, String nombre) {
        this.id = id;
        this.nombre = nombre;
    }

    public Long getId() { return id; }
    public String getNombre() { return nombre; }
}
```

---

## 6) Controlador REST (adaptador de entrada)

```java
// infrastructure/adapters/in/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.service.RegistrarEstudianteService;
import com.riwi.academico.dto.EstudianteRequest;
import com.riwi.academico.dto.EstudianteResponse;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {

    private final RegistrarEstudianteService service;

    public EstudianteController(RegistrarEstudianteService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest request) {
        Estudiante e = service.ejecutar(request.getNombre());
        return ResponseEntity.ok(new EstudianteResponse(e.getId(), e.getNombre()));
    }

    @GetMapping
    public ResponseEntity<List<EstudianteResponse>> listar() {
        List<EstudianteResponse> lista = service.listar()
                .stream()
                .map(e -> new EstudianteResponse(e.getId(), e.getNombre()))
                .collect(Collectors.toList());
        return ResponseEntity.ok(lista);
    }
}
```

**Puntos importantes:**
- `@RestController` combina `@Controller` y `@ResponseBody`.  
- `@RequestBody` convierte JSON a objeto Java.  
- `@Valid` activa las validaciones del DTO.  
- `ResponseEntity` permite controlar códigos y cuerpo de respuesta.  

---

## 7) Mapeo entre capas

En una arquitectura limpia:
- El **controlador** no accede a las entidades JPA.  
- La **capa de dominio** no depende de DTOs ni anotaciones Spring.  
- Los **mappers** transforman DTO ↔ modelo de dominio.  

```java
public class EstudianteMapper {
    public static EstudianteResponse toResponse(Estudiante e) {
        return new EstudianteResponse(e.getId(), e.getNombre());
    }
}
```

---

## 8) Manejo de errores y validaciones

Para manejar errores de validación de forma centralizada:

```java
// infrastructure/config/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.config;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errores = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> errores.put(error.getField(), error.getDefaultMessage()));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errores);
    }
}
```

---

## 9) Prueba del controlador con MockMvc

```java
@WebMvcTest(EstudianteController.class)
class EstudianteControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private RegistrarEstudianteService service;

    @Test
    void debeCrearEstudianteConExito() throws Exception {
        Mockito.when(service.ejecutar("Ana")).thenReturn(new Estudiante("Ana"));

        mvc.perform(post("/api/estudiantes")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"nombre\": \"Ana\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.nombre").value("Ana"));
    }
}
```

---

## 10) Configuración en IntelliJ IDEA

1. Crear el proyecto con **Spring Initializr**.  
2. Dependencias: Spring Web, Validation, Lombok, Spring Boot Test, H2.  
3. Habilitar `Annotation Processing` desde Compiler Settings.  
4. Ejecutar controladores con `Ctrl+Shift+F10` (Windows/Linux) o `⌘⇧R` (Mac).  
5. Ver endpoints disponibles: `http://localhost:8080/api/estudiantes`.  

---

## 11) Buenas prácticas

| Práctica | Beneficio |
|-----------|------------|
| Usar DTOs para entrada y salida | Evita acoplar el dominio al API |
| Validar datos con `@Valid` | Aumenta la seguridad del sistema |
| Centralizar excepciones | Código más limpio y mantenible |
| Evitar lógica en el controlador | Mantiene responsabilidad única |
| Responder con códigos HTTP adecuados | Mejora la comunicación API/cliente |

---

## 12) Resultado esperado

- API REST funcional con endpoints `GET` y `POST`.  
- Validaciones automáticas y manejo centralizado de errores.  
- Controladores desacoplados del dominio.  
- Código preparado para ampliar con autenticación y seguridad.  