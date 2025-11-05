
# Guía Profesional: Respuestas en APIs Spring Boot

## Introducción

Una API profesional debe responder de forma consistente, clara y alineada a estándares internacionales. En Spring Boot, esto implica usar contratos uniformes para éxito y error, códigos HTTP correctos, metadatos útiles y pruebas automatizadas. Esta guía te enseña cómo lograrlo con ejemplos, teoría y buenas prácticas.

---

## Objetivos

- Definir un contrato uniforme de éxito: `ApiResponse<T>` con `status`, `data` y `meta` (mensaje, `traceId`, versión, paginación).
- Implementar un manejador global de errores con `@RestControllerAdvice` y `ProblemDetail` (RFC 7807).
- Usar códigos HTTP correctos: `200/201/204` para éxito; `400/401/403/404/409/422/500` para errores.
- Proveer ejemplos de controladores, DTOs, validación y pruebas.

---

## Contrato de Éxito: Envelope Genérico

### ApiResponse<T>
```java
public record ApiResponse<T>(
    String status, // "success"
    T data,
    Meta meta
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>("success", data, null);
    }
    public static <T> ApiResponse<T> withMeta(T data, Meta meta) {
        return new ApiResponse<>("success", data, meta);
    }

    public record Meta(
        String message,
        String traceId,
        String version,
        Pagination page // opcional
    ) {}

    public record Pagination(
        int page, int size, long totalElements, int totalPages
    ) {}
}
```

**Puntos clave:**
- No mezcles ProblemDetail (error) con este envelope de éxito.
- Incluye `traceId` para correlación de logs y soporte.
- La paginación se modela en `meta.page`.

---

## Códigos HTTP y Headers

- **200 OK**: GET exitoso.
- **201 Created** + `Location`: recurso creado.
- **204 No Content**: operación sin cuerpo (DELETE).
- **400 Bad Request**: entrada inválida.
- **401/403**: autenticación/autorización fallida.
- **404**: recurso no encontrado.
- **409 Conflict**: colisión de estado.
- **422 Unprocessable Entity**: regla de negocio incumplida.
- **500 Internal Server Error**: error no controlado.

**Headers útiles:**
- `Location`: URI del recurso creado.
- `ETag`/`If-Match`: concurrencia optimista.
- `X-Trace-Id`: correlación de logs.

---

## Ejemplos de Controladores

### GET por id (200)
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    private final UserService service;
    public UserController(UserService service) { this.service = service; }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UserDto>> get(@PathVariable Long id) {
        UserDto dto = service.get(id); // lanza NotFoundException si no existe
        return ResponseEntity.ok(ApiResponse.ok(dto));
    }
}
```

### POST crear (201 + Location)
```java
@PostMapping
public ResponseEntity<ApiResponse<UserDto>> create(
        @Valid @RequestBody CreateUserReq req,
        UriComponentsBuilder uri) {

    UserDto created = service.create(req);
    URI location = uri.path("/api/v1/users/{id}")
                      .buildAndExpand(created.id()).toUri();

    var meta = new ApiResponse.Meta(
            "Usuario creado correctamente",
            Trace.currentId(), "v1", null
    );

    return ResponseEntity.created(location)
                         .body(ApiResponse.withMeta(created, meta));
}
```

### DELETE (204)
```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return ResponseEntity.noContent().build();
}
```

### Listado paginado (200 + meta.page)
```java
@GetMapping
public ResponseEntity<ApiResponse<List<UserDto>>> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Page<UserDto> result = service.list(PageRequest.of(page, size));

    var meta = new ApiResponse.Meta(
            "Listado de usuarios",
            Trace.currentId(), "v1",
            new ApiResponse.Pagination(
                    result.getNumber(),
                    result.getSize(),
                    result.getTotalElements(),
                    result.getTotalPages()
            )
    );
    return ResponseEntity.ok(ApiResponse.withMeta(result.getContent(), meta));
}
```

---

## Contrato de Error: RFC 7807 y ProblemDetail

### Principios
- Usa `ProblemDetail` (Spring Boot 3) para todos los errores.
- Campos estándar: `type`, `title`, `status`, `detail`, `instance`.
- Extiende con `code` (código interno), `errors` (validación), `traceId`.

### Excepciones de dominio
```java
public class NotFoundException extends RuntimeException {
    private final String code;
    public NotFoundException(String message, String code) {
        super(message); this.code = code;
    }
    public String code() { return code; }
}

public class BusinessException extends RuntimeException {
    private final String code;
    public BusinessException(String message, String code) {
        super(message); this.code = code;
    }
    public String code() { return code; }
}
```

### Manejador global de errores
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail handleNotFound(NotFoundException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setType(URI.create("/errors/not-found"));
        pd.setTitle("Recurso no encontrado");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", ex.code());
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setType(URI.create("/errors/business"));
        pd.setTitle("Regla de negocio incumplida");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", ex.code());
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("/errors/validation"));
        pd.setTitle("Error de validación");
        pd.setDetail("Uno o más campos son inválidos");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());

        List<Map<String, Object>> errors = ex.getBindingResult()
            .getFieldErrors().stream()
            .map(fe -> Map.of(
                "field", fe.getField(),
                "message", fe.getDefaultMessage(),
                "rejectedValue", fe.getRejectedValue()
            ))
            .toList();
        pd.setProperty("errors", errors);
        return pd;
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ProblemDetail handleUnreadable(HttpMessageNotReadableException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("/errors/bad-request"));
        pd.setTitle("Solicitud mal formada");
        pd.setDetail("El cuerpo de la solicitud no pudo parsearse");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setType(URI.create("/errors/internal"));
        pd.setTitle("Error interno");
        pd.setDetail("Ha ocurrido un error inesperado");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        // log.error("Unhandled", ex) con traceId
        return pd;
    }
}
```

### Ejemplo de respuesta de error (JSON)
```json
{
  "type": "/errors/validation",
  "title": "Error de validación",
  "status": 400,
  "detail": "Uno o más campos son inválidos",
  "errors": [
    { "field": "email", "message": "must be a well-formed email address", "rejectedValue": "foo" }
  ],
  "traceId": "a3c4e0b1f2...",
  "instance": "/api/v1/users"
}
```

---

## DTOs y Validación

```java
public record CreateUserReq(
    @NotBlank String name,
    @Email String email,
    @Min(18) int age
) {}

public record UserDto(Long id, String name, String email, int age) {}
```

- La validación fallida dispara `MethodArgumentNotValidException` y se captura en el manejador global.
- Mantén los mensajes en `messages.properties` para internacionalización.

---

## Trazabilidad y Consistencia

```java
public final class Trace {
    private Trace() {}
    public static String currentId() {
        return Optional.ofNullable(org.slf4j.MDC.get("traceId"))
                       .orElse(UUID.randomUUID().toString());
    }
}
```

**Buenas prácticas:**
- Poner `traceId` en el body y/o header.
- Propagarlo a llamadas externas.
- Loguear si cae en el manejador global.

---

## Versionado, Contenido y HATEOAS

- Prefija rutas: `/api/v1/...`.
- Usa `application/json` como Content-Type.
- Links/HATEOAS si tu cliente lo necesita.
- ETags para caching o concurrencia optimista.

---

## Pruebas de Contrato

### Éxito con MockMvc
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService service;

    @Test
    void get_ok() throws Exception {
        when(service.get(1L)).thenReturn(new UserDto(1L,"Ana","ana@acme.com",25));

        mvc.perform(get("/api/v1/users/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.status").value("success"))
           .andExpect(jsonPath("$.data.id").value(1));
    }
}
```

### Error de validación
```java
@Test
void create_validation_error() throws Exception {
    var body = """
      {"name": "", "email": "bad", "age": 10}
    """;

    mvc.perform(post("/api/v1/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(body))
       .andExpect(status().isBadRequest())
       .andExpect(jsonPath("$.type").value("/errors/validation"))
       .andExpect(jsonPath("$.errors").isArray());
}
```

---

## Antipatrones y Advertencias

- No mezcles envelopes y ProblemDetail en una misma respuesta.
- No encapsules ProblemDetail dentro de un envelope de éxito.
- Usa el código HTTP correcto, no solo 400 para todo.
- No ocultes traceId.
- No devuelvas 200 con "status":"error" en el body.
- Incluye Location en 201 Created.

---

## Checklist de Adopción

- [ ] ApiResponse<T> implementado en todas las respuestas de éxito.
- [ ] Manejador global con ProblemDetail para errores comunes.
- [ ] Status codes correctos y Location en creaciones.
- [ ] traceId en body/header y logs correlacionados.
- [ ] Validación con Bean Validation.
- [ ] Tests MockMvc para éxito y error.
- [ ] Documentación OpenAPI actualizada.

---

## Plantilla mínima (pom.xml)
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

## Preguntas Frecuentes y Recomendaciones

**¿Debo usar Problem Spring Web?**
Solo si necesitas personalización avanzada; para la mayoría de casos, ProblemDetail es suficiente.

**¿Dónde poner traceId?**
En el body, en el header y en los logs.

**¿Cómo versiono mi API?**
Prefija las rutas y documenta los cambios en OpenAPI.

**¿Qué hago si mi cliente necesita links?**
Agrega HATEOAS solo si es necesario; no es obligatorio si tienes buena documentación.

**¿Cómo valido los contratos?**
Usa MockMvc y aserciones sobre el shape del JSON.

---

¿Dudas o quieres ejemplos específicos? ¡Solicítalos!

---

## 1) Contrato de **éxito** (envelope)

### 1.1 `ApiResponse<T>` (genérico y extensible)
```java
public record ApiResponse<T>(
        String status,  // "success"
        T data,
        Meta meta
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>("success", data, null);
    }
    public static <T> ApiResponse<T> withMeta(T data, Meta meta) {
        return new ApiResponse<>("success", data, meta);
    }

    public record Meta(
            String message,
            String traceId,
            String version,
            Pagination page // opcional
    ) {}

    public record Pagination(
            int page, int size, long totalElements, int totalPages
    ) {}
}
```

**Notas clave**
- No mezcles **ProblemDetail** (error) con este envelope (éxito).  
- Incluye `traceId` (correlación de logs) y `version` (versión del API).  
- La paginación se modela en `meta.page` —no dupliques datos en el cuerpo.

---

## 2) Códigos HTTP y headers (reglas de oro)

- **200 OK** → GET/acciones con cuerpo.  
- **201 Created** + **`Location`** → creación de recurso.  
- **204 No Content** → operaciones sin cuerpo (DELETE, toggles sin payload).  
- **400 Bad Request** → entrada inválida o JSON mal formado.  
- **401/403** → autenticación/autorización.  
- **404** → no encontrado.  
- **409 Conflict** → colisión de estado (ej. nombre único).  
- **422 Unprocessable Entity** → regla de negocio incumplida (opcional, si tu estándar lo usa).  
- **500** → error no controlado.

**Headers útiles**  
- `Location`: URI del recurso creado.  
- `ETag`/`If-Match`: control de concurrencia optimista.  
- `X-Trace-Id`: opcional si además lo pones en el body (`meta.traceId`).

---

## 3) Éxito en controladores (snippets listos)

### 3.1 GET por id (200)
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService service;

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UserDto>> get(@PathVariable Long id) {
        UserDto dto = service.get(id); // lanza NotFoundException si no existe
        return ResponseEntity.ok(ApiResponse.ok(dto));
    }
}
```

### 3.2 POST crear (201 + Location)
```java
@PostMapping
public ResponseEntity<ApiResponse<UserDto>> create(
        @Valid @RequestBody CreateUserReq req,
        UriComponentsBuilder uri) {

    UserDto created = service.create(req);
    URI location = uri.path("/api/v1/users/{id}")
                      .buildAndExpand(created.id()).toUri();

    var meta = new ApiResponse.Meta(
            "Usuario creado correctamente",
            Trace.currentId(), "v1", null
    );

    return ResponseEntity.created(location)
                         .body(ApiResponse.withMeta(created, meta));
}
```

### 3.3 DELETE (204)
```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return ResponseEntity.noContent().build();
}
```

### 3.4 Listado paginado (200 + `meta.page`)
```java
@GetMapping
public ResponseEntity<ApiResponse<List<UserDto>>> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Page<UserDto> result = service.list(PageRequest.of(page, size));

    var meta = new ApiResponse.Meta(
            "Listado de usuarios",
            Trace.currentId(), "v1",
            new ApiResponse.Pagination(
                    result.getNumber(),
                    result.getSize(),
                    result.getTotalElements(),
                    result.getTotalPages()
            )
    );
    return ResponseEntity.ok(ApiResponse.withMeta(result.getContent(), meta));
}
```

---

## 4) Contrato de **error** con RFC 7807 (Problem Details)

### 4.1 Principios
- Usa `ProblemDetail` (Spring Boot 3) para **todos** los errores.  
- Campos estándar: `type`, `title`, `status`, `detail`, `instance`.  
- Extiende con `code` (código interno), `errors` (validación), `traceId`.

### 4.2 Excepciones de dominio
```java
public class NotFoundException extends RuntimeException {
    private final String code;
    public NotFoundException(String message, String code) {
        super(message);
        this.code = code;
    }
    public String code() { return code; }
}

public class BusinessException extends RuntimeException {
    private final String code;
    public BusinessException(String message, String code) {
        super(message); this.code = code;
    }
    public String code() { return code; }
}
```

### 4.3 Manejador global (`@RestControllerAdvice`)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail handleNotFound(NotFoundException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setType(URI.create("/errors/not-found"));
        pd.setTitle("Recurso no encontrado");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", ex.code());
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setType(URI.create("/errors/business"));
        pd.setTitle("Regla de negocio incumplida");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", ex.code());
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("/errors/validation"));
        pd.setTitle("Error de validación");
        pd.setDetail("Uno o más campos son inválidos");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());

        List<Map<String, Object>> errors = ex.getBindingResult()
            .getFieldErrors().stream()
            .map(fe -> Map.of(
                "field", fe.getField(),
                "message", fe.getDefaultMessage(),
                "rejectedValue", fe.getRejectedValue()
            ))
            .toList();
        pd.setProperty("errors", errors);
        return pd;
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ProblemDetail handleUnreadable(HttpMessageNotReadableException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("/errors/bad-request"));
        pd.setTitle("Solicitud mal formada");
        pd.setDetail("El cuerpo de la solicitud no pudo parsearse");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setType(URI.create("/errors/internal"));
        pd.setTitle("Error interno");
        pd.setDetail("Ha ocurrido un error inesperado");
        pd.setProperty("traceId", Trace.currentId());
        pd.setProperty("instance", req.getRequestURI());
        // log.error("Unhandled", ex) con traceId
        return pd;
    }
}
```

### 4.4 Ejemplo de respuesta de error (JSON)
```json
{
  "type": "/errors/validation",
  "title": "Error de validación",
  "status": 400,
  "detail": "Uno o más campos son inválidos",
  "errors": [
    { "field": "email", "message": "must be a well-formed email address", "rejectedValue": "foo" }
  ],
  "traceId": "a3c4e0b1f2...",
  "instance": "/api/v1/users"
}
```

---

## 5) DTOs y validación (Bean Validation)

```java
public record CreateUserReq(
        @NotBlank String name,
        @Email String email,
        @Min(18) int age
) {}

public record UserDto(Long id, String name, String email, int age) {}
```

- La validación fallida dispara `MethodArgumentNotValidException` → la capturamos en el `@RestControllerAdvice`.  
- Mantén **mensajes** en *messages.properties* para i18n (opcional).

---

## 6) Trazabilidad (`traceId`) y consistencia

```java
public final class Trace {
    private Trace() {}
    public static String currentId() {
        return Optional.ofNullable(org.slf4j.MDC.get("traceId"))
                       .orElse(UUID.randomUUID().toString());
    }
}
```

**Buenas prácticas**
- Poner `traceId` en **response body** (meta) y/o **header** (`X-Trace-Id`).  
- Propagarlo a llamadas externas (HTTP, mensajería).  
- Loguear si cae en `@RestControllerAdvice`.

---

## 7) Versionado, contenido y *HATEOAS* (opcional)

- Prefija rutas: `/api/v1/...`.  
- `Content-Type`: usa `application/json`.  
- Links/HATEOAS si tu cliente lo necesita (no imprescindible si ya documentas con OpenAPI).  
- ETags para *reads* que requieran **caching** o **concurrencia optimista**.

---

## 8) Pruebas de contrato de respuesta

### 8.1 Éxito con `MockMvc`
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mvc;
    @MockBean UserService service;

    @Test
    void get_ok() throws Exception {
        when(service.get(1L)).thenReturn(new UserDto(1L,"Ana","ana@acme.com",25));

        mvc.perform(get("/api/v1/users/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.status").value("success"))
           .andExpect(jsonPath("$.data.id").value(1))
           .andExpect(jsonPath("$.meta.traceId").doesNotExist());
    }
}
```

### 8.2 Error de validación
```java
@Test
void create_validation_error() throws Exception {
    var body = """
      {"name": "", "email": "bad", "age": 10}
    """;

    mvc.perform(post("/api/v1/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(body))
       .andExpect(status().isBadRequest())
       .andExpect(jsonPath("$.type").value("/errors/validation"))
       .andExpect(jsonPath("$.errors").isArray());
}
```

---

## 9) Antipatrones y advertencias

- **Mezclar envelopes y ProblemDetail** en una misma respuesta: **no**.  
- Encapsular `ProblemDetail` dentro de un envelope de éxito: **no** (rompe RFC 7807).  
- Reutilizar `400` para **todo**: usa el código correcto (404/409/422…).  
- Ocultar `traceId`: dificulta el soporte.  
- Devolver `200` con "status":"error" en el body: **anti-REST**.  
- No incluir `Location` en `201 Created`: pierde valor semántico.

---

## 10) Checklist de adopción

- [ ] `ApiResponse<T>` implementado y usado en **todas las respuestas de éxito**.  
- [ ] `@RestControllerAdvice` con **ProblemDetail** para errores comunes (404, 400, 422, 409, 500).  
- [ ] **Status codes** correctos y `Location` en creaciones.  
- [ ] `traceId` en body/header y logs correlacionados.  
- [ ] Validación con *Bean Validation* (`@Valid` en endpoints).  
- [ ] Tests *MockMvc* para **éxito y error** (contratos).  
- [ ] Documentación OpenAPI actualizada (opcional pero recomendado).

---

## 11) Plantilla mínima (pom.xml) — fragmento

```xml
<dependencies>
  <!-- Web + Validación -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- Tests -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

> Si prefieres una lib dedicada para RFC 7807, puedes usar **Problem Spring Web**; si no, `ProblemDetail` es suficiente.

---

## 12) Ejercicios propuestos

1. Refactoriza un endpoint para devolver `201 Created` con `Location` y `ApiResponse.withMeta(...)`.  
2. Implementa `@RestControllerAdvice` y migra un error clásico a **ProblemDetail** (`/errors/not-found`).  
3. Agrega `traceId` a todas las respuestas y conéctalo al logger (MDC).  
4. Escribe pruebas *MockMvc* que validen el shape del JSON de éxito/error.  
5. Añade paginación a un GET y retorna `meta.page` correctamente poblado.