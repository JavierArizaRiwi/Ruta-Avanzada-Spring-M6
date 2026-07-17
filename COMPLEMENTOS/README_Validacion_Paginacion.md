
# Persistencia JPA, validación REST, errores y paginación con Spring Boot 4

> **Estado curricular:** material consolidado en Semanas 2–4. La línea actual usa Spring Boot 4.1.x, RFC 9457, PostgreSQL/Flyway y Testcontainers; H2 queda como alternativa didáctica limitada.

Esta guía enseña cómo construir una API REST profesional en Spring Boot, enfocada en:

- Uso correcto de **códigos HTTP**
- Validaciones en DTOs con **Bean Validation**
- Limpieza de código con **Lombok**
- Manejo estructurado de errores con `@ExceptionHandler` + `ProblemDetail` (RFC-7807)
- Implementación de **paginación** con `Pageable` y `Page<T>`

Los ejemplos se basan en un recurso común: **Usuarios**.

---

## 1. Dependencias esenciales (pom.xml)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>

  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

---

## 2. Configuración H2 para desarrollo

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:app_db
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop # solo laboratorio H2
    open-in-view: false
  h2:
    console:
      enabled: true
      path: /h2
```

Acceder a consola:  
http://localhost:8080/h2

---

## 3. Entidad Usuario

```java
@Entity
@Table(name = "usuarios")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Usuario {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String nombre;

    @Column(nullable = false, unique = true, length = 180)
    private String email;
}
```

---

## 4. DTOs con Validación Bean Validation

```java
public record UsuarioRequest(
        @NotBlank @Size(max = 120) String nombre,
        @NotBlank @Email @Size(max = 180) String email
) {}

public record UsuarioResponse(Long id, String nombre, String email) {}
```

---

## 5. Repositorio JPA

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    boolean existsByEmailIgnoreCase(String email);
}
```

---

## 6. Servicio con lógica de dominio

```java
@Service
@RequiredArgsConstructor
public class UsuarioService {

    private final UsuarioRepository repo;

    public UsuarioResponse crear(UsuarioRequest req) {
        if (repo.existsByEmailIgnoreCase(req.email()))
            throw new ConflictException("El email ya existe");

        Usuario saved = repo.save(Usuario.builder()
                .nombre(req.nombre())
                .email(req.email())
                .build());

        return new UsuarioResponse(saved.getId(), saved.getNombre(), saved.getEmail());
    }

    public UsuarioResponse get(Long id) {
        return repo.findById(id)
            .map(u -> new UsuarioResponse(u.getId(), u.getNombre(), u.getEmail()))
            .orElseThrow(() -> new NotFoundException("Usuario no encontrado"));
    }

    public Page<UsuarioResponse> listar(Pageable pageable) {
        return repo.findAll(pageable)
            .map(u -> new UsuarioResponse(u.getId(), u.getNombre(), u.getEmail()));
    }
}
```

---

## 7. Controlador REST + Códigos HTTP Correctos

```java
@RestController
@RequestMapping("/api/v1/usuarios")
@RequiredArgsConstructor
public class UsuarioController {

    private final UsuarioService service;

    @PostMapping
    public ResponseEntity<ApiResponse<UsuarioResponse>> crear(
            @Valid @RequestBody UsuarioRequest req,
            UriComponentsBuilder uri) {

        UsuarioResponse res = service.crear(req);
        URI location = uri.path("/api/v1/usuarios/{id}")
                .buildAndExpand(res.id()).toUri();

        return ResponseEntity.created(location)
                .body(ApiResponse.success(res, "Usuario creado"));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UsuarioResponse>> get(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(service.get(id), null));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<List<UsuarioResponse>>> listar(Pageable pageable) {
        Page<UsuarioResponse> page = service.listar(pageable);
        return ResponseEntity.ok(ApiResponse.withPage(page.getContent(), page));
    }
}
```

---

## 8. Envelope estándar de éxito

```java
public record ApiResponse<T>(String status, T data, Meta meta) {

    public static <T> ApiResponse<T> success(T data, String msg) {
        return new ApiResponse<>("success", data, new Meta(msg, null));
    }

    public static <T> ApiResponse<T> withPage(T data, Page<?> p) {
        Pagination pg = new Pagination(
                p.getNumber(), p.getSize(), p.getTotalElements(), p.getTotalPages());
        return new ApiResponse<>("success", data, new Meta(null, pg));
    }

    public record Meta(String message, Pagination page) {}
    public record Pagination(int page, int size, long totalElements, int totalPages) {}
}
```

---

## 9. Excepciones personalizadas

```java
public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) { super(message); }
}

public class ConflictException extends RuntimeException {
    public ConflictException(String message) { super(message); }
}
```

---

## 10. Manejo de errores global (RFC 9457)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail notFound(NotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("No encontrado");
        pd.setDetail(ex.getMessage());
        return pd;
    }

    @ExceptionHandler(ConflictException.class)
    public ProblemDetail conflict(ConflictException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setTitle("Conflicto");
        pd.setDetail(ex.getMessage());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail validation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Error en los datos enviados");

        List<Map<String, Object>> errors = ex.getBindingResult()
                .getFieldErrors().stream()
                .map(error -> Map.of(
                        "field", error.getField(),
                        "message", Optional.ofNullable(error.getDefaultMessage()).orElse("invalid"),
                        "rejectedValue", error.getRejectedValue()))
                .toList();

        pd.setProperty("errors", errors);
        return pd;
    }
}
```

---

## 11. Paginación en REST

Ejemplo de consumo:
```bash
GET /api/v1/usuarios?page=0&size=5&sort=nombre,asc
```

Respuesta estructurada:

```json
{
  "status": "success",
  "data": [
    { "id": 1, "nombre": "Ana", "email": "ana@mail.com" }
  ],
  "meta": {
    "page": {
      "page": 0,
      "size": 5,
      "totalElements": 12,
      "totalPages": 3
    }
  }
}
```

---

## 12. Checklist profesional

- Validar siempre **DTOs**, no entidades
- `201 Created` con `Location` en creación
- `ProblemDetail` para todos los errores
- Paginación en GET de colecciones
- Contrato consistente de éxito y error

---

## 13. Antipatrones a evitar

- Responder `200 OK` cuando hay error
- Devolver entidades JPA directamente
- No usar paginación en recursos múltiples
- Mezclar mensajes sin estructura en errores

---

Fin del documento.
