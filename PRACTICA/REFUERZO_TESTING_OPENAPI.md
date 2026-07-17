# Refuerzo — Testing y documentación OpenAPI

> **Estado curricular:** refuerzo de testing/OpenAPI para Semanas 2 y 4. Las versiones se gestionan con BOM.

En esta sesión aprenderás a **probar tu API REST** (unitarias de casos de uso y web-slice del controlador) con **JUnit 5 y Mockito** y a **documentarla automáticamente** con **Swagger (springdoc-openapi)**, manteniendo **arquitectura hexagonal**: dominio y casos de uso libres de framework; documentación y controladores en **adapters**.  
Este refuerzo consolida Semanas 2 y 4 con código probado y contrato documentado. API Gateway se incorpora más adelante; aquí se prueba el servicio localmente.

---

## 1) Objetivos del día

- Implementar **pruebas unitarias** con **JUnit 5** y **Mockito** sobre **casos de uso** (application).  
- Crear **pruebas web-slice** con **MockMvc** sobre el **adaptador REST**.  
- Agregar documentación automática con **OpenAPI / Swagger UI (springdoc)** en el **adaptador REST**.  
- Dejar dependencias y estructura **reutilizables** para todos los microservicios educativos.  
- Entender cómo testing y documentación se integran en un ciclo de desarrollo profesional.

---

## 2) Dependencias necesarias (`pom.xml`)

```xml
<dependencies>
  <!-- Web + Validation (solo para el adapter REST) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- Testing -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- Documentación OpenAPI -->
  <dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.3</version>
  </dependency>
</dependencies>
```

*(Opcional para cobertura: agrega el plugin JaCoCo como en Semana 4 Día 3.)*

---

## 3) Estructura de proyecto (hexagonal)

```
com.riwi.academico
 ├─ domain/                         # Entidades y reglas puras
 ├─ application/
 │   ├─ ports/                      # Puertos (repos, cache, etc.)
 │   └─ usecase/                    # Casos de uso (servicios de aplicación)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/EstudianteController.java   # REST (springdoc anota aquí)
 │   │   └─ out/persistence/...                 # Repositorios (JPA, in-memory, etc.)
 │   └─ config/                                 # OpenAPI config, handlers
 ├─ dto/
 └─ src/test/java/com/riwi/academico
     ├─ application/usecase/                    # Tests unitarios
     └─ infrastructure/adapters/in/rest/        # Tests MockMvc
```

---

## 4) Caso de uso y puerto (application)

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
    if (repo.existsByNombre(nombre)) throw new IllegalArgumentException("estudiante duplicado");
    var e = new Estudiante(repo.nextId(), nombre);
    return repo.save(e);
  }

  public Estudiante obtener(Long id) {
    return repo.findById(id).orElseThrow(() -> new NoSuchElementException("no existe estudiante"));
  }

  public List<Estudiante> listar() { return repo.findAll(); }
}
```

---

## 5) Dominio y DTOs

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
// dto/EstudianteRequest.java
package com.riwi.academico.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class EstudianteRequest {
  @NotBlank @Size(min=3, max=50) private String nombre;
  public String getNombre(){ return nombre; }
  public void setNombre(String nombre){ this.nombre = nombre; }
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

---

## 6) Adaptador de entrada (REST) + OpenAPI

```java
// infrastructure/adapters/in/rest/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in.rest;

import com.riwi.academico.application.usecase.EstudianteUseCase;
import com.riwi.academico.dto.EstudianteRequest;
import com.riwi.academico.dto.EstudianteResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/estudiantes")
@Tag(name = "Estudiantes", description = "Operaciones CRUD educativas")
public class EstudianteController {

  private final EstudianteUseCase useCase;
  public EstudianteController(EstudianteUseCase useCase){ this.useCase = useCase; }

  @Operation(summary = "Crear estudiante", description = "Crea un nuevo estudiante con validación de nombre único.")
  @ApiResponse(responseCode = "201", description = "Estudiante creado con éxito")
  @ApiResponse(responseCode = "400", description = "Validación fallida o duplicado")
  @PostMapping
  public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest req){
    var e = useCase.crear(req.getNombre());
    return ResponseEntity.status(HttpStatus.CREATED).body(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @Operation(summary = "Obtener estudiante por id")
  @ApiResponse(responseCode = "200", description = "OK")
  @ApiResponse(responseCode = "404", description = "No encontrado")
  @GetMapping("/{id}")
  public ResponseEntity<EstudianteResponse> obtener(@PathVariable Long id){
    var e = useCase.obtener(id);
    return ResponseEntity.ok(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @Operation(summary = "Listar estudiantes")
  @GetMapping
  public ResponseEntity<List<EstudianteResponse>> listar(){
    var res = useCase.listar().stream().map(e -> new EstudianteResponse(e.getId(), e.getNombre())).toList();
    return ResponseEntity.ok(res);
  }
}
```

**Configuración OpenAPI global:**

```java
// infrastructure/config/OpenApiConfig.java
package com.riwi.academico.infrastructure.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {
  @Bean
  public OpenAPI apiInfo(){
    return new OpenAPI()
      .info(new Info().title("API Académico RIWI").version("1.0.0")
      .description("Documentación de endpoints educativos REST para gestión de estudiantes."));
  }
}
```

**Rutas por defecto:**  
- Swagger UI: `http://localhost:8080/swagger-ui.html`  
- JSON OpenAPI: `http://localhost:8080/v3/api-docs`

---

## 7) Manejo global de errores (adapter)

```java
// infrastructure/config/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.config;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.NoSuchElementException;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<Map<String,Object>> badRequest(MethodArgumentNotValidException ex, HttpServletRequest req){
    var fields = new LinkedHashMap<String,String>();
    ex.getBindingResult().getFieldErrors().forEach(f -> fields.put(f.getField(), f.getDefaultMessage()));
    var body = new LinkedHashMap<String,Object>();
    body.put("timestamp", Instant.now().toString());
    body.put("status", 400); body.put("error","Bad Request");
    body.put("message", "validación fallida");
    body.put("path", req.getRequestURI());
    body.put("fields", fields);
    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<Map<String,Object>> conflict(IllegalArgumentException ex, HttpServletRequest req){
    var body = Map.of("timestamp", Instant.now().toString(), "status", 409, "error","Conflict",
                      "message", ex.getMessage(), "path", req.getRequestURI());
    return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
  }

  @ExceptionHandler(NoSuchElementException.class)
  public ResponseEntity<Map<String,Object>> notFound(NoSuchElementException ex, HttpServletRequest req){
    var body = Map.of("timestamp", Instant.now().toString(), "status", 404, "error","Not Found",
                      "message", ex.getMessage(), "path", req.getRequestURI());
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
  }
}
```

---

## 8) Pruebas **unitarias** (caso de uso) con JUnit 5 + Mockito

```java
// src/test/java/com/riwi/academico/application/usecase/EstudianteUseCaseTest.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.application.ports.EstudianteRepositoryPort;
import com.riwi.academico.domain.Estudiante;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;

import java.util.NoSuchElementException;
import java.util.Optional;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class EstudianteUseCaseTest {

  @Mock private EstudianteRepositoryPort repo;
  private EstudianteUseCase useCase;

  @BeforeEach void init(){
    MockitoAnnotations.openMocks(this);
    useCase = new EstudianteUseCase(repo);
  }

  @Test
  void crear_ok(){
    when(repo.existsByNombre("Ana")).thenReturn(false);
    when(repo.nextId()).thenReturn(1L);
    when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));

    var e = useCase.crear("Ana");
    assertEquals(1L, e.getId());
    assertEquals("Ana", e.getNombre());
    verify(repo).save(any());
  }

  @Test
  void crear_duplicado_lanza_excepcion(){
    when(repo.existsByNombre("Ana")).thenReturn(true);
    assertThrows(IllegalArgumentException.class, () -> useCase.crear("Ana"));
  }

  @Test
  void obtener_ok(){
    when(repo.findById(1L)).thenReturn(Optional.of(new Estudiante(1L, "Ana")));
    var e = useCase.obtener(1L);
    assertEquals("Ana", e.getNombre());
  }

  @Test
  void obtener_no_existe(){
    when(repo.findById(99L)).thenReturn(Optional.empty());
    assertThrows(NoSuchElementException.class, () -> useCase.obtener(99L));
  }

  @Test
  void listar_ok(){
    when(repo.findAll()).thenReturn(List.of(new Estudiante(1L,"A"), new Estudiante(2L,"B")));
    var lista = useCase.listar();
    assertEquals(2, lista.size());
  }
}
```

---

## 9) Pruebas **web-slice** (MockMvc) del adaptador REST

```java
// src/test/java/com/riwi/academico/infrastructure/adapters/in/rest/EstudianteControllerTest.java
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

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(EstudianteController.class)
class EstudianteControllerTest {

  @Autowired private MockMvc mvc;
  @MockBean private EstudianteUseCase useCase;

  @Test
  void crear_201() throws Exception {
    Mockito.when(useCase.crear("Ana")).thenReturn(new Estudiante(1L, "Ana"));

    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"nombre":"Ana"}"))
      .andExpect(status().isCreated())
      .andExpect(jsonPath("$.id").value(1))
      .andExpect(jsonPath("$.nombre").value("Ana"));
  }

  @Test
  void crear_400_validacion() throws Exception {
    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"nombre":""}"))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.error").value("Bad Request"));
  }
}
```

---

## 10) Buenas prácticas de documentación (OpenAPI)

| Recomendación | Beneficio |
|----------------|-----------|
| Anotar métodos con `@Operation` y `@ApiResponse` | Contrato explícito y navegable |
| Versionar rutas (`/api/v1/...`) | Evolución segura del API |
| Mantener `OpenApiConfig` en **infraestructura** | Dominio y casos de uso limpios |
| Exportar `/v3/api-docs.yaml` | Útil para Postman/Frontends |
| Reutilizar esquemas DTO | Consistencia entre servicios |

---

## 11) Ejecución y validación final

1. `mvn clean test` → deben pasar unitarias y web-slice.  
2. Levanta el servicio: `mvn spring-boot:run`.  
3. Visita **Swagger UI**: `http://localhost:8080/swagger-ui.html`.  
4. Prueba los endpoints y revisa errores del `GlobalExceptionHandler`.  
5. (Posterior) Exponlo por API Gateway y agrega pruebas de contrato como en Semana 14.

---

## 12) Resultado esperado

- Servicio probado con **JUnit 5 + Mockito** (caso de uso) y **MockMvc** (REST).  
- **Swagger** operativo con contrato claro.  
- Proyecto listo para integrar persistencia, seguridad y despliegues educativos.
