# Día 1 — Monolito modular y Clean Architecture aplicada en Spring Boot

> **Estado curricular:** arquitectura de Semana 4, aplicada después de construir una API REST persistente. Los puertos se introducen solo en fronteras con valor.

> Meta del día: diseñar y probar un monolito modular con dominio independiente, wiring explícito y adaptadores intercambiables. El ejemplo H2 se conserva como comparación rápida; la persistencia obligatoria sigue siendo PostgreSQL/Flyway construida en Semana 3. API Gateway se estudia en Semana 14.

---

## 0) Idea clave (antes de código)

- **Regla de oro**: **las dependencias apuntan hacia el dominio**. El dominio **no** conoce a Spring, ni a JPA, ni a HTTP.
- **Puertos (Ports / SPI)**: interfaces **definidas por el dominio** que describen lo que “necesita” (p. ej. guardar/listar estudiantes).
- **Adaptadores (Adapters)**: implementaciones concretas de esos puertos (JPA, JDBC, Redis, MQ…).
- **Casos de uso (Application)**: orquestan reglas de negocio usando **puertos**.
- **Entrypoints**: REST/gRPC/CLI que traducen el mundo externo a **casos de uso**.

```
Entrypoint (REST) → Application (UseCase) → Domain (Port) ← Infrastructure (Adapter JPA/Memory/MQ)
```

---

## 1) Estructura propuesta del proyecto

```
com.riwi.academico
 ├─ domain/                              # Núcleo independiente
 │   ├─ model/                           # Entidades, Value Objects (sin anotaciones Spring/JPA)
 │   ├─ spi/                             # Ports (interfaces que el dominio necesita)
 │   └─ service/                         # Servicios de dominio puros (si aplica)
 ├─ application/                         # Casos de uso (orquestación, sin framework)
 │   ├─ usecase/                         # Registrar, Obtener, Listar...
 │   └─ mapper/                          # Mappers Domain ↔ DTO (si decides ponerlos aquí)
 ├─ infrastructure/                      # Adaptadores y configuración
 │   ├─ jpa/
 │   │   ├─ entity/                      # Entidades JPA (solo aquí usan anotaciones JPA)
 │   │   ├─ repository/                  # Spring Data Repositories
 │   │   └─ adapter/                     # Implementación del port usando JPA
 │   ├─ memory/                          # Adaptador en memoria (para pruebas/arranque rápido)
 │   ├─ web/                             # Config web (CORS, excepciones), si no lo pones en entrypoints
 │   └─ config/                          # @Configuration: beans y wiring de Ports→Adapters
 ├─ entrypoints/
 │   └─ rest/                            # Controladores REST (Spring MVC)
 ├─ dto/                                 # DTOs request/response (solo para la capa web)
 └─ AcademicoApplication.java
```

> *Puedes mover los mappers a `entrypoints` o `application` según prefieras mantener DTOs fuera del dominio.*

---

## 2) Dominio: entidades y puertos (cero Spring)

### 2.1 Entidad (modelo de dominio simple)
```java
// domain/model/Estudiante.java
package com.riwi.academico.domain.model;

import java.util.Objects;

public class Estudiante {
  private final String id;
  private final String nombre;

  public Estudiante(String id, String nombre) {
    if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
    if (nombre == null || nombre.isBlank()) throw new IllegalArgumentException("nombre requerido");
    this.id = id;
    this.nombre = nombre;
  }

  public String getId() { return id; }
  public String getNombre() { return nombre; }

  @Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Estudiante)) return false;
    Estudiante that = (Estudiante) o;
    return Objects.equals(id, that.id);
  }
  @Override public int hashCode() { return Objects.hash(id); }
}
```

> **Sin** `@Entity`, **sin** `@Getter` de Lombok (puedes usarlo fuera del dominio si quieres).

### 2.2 Puerto que el dominio necesita
```java
// domain/spi/EstudianteRepositoryPort.java
package com.riwi.academico.domain.spi;

import com.riwi.academico.domain.model.Estudiante;
import java.util.List;
import java.util.Optional;

public interface EstudianteRepositoryPort {
  Estudiante guardar(Estudiante e);
  Optional<Estudiante> buscarPorId(String id);
  boolean existePorNombre(String nombre);
  List<Estudiante> listar();
}
```

---

## 3) Casos de uso (Application): orquestación sobre puertos

```java
// application/usecase/RegistrarEstudianteUseCase.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;

public class RegistrarEstudianteUseCase {

  private final EstudianteRepositoryPort repo;

  public RegistrarEstudianteUseCase(EstudianteRepositoryPort repo) {
    this.repo = repo;
  }

  public Estudiante ejecutar(String id, String nombre) {
    if (repo.existePorNombre(nombre)) {
      throw new IllegalArgumentException("Nombre duplicado");
    }
    return repo.guardar(new Estudiante(id, nombre));
  }
}
```

```java
// application/usecase/ObtenerEstudianteUseCase.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;

public class ObtenerEstudianteUseCase {
  private final EstudianteRepositoryPort repo;
  public ObtenerEstudianteUseCase(EstudianteRepositoryPort repo){ this.repo = repo; }

  public Estudiante porId(String id){
    return repo.buscarPorId(id).orElseThrow(() -> new IllegalArgumentException("No existe estudiante"));
  }
}
```

> No hay anotaciones Spring aquí. Estos casos de uso son **POJOs** fáciles de testear con Mockito.

---

## 4) Adaptadores de infraestructura

### 4.1 Adaptador en memoria (simple y útil para talleres)
```java
// infrastructure/memory/adapter/EstudianteInMemoryAdapter.java
package com.riwi.academico.infrastructure.memory.adapter;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class EstudianteInMemoryAdapter implements EstudianteRepositoryPort {

  private final Map<String, Estudiante> store = new ConcurrentHashMap<>();

  @Override public Estudiante guardar(Estudiante e) { store.put(e.getId(), e); return e; }
  @Override public Optional<Estudiante> buscarPorId(String id) { return Optional.ofNullable(store.get(id)); }
  @Override public boolean existePorNombre(String nombre) {
    return store.values().stream().anyMatch(s -> s.getNombre().equalsIgnoreCase(nombre));
  }
  @Override public List<Estudiante> listar() { return new ArrayList<>(store.values()); }
}
```

> Perfecto para **desarrollo rápido** y **tests de integración** sin BD.

### 4.2 Adaptador JPA/H2 (comparación didáctica)
**Entidad JPA (solo aquí usamos anotaciones JPA):**
```java
// infrastructure/jpa/entity/EstudianteEntity.java
package com.riwi.academico.infrastructure.jpa.entity;

import jakarta.persistence.*;

@Entity @Table(name="estudiantes")
public class EstudianteEntity {
  @Id private String id;
  @Column(nullable=false, unique=true) private String nombre;

  public String getId(){ return id; }
  public void setId(String id){ this.id = id; }
  public String getNombre(){ return nombre; }
  public void setNombre(String nombre){ this.nombre = nombre; }
}
```

**Repositorio Spring Data:**
```java
// infrastructure/jpa/repository/EstudianteJpaRepository.java
package com.riwi.academico.infrastructure.jpa.repository;

import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface EstudianteJpaRepository extends JpaRepository<EstudianteEntity, String> {
  boolean existsByNombreIgnoreCase(String nombre);
}
```

**Mapper (aisla dominio de JPA):**
```java
// infrastructure/jpa/adapter/EstudianteJpaMapper.java
package com.riwi.academico.infrastructure.jpa.adapter;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;

class EstudianteJpaMapper {
  static EstudianteEntity toEntity(Estudiante d){
    var e = new EstudianteEntity();
    e.setId(d.getId());
    e.setNombre(d.getNombre());
    return e;
  }
  static Estudiante toDomain(EstudianteEntity e){
    return new Estudiante(e.getId(), e.getNombre());
  }
}
```

**Adaptador que implementa el port usando JPA:**
```java
// infrastructure/jpa/adapter/EstudianteJpaAdapter.java
package com.riwi.academico.infrastructure.jpa.adapter;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import com.riwi.academico.infrastructure.jpa.repository.EstudianteJpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {

  private final EstudianteJpaRepository jpa;
  public EstudianteJpaAdapter(EstudianteJpaRepository jpa){ this.jpa = jpa; }

  @Override public Estudiante guardar(Estudiante e) {
    EstudianteEntity saved = jpa.save(EstudianteJpaMapper.toEntity(e));
    return EstudianteJpaMapper.toDomain(saved);
  }
  @Override public Optional<Estudiante> buscarPorId(String id) {
    return jpa.findById(id).map(EstudianteJpaMapper::toDomain);
  }
  @Override public boolean existePorNombre(String nombre) { return jpa.existsByNombreIgnoreCase(nombre); }
  @Override public List<Estudiante> listar() {
    return jpa.findAll().stream().map(EstudianteJpaMapper::toDomain).toList();
  }
}
```

> Reemplazar JPA por JDBC/Redis/S3 = crear **otro adaptador** que implemente el **mismo port**.

---

## 5) Wiring (configuración explícita de beans)

### 5.1 Con adaptador **InMemory** (dev/workshop)
```java
// infrastructure/config/BeanConfigMemory.java
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.application.usecase.ObtenerEstudianteUseCase;
import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.memory.adapter.EstudianteInMemoryAdapter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
@Profile("memory")
public class BeanConfigMemory {

  @Bean EstudianteRepositoryPort estudianteRepositoryPort(){
    return new EstudianteInMemoryAdapter();
  }

  @Bean RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo){
    return new RegistrarEstudianteUseCase(repo);
  }

  @Bean ObtenerEstudianteUseCase obtenerEstudianteUseCase(EstudianteRepositoryPort repo){
    return new ObtenerEstudianteUseCase(repo);
  }
}
```

### 5.2 Con adaptador **JPA** (real)
```java
// infrastructure/config/BeanConfigJpa.java
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.application.usecase.ObtenerEstudianteUseCase;
import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.jpa.adapter.EstudianteJpaAdapter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
@Profile("jpa")
public class BeanConfigJpa {

  // Spring inyecta el EstudianteJpaAdapter (tiene @Repository) como implementación del Port
  @Bean RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo){
    return new RegistrarEstudianteUseCase(repo);
  }
  @Bean ObtenerEstudianteUseCase obtenerEstudianteUseCase(EstudianteRepositoryPort repo){
    return new ObtenerEstudianteUseCase(repo);
  }
}
```

**Activar perfil:**
- VM: `-Dspring.profiles.active=memory` (o `jpa`)
- Env: `SPRING_PROFILES_ACTIVE=memory`

---

## 6) Entrypoint REST (controlador + DTOs + mapper)

### 6.1 DTOs Web
```java
// dto/EstudianteRequest.java
package com.riwi.academico.dto;

import jakarta.validation.constraints.NotBlank;
public class EstudianteRequest {
  @NotBlank(message="id requerido") public String id;
  @NotBlank(message="nombre requerido") public String nombre;
}
```

```java
// dto/EstudianteResponse.java
package com.riwi.academico.dto;

public class EstudianteResponse {
  public String id;
  public String nombre;
  public EstudianteResponse(String id, String nombre){ this.id=id; this.nombre=nombre; }
}
```

### 6.2 Mapper Web
```java
// entrypoints/rest/mapper/EstudianteWebMapper.java
package com.riwi.academico.entrypoints.rest.mapper;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.dto.EstudianteResponse;

public class EstudianteWebMapper {
  public static EstudianteResponse toResponse(Estudiante d){
    return new EstudianteResponse(d.getId(), d.getNombre());
  }
}
```

### 6.3 Controlador
```java
// entrypoints/rest/EstudianteController.java
package com.riwi.academico.entrypoints.rest;

import com.riwi.academico.application.usecase.ObtenerEstudianteUseCase;
import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.dto.EstudianteRequest;
import com.riwi.academico.entrypoints.rest.mapper.EstudianteWebMapper;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/estudiantes")
public class EstudianteController {

  private final RegistrarEstudianteUseCase registrar;
  private final ObtenerEstudianteUseCase obtener;

  public EstudianteController(RegistrarEstudianteUseCase registrar, ObtenerEstudianteUseCase obtener) {
    this.registrar = registrar; this.obtener = obtener;
  }

  @PostMapping
  public ResponseEntity<?> crear(@Valid @RequestBody EstudianteRequest req){
    var d = registrar.ejecutar(req.id, req.nombre);
    return ResponseEntity.status(201).body(EstudianteWebMapper.toResponse(d));
  }

  @GetMapping("/{id}")
  public ResponseEntity<?> porId(@PathVariable String id){
    var d = obtener.porId(id);
    return ResponseEntity.ok(EstudianteWebMapper.toResponse(d));
  }
}
```

---

## 7) Configuración `application.yml` (perfiles + H2/JPA)
```yaml
spring:
  application:
    name: academico
  datasource:
    url: jdbc:h2:mem:academico;DB_CLOSE_DELAY=-1
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop # solo laboratorio H2; PostgreSQL usa Flyway + validate
    properties:
      hibernate:
        format_sql: true

---
spring:
  config:
    activate:
      on-profile: memory

# (no datasource requerido; usamos adaptador en memoria)
logging:
  level:
    root: info

---
spring:
  config:
    activate:
      on-profile: jpa

# Usa el datasource definido arriba + JPA adapter
```

---

## 8) Dependencias Maven (extracto)
```xml
<dependencies>
  <!-- Web / Validación -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- JPA + H2 (si usas perfil jpa) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>

  <!-- Tests -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## 9) Estrategia de pruebas por capas (rápida y efectiva)

### 9.1 Unitarias de **casos de uso** (sin Spring)
```java
// test/application/usecase/RegistrarEstudianteUseCaseTest.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class RegistrarEstudianteUseCaseTest {

  @Test
  void registra_ok_si_no_duplicado(){
    var repo = mock(EstudianteRepositoryPort.class);
    when(repo.existePorNombre("Ana")).thenReturn(false);
    when(repo.guardar(any())).thenAnswer(inv -> inv.getArgument(0));
    var uc = new RegistrarEstudianteUseCase(repo);

    Estudiante e = uc.ejecutar("1","Ana");
    assertEquals("Ana", e.getNombre());
    verify(repo).guardar(any());
  }

  @Test
  void falla_si_nombre_duplicado(){
    var repo = mock(EstudianteRepositoryPort.class);
    when(repo.existePorNombre("Ana")).thenReturn(true);
    var uc = new RegistrarEstudianteUseCase(repo);
    assertThrows(IllegalArgumentException.class, () -> uc.ejecutar("1","Ana"));
  }
}
```

### 9.2 Web slice del **controlador** (MockMvc)
```java
// test/entrypoints/rest/EstudianteControllerTest.java
package com.riwi.academico.entrypoints.rest;

import com.riwi.academico.application.usecase.ObtenerEstudianteUseCase;
import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.model.Estudiante;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(EstudianteController.class)
class EstudianteControllerTest {

  @Autowired MockMvc mvc;
  @MockitoBean RegistrarEstudianteUseCase registrar;
  @MockitoBean ObtenerEstudianteUseCase obtener;

  @Test
  void crear_201() throws Exception {
    Mockito.when(registrar.ejecutar("1","Ana")).thenReturn(new Estudiante("1","Ana"));

    mvc.perform(post("/api/v1/estudiantes")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"id":"1","nombre":"Ana"}"))
      .andExpect(status().isCreated())
      .andExpect(jsonPath("$.id").value("1"))
      .andExpect(jsonPath("$.nombre").value("Ana"));
  }
}
```

### 9.3 (Opcional) Integración rápida con H2
- H2 permite feedback rápido, pero el entregable obligatorio ejecuta estas pruebas con PostgreSQL Testcontainers.
- Verifica que `EstudianteJpaAdapter` persiste y recupera correctamente.

---

## 10) Manejo global de errores (web)

```java
// infrastructure/web/GlobalExceptionHandler.java
package com.riwi.academico.infrastructure.web;

import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import java.time.Instant;
import java.util.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<Map<String,Object>> badRequest(MethodArgumentNotValidException ex){
    var fields = new LinkedHashMap<String,String>();
    ex.getBindingResult().getFieldErrors().forEach(f -> fields.put(f.getField(), f.getDefaultMessage()));
    return ResponseEntity.badRequest().body(Map.of(
      "timestamp", Instant.now().toString(),
      "status", 400, "error","Bad Request",
      "fields", fields));
  }

  @ExceptionHandler(IllegalArgumentException.class)
  ResponseEntity<Map<String,Object>> conflict(IllegalArgumentException ex){
    return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of(
      "timestamp", Instant.now().toString(), "status",409, "error","Conflict", "message", ex.getMessage()));
  }
}
```

---

## 11) Laboratorio guiado (45–60 min)

1. **Crea la estructura** de paquetes como arriba.
2. Implementa **dominio** (entidad + port).
3. Implementa **use cases** (Registrar/Obtener).
4. Crea **adaptador InMemory** y el **BeanConfigMemory**. Activa perfil `memory`.
5. Expone **REST** (POST `/api/v1/estudiantes`, GET `/api/v1/estudiantes/{id}`) con DTOs.
6. Agrega **GlobalExceptionHandler**.
7. Escribe **tests** unitarios de los casos de uso y un web-slice del controlador.
8. Cambia a perfil `jpa`: agrega entidad, repo Spring Data y adaptador JPA. **Sin tocar** casos de uso ni controladores.
9. Verifica que todo sigue funcionando.

---

## 12) Diferencias prácticas vs arquitectura en capas

| Aspecto | Capas tradicional | Clean Architecture |
|---|---|---|
| Reemplazar BD | Mucho refactor | Cambias adaptador |
| Tests de dominio | Duro (Spring/JPA por medio) | Fácil (POJOs + mocks) |
| Dependencia de framework | Alta | Baja en dominio/app |
| Evolución a microservicios | Costosa | Natural (Bounded Contexts) |

---

## 13) Consejos finales

- **Dominio sin Spring**: cero anotaciones.
- **Puertos pequeños** y específicos.
- **Adaptadores delgados** (mapeo + repositorio).
- **Wiring por perfiles** para alternar infra en 1 clic.
- **Tests rápidos** primero en Application/Dominio; integra después.

---

## 14) Resultado esperado

- Proyecto con dominio **independiente**, casos de uso **testeables**, y adaptadores **intercambiables** (memoria/JPA).
- Controladores REST limpios y manejo de errores homogéneo.
- Base sólida para añadir seguridad (JWT), colas (RabbitMQ) o cache (Redis) sin tocar el **core**.
