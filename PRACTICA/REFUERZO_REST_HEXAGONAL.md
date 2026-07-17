# Refuerzo — REST con arquitectura hexagonal

> **Estado curricular:** laboratorio de refactor de Semana 5, no una segunda introducción a REST.

En esta sesión aprenderás a exponer servicios REST profesionales usando **Spring MVC**, aplicando principios de **arquitectura hexagonal**, separación de capas y **validaciones automáticas**.  
Dominio educativo: **Estudiantes**. API Gateway se aborda en Semana 14; aquí trabajamos el servicio local y preparamos una estructura que luego conectaremos a JPA, caché y mensajería.

---

## 1) Objetivos del día

- Comprender el rol de **Spring MVC** como **adaptador de entrada** en una arquitectura hexagonal.  
- Crear **controladores REST** desacoplados del dominio mediante **DTOs**.  
- Implementar **validaciones automáticas** con `@Valid` y `jakarta.validation`.  
- Producir y consumir **JSON** correctamente, devolviendo **códigos HTTP** apropiados.  
- Mantener la **independencia del dominio**; evitar filtrar entidades JPA o detalles de infraestructura hacia la capa web.

---

## 2) Flujo y responsabilidades (hexagonal)

```
Cliente HTTP → Adapter IN (Controller REST)
            → Application (Use Case)        ←→  Application Ports (interfaces)
            → Adapter OUT (Persistence, etc.)  [hoy: InMemory; luego: JPA/H2]
            → Domain (Entidades + reglas)
```

**Reglas clave**:
- El **controller** solo orquesta: recibe DTOs, invoca **use cases**, mapea respuestas.  
- El **use case** contiene la lógica de negocio y depende de **puertos** (interfaces).  
- Los **adapters** implementan puertos (p. ej., repositorio en memoria o JPA).  
- El **dominio** no depende de Spring ni de librerías de infraestructura.

---

## 3) Dependencias mínimas (`pom.xml`)

```xml
<dependencies>
  <!-- Web (adapter IN) + Validación -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- (Opcional para el Día 4: JPA/H2) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>

  <!-- Tests (se usarán en Día 3) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## 4) Estructura sugerida del proyecto (impecable)

```
com.riwi.academico
 ├─ domain/                                   # Entidades y reglas puras (sin Spring)
 ├─ application/
 │   ├─ ports/                                # Puertos (interfaces) p.ej. repositorio
 │   └─ usecase/                              # Casos de uso (servicios de aplicación)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/EstudianteController.java # Adapter de entrada (Spring MVC)
 │   │   └─ out/persistence/                  # Adapter de salida (InMemory/JPA)
 │   └─ config/                               # Beans y manejo global de errores
 ├─ dto/                                      # DTOs request/response
 └─ AcademicoApplication.java
```

> Más adelante podrás reemplazar `out/persistence/InMemory...` por `JPA` **sin tocar** controller ni use case.

---

## 5) DTOs: contrato de entrada/salida

Los **DTOs** evitan exponer clases internas del dominio y facilitan **validaciones** y **evolución** del API.

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

```java
// dto/EstudianteResponse.java
package com.riwi.academico.dto;

public class EstudianteResponse {
  private Long id;
  private String nombre;

  public EstudianteResponse(Long id, String nombre) {
    this.id = id; this.nombre = nombre;
  }
  public Long getId() { return id; }
  public String getNombre() { return nombre; }
}
```

---

## 6) Dominio + Puertos + Caso de uso (application)

```java
// domain/Estudiante.java
package com.riwi.academico.domain;

public class Estudiante {
  private Long id;
  private String nombre;
  public Estudiante(Long id, String nombre){ this.id=id; this.nombre=nombre; }
  public Long getId(){ return id; }
  public String getNombre(){ return nombre; }
}
```

```java
// application/ports/EstudianteRepositoryPort.java
package com.riwi.academico.application.ports;

import com.riwi.academico.domain.Estudiante;
import java.util.List;
import java.util.Optional;

public interface EstudianteRepositoryPort {
  Long nextId();
  boolean existsByNombre(String nombre);
  Estudiante save(Estudiante e);
  Optional<Estudiante> findById(Long id);
  List<Estudiante> findAll();
}
```

```java
// application/usecase/RegistrarEstudianteUseCase.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.application.ports.EstudianteRepositoryPort;
import com.riwi.academico.domain.Estudiante;

import java.util.List;
import java.util.NoSuchElementException;

public class RegistrarEstudianteUseCase {

  private final EstudianteRepositoryPort repo;
  public RegistrarEstudianteUseCase(EstudianteRepositoryPort repo){ this.repo = repo; }

  public Estudiante crear(String nombre){
    if (repo.existsByNombre(nombre)) throw new IllegalArgumentException("El nombre ya existe");
    var e = new Estudiante(repo.nextId(), nombre);
    return repo.save(e);
  }

  public Estudiante obtener(Long id){
    return repo.findById(id).orElseThrow(() -> new NoSuchElementException("Estudiante no encontrado"));
  }

  public List<Estudiante> listar(){ return repo.findAll(); }
}
```

> Observa que **no hay anotaciones de Spring** en el **use case** ni en el **dominio**.

---

## 7) Adapter OUT (persistencia en memoria, educativo)

```java
// infrastructure/adapters/out/persistence/InMemoryEstudianteRepository.java
package com.riwi.academico.infrastructure.adapters.out.persistence;

import com.riwi.academico.application.ports.EstudianteRepositoryPort;
import com.riwi.academico.domain.Estudiante;
import org.springframework.stereotype.Repository;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class InMemoryEstudianteRepository implements EstudianteRepositoryPort {

  private final Map<Long, Estudiante> data = new ConcurrentHashMap<>();
  private final AtomicLong seq = new AtomicLong(0);

  @Override public Long nextId(){ return seq.incrementAndGet(); }

  @Override public boolean existsByNombre(String nombre){
    return data.values().stream().anyMatch(e -> e.getNombre().equalsIgnoreCase(nombre));
  }

  @Override public Estudiante save(Estudiante e){
    data.put(e.getId(), e);
    return e;
  }

  @Override public Optional<Estudiante> findById(Long id){ return Optional.ofNullable(data.get(id)); }

  @Override public List<Estudiante> findAll(){ return new ArrayList<>(data.values()); }
}
```

---

## 8) Wiring (configurar el use case como bean)

```java
// infrastructure/config/AppConfig.java
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.application.ports.EstudianteRepositoryPort;
import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

  @Bean
  public RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo){
    return new RegistrarEstudianteUseCase(repo);
  }
}
```

---

## 9) Adapter IN (Controller REST)

Orquesta HTTP ↔ Use Case ↔ DTOs y controla códigos de respuesta.

```java
// infrastructure/adapters/in/rest/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in.rest;

import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.Estudiante;
import com.riwi.academico.dto.EstudianteRequest;
import com.riwi.academico.dto.EstudianteResponse;
import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/estudiantes")
public class EstudianteController {

  private final RegistrarEstudianteUseCase useCase;
  public EstudianteController(RegistrarEstudianteUseCase useCase){ this.useCase = useCase; }

  @PostMapping
  public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest req){
    Estudiante e = useCase.crear(req.getNombre());
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @GetMapping("/{id}")
  public ResponseEntity<EstudianteResponse> obtener(@PathVariable Long id){
    Estudiante e = useCase.obtener(id);
    return ResponseEntity.ok(new EstudianteResponse(e.getId(), e.getNombre()));
  }

  @GetMapping
  public ResponseEntity<List<EstudianteResponse>> listar(){
    var res = useCase.listar().stream()
            .map(e -> new EstudianteResponse(e.getId(), e.getNombre()))
            .toList();
    return ResponseEntity.ok(res);
  }
}
```

---

## 10) Manejo global de errores (recomendado)

Centraliza respuestas de error coherentes (validación, negocio y no encontrado).

```java
// infrastructure/config/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.config;

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
    body.put("message", "Validación fallida");
    body.put("fields", fields);

    return ResponseEntity.badRequest().body(body);
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ResponseEntity<Map<String, Object>> handleConflict(IllegalArgumentException ex) {
    var body = Map.of(
      "timestamp", Instant.now().toString(),
      "status", 409, "error", "Conflict", "message", ex.getMessage()
    );
    return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
  }

  @ExceptionHandler(NoSuchElementException.class)
  public ResponseEntity<Map<String, Object>> handleNotFound(NoSuchElementException ex) {
    var body = Map.of(
      "timestamp", Instant.now().toString(),
      "status", 404, "error", "Not Found", "message", ex.getMessage()
    );
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
  }
}
```

---

## 11) Pruebas manuales rápidas

**Crear estudiante**
```
POST http://localhost:8080/api/v1/estudiantes
Content-Type: application/json

{
  "nombre": "Ana"
}
```
Respuestas esperadas:
- `201 Created` → `{ "id":1, "nombre":"Ana" }`
- `400 Bad Request` si `nombre` es vacío o < 3 caracteres
- `409 Conflict` si el nombre ya existe

**Obtener por id**
```
GET http://localhost:8080/api/v1/estudiantes/1
```

**Listar**
```
GET http://localhost:8080/api/v1/estudiantes
```

---

## 12) Buenas prácticas

| Práctica | Beneficio |
|----------|-----------|
| DTOs en entrada/salida | Desacopla la API del modelo interno |
| `@Valid` + `jakarta.validation` | Validaciones coherentes y seguras |
| `@RestControllerAdvice` | Respuestas de error consistentes |
| Evitar lógica en controlador | Responsabilidad única y tests simples |
| Códigos HTTP correctos | Contrato claro con el cliente |
| Use cases sin Spring | Dominio portable y testeable |

---

## 13) Resultado esperado

- Endpoints `POST /api/v1/estudiantes`, `GET /api/v1/estudiantes/{id}`, `GET /api/v1/estudiantes`.  
- Validación automática y **manejo de errores** centralizado.  
- Controladores limpios (adapter IN) y repositorio en memoria (adapter OUT) **fácilmente reemplazable** por JPA/H2 (Día 4).  
- Base hexagonal lista para Swagger, testing y la profundización de API Gateway en Semana 14.
