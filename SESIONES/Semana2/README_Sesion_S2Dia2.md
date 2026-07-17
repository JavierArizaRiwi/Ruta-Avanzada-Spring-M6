# Día 2 — APIs REST profesionales con Spring MVC y DTOs

> **Estado curricular:** núcleo REST de Semana 2. La arquitectura hexagonal se formaliza en Semana 4.

En esta sesión aprenderás a exponer servicios REST profesionales usando **Spring MVC**, aplicando principios de **arquitectura limpia (hexagonal)**, separación de capas y validaciones automáticas.
Usaremos el dominio académico como ejemplo (Estudiantes y Cursos).

---

## 1) Objetivos del día

- Comprender el rol de **Spring MVC** dentro del ecosistema Spring Boot.
- Crear controladores REST que comuniquen adaptadores de entrada con el dominio.
- Implementar **DTOs (Data Transfer Objects)** para desacoplar el modelo del dominio.
- Aplicar **validaciones automáticas** con `@Valid` y `jakarta.validation`.
- Producir y consumir JSON correctamente.
- Mantener la independencia del dominio siguiendo el patrón **hexagonal**.

---

## 2) Spring MVC: concepto base

**Spring MVC (Model-View-Controller)** estructura la aplicación web en capas separadas.
En aplicaciones REST, se usa para construir **controladores REST** que procesan peticiones HTTP y delegan la lógica al **dominio**.

Flujo típico:

```
Cliente HTTP → Controlador (adaptador de entrada)
            → Caso de uso (servicio de dominio)
            → Puerto (interfaz)
            → Adaptador de salida (JPA, persistencia, API externa)
```

El **controlador** solo traduce HTTP ↔ dominio; nunca contiene reglas de negocio.

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

## 4) Estructura del proyecto (hexagonal)

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/
 │   ├─ ports/
 │   └─ service/
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/         # Controladores REST (entrada)
 │   │   └─ out/jpa/         # Persistencia (salida)
 │   ├─ config/              # Configuraciones globales
 │   └─ exception/           # Manejo de errores global
 ├─ dto/                     # DTOs request/response
 └─ application/
     └─ AcademicoApplication.java
```

---

## 5) DTOs: Data Transfer Objects

Los **DTOs** desacoplan la representación de datos del dominio y la API.

### 5.1 DTO de entrada
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

### 5.2 DTO de salida
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
// infrastructure/adapters/in/rest/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in.rest;

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
@RequestMapping("/api/v1/estudiantes")
public class EstudianteController {

  private final RegistrarEstudianteService service;

  public EstudianteController(RegistrarEstudianteService service) {
    this.service = service;
  }

  @PostMapping
  public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest request) {
    Estudiante e = service.ejecutar(request.getNombre());
    return ResponseEntity.status(201).body(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @GetMapping
  public ResponseEntity<List<EstudianteResponse>> listar() {
    List<EstudianteResponse> lista = service.listar().stream()
      .map(e -> new EstudianteResponse(e.getId(), e.getNombre()))
      .collect(Collectors.toList());
    return ResponseEntity.ok(lista);
  }
}
```

---

## 7) Puerto y servicio del dominio

```java
// domain/ports/EstudianteRepositoryPort.java
package com.riwi.academico.domain.ports;

import com.riwi.academico.domain.model.Estudiante;
import java.util.List;

public interface EstudianteRepositoryPort {
  Estudiante guardar(Estudiante e);
  List<Estudiante> listar();
}
```

```java
// domain/service/RegistrarEstudianteService.java
package com.riwi.academico.domain.service;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.ports.EstudianteRepositoryPort;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class RegistrarEstudianteService {
  private final EstudianteRepositoryPort repo;

  public RegistrarEstudianteService(EstudianteRepositoryPort repo) {
    this.repo = repo;
  }

  public Estudiante ejecutar(String nombre) {
    return repo.guardar(new Estudiante(null, nombre));
  }

  public List<Estudiante> listar() {
    return repo.listar();
  }
}
```

---

## 8) Manejo global de errores

```java
// infrastructure/exception/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.exception;

import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import java.time.Instant;
import java.util.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
    var fields = new LinkedHashMap<String, String>();
    ex.getBindingResult().getFieldErrors().forEach(f -> fields.put(f.getField(), f.getDefaultMessage()));

    var body = new LinkedHashMap<String, Object>();
    body.put("timestamp", Instant.now().toString());
    body.put("status", 400);
    body.put("error", "Bad Request");
    body.put("fields", fields);

    return ResponseEntity.badRequest().body(body);
  }
}
```

---

## 9) Prueba del controlador con MockMvc

```java
@WebMvcTest(EstudianteController.class)
class EstudianteControllerTest {

  @Autowired private MockMvc mvc;
  @MockitoBean private RegistrarEstudianteService service;

  @Test
  void debeCrearEstudianteConExito() throws Exception {
    Mockito.when(service.ejecutar("Ana")).thenReturn(new Estudiante(1L, "Ana"));

    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"nombre\":\"Ana\"}"))
      .andExpect(status().isCreated())
      .andExpect(jsonPath("$.nombre").value("Ana"));
  }
}
```

---

## 10) Buenas prácticas

| Práctica | Beneficio |
|-----------|-----------|
| DTOs de entrada/salida | Evita acoplar el dominio al API |
| `@Valid` + Bean Validation | Datos confiables y consistentes |
| Controladores simples | Facilita pruebas y mantenimiento |
| Respuestas con `ResponseEntity` | Códigos HTTP adecuados |
| Arquitectura hexagonal | Independencia del framework |

---

## 11) Resultado esperado

- API funcional con endpoints `POST` y `GET`.
- Validaciones automáticas activas.
- Controladores limpios y desacoplados del dominio.
- Base para integrar seguridad y gateway más adelante.
