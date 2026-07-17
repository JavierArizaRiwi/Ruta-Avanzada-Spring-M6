# Refuerzo — Testing completo sobre el dominio de pedidos

> **Estado curricular:** práctica adicional. El núcleo obligatorio de testing está en `SESIONES/Semana4/README_Sesion_S4Dia3.md` y usa Riwi Learning Platform.

En esta sesión construirás una **base sólida de calidad** para microservicios Spring Boot con enfoque **hexagonal**: pruebas **unitarias** (casos de uso) y **web-slice** (controladores) con **JUnit 5** y **Mockito**, **cobertura** con **JaCoCo**, y **análisis estático** con **SonarLint** (y opcionalmente SonarQube Community local).  
Usaremos un microservicio educativo: **`pedidos-service`**. Todo acceso externo real a la plataforma se hace **a través del API Gateway** (ver Día 2 y 3), pero en este día nos enfocamos en **tests locales del servicio**.

---

## 1) Objetivos del día

- Diseñar una **estrategia de pruebas** para microservicios hexagonales (unitarias de **casos de uso**, web-slice del **adaptador REST**).  
- Escribir pruebas con **JUnit 5** y **Mockito** cubriendo lógica de negocio y errores.  
- Medir **cobertura** con **JaCoCo** y comprender métricas clave.  
- Integrar **SonarLint** en el IDE y (opcional) **SonarQube** local con Maven.  
- Mantener **dominio y casos de uso libres de framework**; adapters finos y testeables.

---

## 2) Estructura de proyecto (hexagonal e impecable)

```
pedidos-service/
 ├─ src/
 │  ├─ main/java/com/riwi/pedidos
 │  │   ├─ domain/                     # Entidades y reglas puras
 │  │   ├─ application/
 │  │   │    ├─ ports/                 # Puertos (interfaces)
 │  │   │    └─ usecase/               # Casos de uso (servicios de aplicación)
 │  │   ├─ infrastructure/
 │  │   │    ├─ adapters/
 │  │   │    │    ├─ in/rest/          # Controladores (Spring MVC)
 │  │   │    │    └─ out/persistence/  # Repos (JPA/H2 o fake educativo)
 │  │   │    └─ config/                # Configuración general
 │  │   └─ dto/                        # DTOs request/response
 │  └─ test/java/com/riwi/pedidos
 │      ├─ application/usecase/        # Tests unitarios de casos de uso
 │      └─ infrastructure/adapters/in/rest/ # Tests WebSlice (MockMvc)
 └─ pom.xml
```

**Principio guía:** cada paquete tiene su espejo en `test/` con nombres de pruebas coherentes.

---

## 3) Dependencias y plugins (pom.xml)

```xml
<dependencies>
  <!-- API web y validación (para el adapter REST) -->
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
</dependencies>

<build>
  <plugins>
    <!-- Ejecuta tests JUnit 5 -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
      <configuration>
        <includes>
          <include>**/*Test.java</include>
          <include>**/*Tests.java</include>
          <include>**/*IT.java</include>
        </includes>
      </configuration>
    </plugin>

    <!-- Cobertura JaCoCo -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.11</version>
      <executions>
        <execution>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals><goal>report</goal></goals>
        </execution>
        <execution>
          <id>check</id>
          <goals><goal>check</goal></goals>
          <configuration>
            <rules>
              <rule>
                <element>CLASS</element>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>
                  </limit>
                </limits>
              </rule>
            </rules>
          </configuration>
        </execution>
      </executions>
    </plugin>

    <!-- (Opcional) SonarQube Maven -->
    <plugin>
      <groupId>org.sonarsource.scanner.maven</groupId>
      <artifactId>sonar-maven-plugin</artifactId>
      <version>3.10.0.2594</version>
    </plugin>
  </plugins>
</build>
```

**Notas:**  
- JaCoCo generará `target/site/jacoco/index.html`.  
- La regla mínima de cobertura (80 %) es **educativa**; puedes ajustar por paquete.

---

## 4) Dominio, DTOs y contrato REST (adapters in)

### 4.1 Dominio simple
```java
// domain/Pedido.java
package com.riwi.pedidos.domain;

import java.util.Objects;

public class Pedido {
  private Long id;
  private String producto;
  private int cantidad;

  public Pedido(Long id, String producto, int cantidad) {
    this.id = id; this.producto = producto; this.cantidad = cantidad;
  }
  public Long getId() { return id; }
  public String getProducto() { return producto; }
  public int getCantidad() { return cantidad; }

  @Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Pedido)) return false;
    Pedido p = (Pedido) o;
    return cantidad == p.cantidad && Objects.equals(id, p.id) && Objects.equals(producto, p.producto);
  }
  @Override public int hashCode() { return Objects.hash(id, producto, cantidad); }
}
```

### 4.2 DTOs con validación
```java
// dto/PedidoRequest.java
package com.riwi.pedidos.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;

public class PedidoRequest {
  @NotBlank(message = "producto es obligatorio")
  private String producto;

  @Min(value = 1, message = "cantidad debe ser >= 1")
  private int cantidad;

  public String getProducto() { return producto; }
  public void setProducto(String producto) { this.producto = producto; }
  public int getCantidad() { return cantidad; }
  public void setCantidad(int cantidad) { this.cantidad = cantidad; }
}
```

```java
// dto/PedidoResponse.java
package com.riwi.pedidos.dto;

public class PedidoResponse {
  private Long id;
  private String producto;
  private int cantidad;

  public PedidoResponse(Long id, String producto, int cantidad) {
    this.id = id; this.producto = producto; this.cantidad = cantidad;
  }
  public Long getId(){ return id; }
  public String getProducto(){ return producto; }
  public int getCantidad(){ return cantidad; }
}
```

---

## 5) Puertos y Casos de Uso (application)

### 5.1 Puertos
```java
// application/ports/PedidoRepositoryPort.java
package com.riwi.pedidos.application.ports;

import com.riwi.pedidos.domain.Pedido;
import java.util.List;
import java.util.Optional;

public interface PedidoRepositoryPort {
  Pedido save(Pedido p);
  Optional<Pedido> findById(Long id);
  List<Pedido> findAll();
}
```

### 5.2 Caso de Uso (servicio de aplicación, libre de framework)
```java
// application/usecase/PedidoUseCase.java
package com.riwi.pedidos.application.usecase;

import com.riwi.pedidos.application.ports.PedidoRepositoryPort;
import com.riwi.pedidos.domain.Pedido;

import java.util.List;
import java.util.NoSuchElementException;

public class PedidoUseCase {

  private final PedidoRepositoryPort repo;
  public PedidoUseCase(PedidoRepositoryPort repo) { this.repo = repo; }

  public Pedido registrar(String producto, int cantidad) {
    if (cantidad <= 0) throw new IllegalArgumentException("cantidad inválida");
    var pedido = new Pedido(null, producto, cantidad);
    return repo.save(pedido);
  }

  public Pedido obtenerPorId(Long id) {
    return repo.findById(id).orElseThrow(() -> new NoSuchElementException("pedido no encontrado"));
  }

  public List<Pedido> listar() { return repo.findAll(); }
}
```

*(Puedes crear un `@Configuration` que exponga `PedidoUseCase` como `@Bean` para inyectarlo en el controller.)*

---

## 6) Adaptadores (in/out)

### 6.1 Adaptador de entrada (REST Controller)
```java
// infrastructure/adapters/in/rest/PedidoController.java
package com.riwi.pedidos.infrastructure.adapters.in.rest;

import com.riwi.pedidos.application.usecase.PedidoUseCase;
import com.riwi.pedidos.dto.PedidoRequest;
import com.riwi.pedidos.dto.PedidoResponse;
import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/pedidos")
public class PedidoController {

  private final PedidoUseCase useCase;
  public PedidoController(PedidoUseCase useCase) { this.useCase = useCase; }

  @PostMapping
  public ResponseEntity<PedidoResponse> crear(@Valid @RequestBody PedidoRequest req) {
    var p = useCase.registrar(req.getProducto(), req.getCantidad());
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(new PedidoResponse(p.getId(), p.getProducto(), p.getCantidad()));
  }

  @GetMapping("/{id}")
  public ResponseEntity<PedidoResponse> obtener(@PathVariable Long id) {
    var p = useCase.obtenerPorId(id);
    return ResponseEntity.ok(new PedidoResponse(p.getId(), p.getProducto(), p.getCantidad()));
  }

  @GetMapping
  public ResponseEntity<List<PedidoResponse>> listar() {
    var res = useCase.listar().stream()
        .map(p -> new PedidoResponse(p.getId(), p.getProducto(), p.getCantidad()))
        .toList();
    return ResponseEntity.ok(res);
  }
}
```

### 6.2 Adaptador de salida (Repositorio educativo in‑memory)
```java
// infrastructure/adapters/out/persistence/InMemoryPedidoRepository.java
package com.riwi.pedidos.infrastructure.adapters.out.persistence;

import com.riwi.pedidos.application.ports.PedidoRepositoryPort;
import com.riwi.pedidos.domain.Pedido;
import org.springframework.stereotype.Repository;

import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class InMemoryPedidoRepository implements PedidoRepositoryPort {

  private final Map<Long, Pedido> data = new LinkedHashMap<>();
  private final AtomicLong seq = new AtomicLong(0);

  @Override
  public Pedido save(Pedido p) {
    long id = seq.incrementAndGet();
    var saved = new Pedido(id, p.getProducto(), p.getCantidad());
    data.put(id, saved);
    return saved;
  }

  @Override public Optional<Pedido> findById(Long id) { return Optional.ofNullable(data.get(id)); }
  @Override public List<Pedido> findAll() { return new ArrayList<>(data.values()); }
}
```

### 6.3 Wiring (configuración de beans)
```java
// infrastructure/config/AppConfig.java
package com.riwi.pedidos.infrastructure.config;

import com.riwi.pedidos.application.ports.PedidoRepositoryPort;
import com.riwi.pedidos.application.usecase.PedidoUseCase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {
  @Bean
  public PedidoUseCase pedidoUseCase(PedidoRepositoryPort repo) {
    return new PedidoUseCase(repo);
  }
}
```

### 6.4 Manejo global de errores
```java
// infrastructure/config/GlobalExceptionHandler.java
package com.riwi.pedidos.infrastructure.config;

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

## 7) Pruebas **unitarias** de casos de uso (JUnit 5 + Mockito)

```java
// test/application/usecase/PedidoUseCaseTest.java
package com.riwi.pedidos.application.usecase;

import com.riwi.pedidos.application.ports.PedidoRepositoryPort;
import com.riwi.pedidos.domain.Pedido;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;

import java.util.List;
import java.util.NoSuchElementException;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class PedidoUseCaseTest {

  @Mock private PedidoRepositoryPort repo;
  private PedidoUseCase useCase;

  @BeforeEach void init() {
    MockitoAnnotations.openMocks(this);
    useCase = new PedidoUseCase(repo);
  }

  @Test
  void registrar_ok() {
    when(repo.save(any())).thenAnswer(inv -> {
      Pedido p = inv.getArgument(0);
      return new Pedido(1L, p.getProducto(), p.getCantidad());
    });

    var r = useCase.registrar("Laptop", 1);

    assertEquals(1L, r.getId());
    assertEquals("Laptop", r.getProducto());
    assertEquals(1, r.getCantidad());
    verify(repo).save(any());
  }

  @Test
  void registrar_cantidadInvalida_lanzaExcepcion() {
    assertThrows(IllegalArgumentException.class, () -> useCase.registrar("Laptop", 0));
    verify(repo, never()).save(any());
  }

  @Test
  void obtenerPorId_ok() {
    when(repo.findById(10L)).thenReturn(Optional.of(new Pedido(10L, "Mouse", 2)));
    var p = useCase.obtenerPorId(10L);
    assertEquals("Mouse", p.getProducto());
  }

  @Test
  void obtenerPorId_noExiste() {
    when(repo.findById(99L)).thenReturn(Optional.empty());
    assertThrows(NoSuchElementException.class, () -> useCase.obtenerPorId(99L));
  }

  @Test
  void listar_ok() {
    when(repo.findAll()).thenReturn(List.of(new Pedido(1L, "A", 1), new Pedido(2L, "B", 2)));
    var lista = useCase.listar();
    assertEquals(2, lista.size());
  }
}
```

---

## 8) Pruebas **web-slice** del adaptador REST (MockMvc)

```java
// test/infrastructure/adapters/in/rest/PedidoControllerTest.java
package com.riwi.pedidos.infrastructure.adapters.in.rest;

import com.riwi.pedidos.application.usecase.PedidoUseCase;
import com.riwi.pedidos.domain.Pedido;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(PedidoController.class)
class PedidoControllerTest {

  @Autowired private MockMvc mvc;
  @MockBean private PedidoUseCase useCase;

  @Test
  void crear_201() throws Exception {
    Mockito.when(useCase.registrar("Laptop", 1)).thenReturn(new Pedido(1L, "Laptop", 1));

    mvc.perform(post("/api/v1/pedidos")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"producto":"Laptop","cantidad":1}"))
      .andExpect(status().isCreated())
      .andExpect(jsonPath("$.id").value(1))
      .andExpect(jsonPath("$.producto").value("Laptop"))
      .andExpect(jsonPath("$.cantidad").value(1));
  }

  @Test
  void crear_400_validacion() throws Exception {
    mvc.perform(post("/api/v1/pedidos")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{"producto":"","cantidad":0}"))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.error").value("Bad Request"));
  }
}
```

---

## 9) Cobertura con JaCoCo y reporte

- Ejecuta: `mvn clean test`  
- Abre: `target/site/jacoco/index.html`  
- Revisa paquetes con menor cobertura y agrega pruebas hasta alcanzar el **80 %** (o la meta que definas).

**Consejos:** cubre **rutas felices** y **errores** (excepciones, entradas inválidas).

---

## 10) SonarLint (IDE) y SonarQube (opcional)

### 10.1 SonarLint (recomendado)
- Instala el plugin y habilítalo para el proyecto.  
- Corrige *code smells* (nombres, duplicación de literales, complejidad).

### 10.2 SonarQube Community (opcional, local)
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Maven:
```bash
mvn clean verify sonar:sonar   -Dsonar.host.url=http://localhost:9000   -Dsonar.projectKey=pedidos-service   -Dsonar.login=<TOKEN>
```

**Métricas educativas mínimas:**

| Métrica | Meta |
|--------|------|
| Cobertura (casos de uso y adapters in) | ≥ 80 % |
| Bugs/Vuln críticas | 0 |
| Duplicación | ≤ 5 % |

---

## 11) Guía de estilo de pruebas (impecable)

- Nombres: `metodo_condicion_resultado` (p. ej., `registrar_cantidadInvalida_lanzaExcepcion`).  
- AAA (Arrange-Act-Assert) bien delimitado.  
- **No** mockear lo que no posees (evita mocks de clases estándar).  
- Usa `ArgumentCaptor` cuando necesites verificar el objeto enviado al repo.  
- Evita *sleep* y dependencias en tiempo real; usa datos deterministas.  
- Pruebas **idempotentes** y **aisladas** (sin leer/escribir archivos externos).

**Ejemplo con `ArgumentCaptor`:**
```java
@Captor ArgumentCaptor<Pedido> pedidoCaptor;

@Test
void registrar_enviaEntidadCorrectaAlRepositorio() {
  when(repo.save(any())).thenReturn(new Pedido(5L, "Teclado", 3));
  useCase.registrar("Teclado", 3);
  verify(repo).save(pedidoCaptor.capture());
  assertEquals("Teclado", pedidoCaptor.getValue().getProducto());
  assertEquals(3, pedidoCaptor.getValue().getCantidad());
}
```

---

## 12) Laboratorio guiado (45–60 min)

1. Crear el proyecto `pedidos-service` con estructura **hexagonal**.  
2. Implementar `PedidoUseCase` y `PedidoController`.  
3. Escribir **5 pruebas unitarias** (rutas felices y errores).  
4. Escribir **2 pruebas web** con `@WebMvcTest` (201 y 400).  
5. Ejecutar `mvn clean test` y revisar JaCoCo.  
6. Instalar SonarLint, corregir al menos **3** hallazgos.  
7. (Opcional) Ejecutar SonarQube local y publicar análisis Maven.

---

## 13) Checklist del día

- Estructura hexagonal limpia y espejo en `test/`.  
- Casos de uso probados (JUnit 5 + Mockito) con ≥ 80 % de cobertura.  
- Controlador probado con MockMvc y validaciones activas.  
- Reporte JaCoCo generado y revisado.  
- SonarLint activo y hallazgos críticos resueltos.

---

## 14) Resultado esperado

- Microservicio `pedidos-service` con pruebas **unitarias (use case)** y **web** (adapter REST) correctas.  
- **Cobertura y calidad** visibles (JaCoCo + SonarLint).  
- Base lista para **Config/Eureka/Gateway** (Día 2) y **Contratos/Health** (Día 3), consumiendo externamente **vía Gateway**.
