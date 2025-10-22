# Complementos productivos para Spring Boot  
## Lombok a fondo y librerías adicionales para reducir *boilerplate* y mejorar tu DX

Esta guía complementa tu base de Spring con un foco práctico en **Project Lombok** y un conjunto de librerías gratuitas que aceleran el desarrollo profesional: mapeo de DTOs, manejo de errores HTTP, resiliencia, pruebas de integración, validaciones y migraciones de base de datos. Todo con ejemplos, advertencias y *best practices*.

---

## 0) Objetivos

- Dominar **Lombok**: instalación, configuración de IDE, anotaciones clave y patrones recomendados/evitados.  
- Integrar librerías que **eliminan código repetitivo**: **MapStruct**, **Problem Spring Web**, **Resilience4j/Spring Retry**, **Testcontainers**, **ArchUnit**, **AssertJ**, **Flyway/Liquibase**.  
- Comprender compatibilidades entre Lombok ↔ Jackson ↔ MapStruct ↔ JPA.  
- Salir con una **plantilla** y **checklist** para proyectos Spring Boot.

---

## 1) Lombok — qué es, instalación y anatomía

**Qué es:** Lombok es un procesador de anotaciones que **genera código en compilación** (getters, setters, constructores, *builders*, `equals/hashCode`, `toString`, *logs*, etc.), reduciendo *boilerplate* y mejorando la legibilidad.

### 1.1 Dependencia y configuración

**Maven**  
```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.18.32</version>
  <scope>provided</scope>
</dependency>
```

> `scope: provided` es habitual porque Lombok actúa en compilación. Puedes omitirlo si tu pipeline lo requiere.

**IDE (muy importante)**  
- IntelliJ IDEA / Eclipse / VS Code: habilita *annotation processing*.  
  - IntelliJ: *Settings → Build → Compiler → Annotation Processors → Enable*.  
- Instala el *plugin de Lombok* (IDEA/Eclipse) para compatibilidad total (sombreado de métodos generados, navegación).

**Verificación**  
Compila con:
```bash
mvn -q -DskipTests package
```
Si falla por “método no existe” o similar en getters/setters, revisa el *annotation processing*.

---

## 2) Anotaciones Lombok esenciales (con ejemplos y cuándo usarlas)

> Reglas generales:  
> - Prefiere **inyección por constructor** en Spring (`@RequiredArgsConstructor`).  
> - Evita `@Data` en **entidades JPA** (explicamos por qué más abajo).  
> - Usa `@Builder` para DTOs/inmutables y **`@Jacksonized`** si vas a deserializar con Jackson.

### 2.1 Accesores y valor semántico

```java
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import lombok.EqualsAndHashCode;

@Getter @Setter
@ToString
@EqualsAndHashCode
public class Estudiante {
  private Long id;
  private String nombre;
}
```

- `@Getter/@Setter`: generan accesores por campo o a nivel de clase.  
- `@ToString`: útil para logging (evita imprimir campos sensibles).  
- `@EqualsAndHashCode`: define igualdad por valor; especifica `onlyExplicitlyIncluded` si necesitas control fino.

```java
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
class Curso {
  @EqualsAndHashCode.Include private String codigo;
  private String nombre;
}
```

### 2.2 *Data classes*, inmutabilidad y *builders*

```java
import lombok.Data;

@Data // getter/setter, equals/hashCode, toString, requiredArgsConstructor
public class EstudianteDTO {
  private Long id;
  private String nombre;
}
```

**Cuándo `@Data`:** DTOs o *POJOs* simples sin *lazy proxies* ni ciclos.  
**Evita `@Data` en JPA**: `equals/hashCode/toString` pueden tocar colecciones *lazy* o claves naturales y romper el proxy; usa alternativas (ver 2.5).

**Inmutables con `@Value`**  
```java
import lombok.Value;

@Value
public class Punto {
  int x;
  int y;
}
```

**Constructores y *builder***  
```java
import lombok.Builder;
import lombok.With;

@Builder
public class PedidoDTO {
  private Long id;
  private String producto;
  @Builder.Default private int cantidad = 1;
}

var p = PedidoDTO.builder().producto("Laptop").build();
var p2 = p.toBuilder().cantidad(3).build(); // si usas @SuperBuilder en jerarquías
```

- `@Builder.Default` asigna valores por defecto cuando construyes vía *builder*.  
- `@With` genera métodos *copy-with* inmutables.

**Compatibilidad Jackson**  
```java
import lombok.Builder;
import lombok.extern.jackson.Jacksonized;

@Jacksonized
@Builder
public class CrearPedidoRequest {
  private String producto;
  private int cantidad;
}
```
`@Jacksonized` habilita *deserialización* con Jackson para *builders* de Lombok.

### 2.3 Constructores e inyección de dependencias

```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class PedidoService {
  private final PedidoRepository repo; // inyectado por constructor generado
}
```

- `@RequiredArgsConstructor` genera constructor para campos `final` o `@NonNull`.  
- En controladores/servicios Spring evita `@AllArgsConstructor` cuando haya campos no obligatorios.

### 2.4 Logging

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class PagoService {
  public void procesar() {
    log.info("Procesando pago");
  }
}
```

Atajos: `@Log` (java.util), `@CommonsLog`, `@Log4j2`, `@Slf4j` (recomendado: SLF4J).

### 2.5 Lombok y JPA (patrones seguros)

Problemas comunes con `@Data` en entidades:
- `equals/hashCode` que cargan relaciones *lazy* o usan campos mutables.  
- `toString` que recorre gráfos y produce *StackOverflow*.

**Recomendado:**
```java
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import lombok.EqualsAndHashCode;
import lombok.NoArgsConstructor;
import lombok.AccessLevel;

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED) // requerido por JPA
@ToString(onlyExplicitlyIncluded = true)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
class Usuario {
  @Id @GeneratedValue
  @EqualsAndHashCode.Include
  private Long id;

  @ToString.Include
  private String username;

  // relaciones con @ToString.Exclude y cuidado con @EqualsAndHashCode
  // @ManyToOne(fetch = FetchType.LAZY) @ToString.Exclude
}
```

**Builders con JPA:** utiliza `@Builder` en DTOs o *aggregate roots* con cuidado; para entidades JPA típicamente mantén **constructor protegido por defecto** + *setters* controlados.

### 2.6 Utilidades varias

- `@NonNull`: genera *null-check* en parámetros/campos.  
- `@SneakyThrows`: evita *checked exceptions* (útil pero úsalo con mesura).  
- `@Cleanup`: cierra recursos *try-with-resources like*.  
- `@UtilityClass`: clase estática pura (convierte métodos a `static`).

---

## 3) MapStruct — mapeo DTO ↔ dominio sin reflexión

**Qué es:** generador de mappers en compilación (rápido y *type‑safe*, sin reflexión).

**Maven**  
```xml
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.5.5.Final</version>
</dependency>
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct-processor</artifactId>
  <version>1.5.5.Final</version>
  <scope>provided</scope>
</dependency>
```

**Anotador Spring + Lombok:**  
```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok-mapstruct-binding</artifactId>
  <version>0.2.0</version>
  <scope>provided</scope>
</dependency>
```

**Ejemplo**
```java
// DTO
@lombok.Data
public class EstudianteResponse { Long id; String nombre; }

// Entidad/domain
@lombok.Getter @lombok.Setter
public class Estudiante { Long id; String nombre; }

// Mapper
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface EstudianteMapper {
  EstudianteResponse toResponse(Estudiante e);

  @InheritInverseConfiguration
  Estudiante toDomain(EstudianteResponse dto);

  @AfterMapping
  default void normalize(@MappingTarget EstudianteResponse dto){
    if (dto.getNombre() != null) dto.setNombre(dto.getNombre().trim());
  }
}
```

**Ventajas vs ModelMapper:**  
- MapStruct genera código en *compile-time* (rápido, *type-safe*).  
- ModelMapper usa reflexión (más simple al inicio, más lento y menos seguro).

---

## 4) Problem Spring Web — errores HTTP RFC 7807 limpios

Estandariza respuestas de error (`application/problem+json`).

**Maven**
```xml
<dependency>
  <groupId>org.zalando</groupId>
  <artifactId>problem-spring-web-starter</artifactId>
  <version>0.29.1</version>
</dependency>
```

**Uso simple**
```java
import org.zalando.problem.Problem;
import org.zalando.problem.Status;

throw Problem.valueOf(Status.NOT_FOUND, "Estudiante no existe");
```

**ControllerAdvice listo**
```java
import org.zalando.problem.spring.web.advice.ProblemHandling;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
class GlobalProblemHandler implements ProblemHandling {}
```

Resultado estándar:
```json
{
  "type":"about:blank",
  "title":"Not Found",
  "status":404,
  "detail":"Estudiante no existe"
}
```

---

## 5) Resiliencia: Resilience4j y Spring Retry

### 5.1 Resilience4j (circuit breaker, retry, bulkhead, rate limiter)
**Maven**
```xml
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
  <version>2.2.0</version>
</dependency>
```

**Ejemplo**
```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;

@Service
class ClientePago {

  @CircuitBreaker(name = "pagos", fallbackMethod = "fallback")
  @Retry(name = "pagos")
  public PagoResponse cobrar(...) { /* llamada remota */ }

  private PagoResponse fallback(Exception ex){ return PagoResponse.error("No disponible"); }
}
```

**application.yml**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      pagos:
        sliding-window-size: 10
        failure-rate-threshold: 50
  retry:
    instances:
      pagos:
        max-attempts: 3
        wait-duration: 500ms
```

### 5.2 Spring Retry (alternativa ligera)
```xml
<dependency>
  <groupId>org.springframework.retry</groupId>
  <artifactId>spring-retry</artifactId>
</dependency>
```
```java
@EnableRetry
@Configuration
class RetryConfig {}

@Service
class ClientePagoLigero {
  @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 500))
  public String invocar() { ... }
  @Recover String recover(Exception ex){ return "fallback"; }
}
```

---

## 6) Testcontainers — pruebas de integración realistas

**Qué es:** *spin* de contenedores Docker para tests (PostgreSQL, Kafka, Redis…).

**Maven**
```xml
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>
  <version>1.20.1</version>
  <scope>test</scope>
</dependency>
```

**Ejemplo**
```java
@Testcontainers
class RepoIT {

  @Container
  static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", db::getJdbcUrl);
    r.add("spring.datasource.username", db::getUsername);
    r.add("spring.datasource.password", db::getPassword);
  }

  @Autowired PedidoJpaRepository repo;

  @Test void guardaYLee(){ ... }
}
```

---

## 7) Calidad de código en tiempo de compilación

### 7.1 ArchUnit — reglas de arquitectura
```xml
<dependency>
  <groupId>com.tngtech.archunit</groupId>
  <artifactId>archunit-junit5</artifactId>
  <version>1.2.1</version>
  <scope>test</scope>
</dependency>
```
```java
@AnalyzeClasses(packages = "com.riwi")
class ArquitecturaTest {
  @Test
  void controladoresNoDependenDeInfraestructura() {
    noClasses().that().resideInAPackage("..entrypoints..")
      .should().dependOnClassesThat().resideInAnyPackage("..infrastructure..")
      .check(importedClasses());
  }
}
```

### 7.2 AssertJ — aserciones fluidas
```xml
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <version>3.25.3</version>
  <scope>test</scope>
</dependency>
```
```java
assertThat(lista).hasSize(2).extracting("nombre").contains("Ana");
```

---

## 8) Validación y migraciones de esquema

- **Hibernate Validator** (incluido con `spring-boot-starter-validation`): `@NotBlank`, `@Email`, `@Size`…  
- **Flyway** o **Liquibase** para migraciones *versionadas*.

**Flyway (recomendado por simplicidad)**
```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
```
`src/main/resources/db/migration/V1__init.sql`  
```sql
create table estudiante (id bigint primary key, nombre varchar(50) not null);
```

---

## 9) Plantilla mínima de *pom.xml* (fragmento)

```xml
<dependencies>
  <!-- Spring Boot básicos -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- Lombok -->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.32</version>
    <scope>provided</scope>
  </dependency>

  <!-- MapStruct -->
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
  </dependency>
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.5.5.Final</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok-mapstruct-binding</artifactId>
    <version>0.2.0</version>
    <scope>provided</scope>
  </dependency>

  <!-- Problem Spring Web -->
  <dependency>
    <groupId>org.zalando</groupId>
    <artifactId>problem-spring-web-starter</artifactId>
    <version>0.29.1</version>
  </dependency>

  <!-- Resilience4j -->
  <dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
  </dependency>

  <!-- Testcontainers + AssertJ -->
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.20.1</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
  </dependency>

  <!-- Flyway -->
  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>
</dependencies>

<build>
  <plugins>
    <!-- Necesario para procesadores de anotaciones (MapStruct, Lombok) -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
      <configuration>
        <source>17</source>
        <target>17</target>
        <annotationProcessorPaths>
          <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.32</version>
          </path>
          <path>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>1.5.5.Final</version>
          </path>
          <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok-mapstruct-binding</artifactId>
            <version>0.2.0</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

---

## 10) Recetas frecuentes (copiar/pegar)

### 10.1 DTO con Lombok + Builder + Jackson
```java
import lombok.Builder;
import lombok.extern.jackson.Jacksonized;

@Jacksonized
@Builder
public record CrearUsuarioRequest(String username, String email) {}
```

### 10.2 Entidad JPA segura con Lombok
```java
@Entity @Table(name="usuarios")
@Getter @Setter @NoArgsConstructor(access = AccessLevel.PROTECTED)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@ToString(onlyExplicitlyIncluded = true)
class Usuario {
  @Id @GeneratedValue
  @EqualsAndHashCode.Include
  private Long id;

  @Column(nullable=false, unique=true)
  @ToString.Include
  private String username;
}
```

### 10.3 Mapper con MapStruct y Spring
```java
@Mapper(componentModel = "spring")
public interface UsuarioMapper {
  UsuarioDto toDto(Usuario u);
  Usuario toEntity(UsuarioDto d);
}
```

### 10.4 Error RFC7807 con Problem
```java
throw Problem.valueOf(Status.CONFLICT, "Nombre duplicado");
```

### 10.5 Circuit Breaker con Resilience4j
```java
@CircuitBreaker(name="servicio-x", fallbackMethod="fb")
public Respuesta invocar(){ ... }
public Respuesta fb(Throwable t){ return Respuesta.error("No disponible"); }
```

### 10.6 Test de integración con Testcontainers (Postgres)
```java
@Testcontainers
class UsuarioRepoIT {
  @Container static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16-alpine");
  @DynamicPropertySource static void cfg(DynamicPropertyRegistry r){
    r.add("spring.datasource.url", db::getJdbcUrl);
    r.add("spring.datasource.username", db::getUsername);
    r.add("spring.datasource.password", db::getPassword);
  }
}
```

---

## 11) Antipatrones y advertencias

- `@Data` en entidades JPA: puede romper `equals/hashCode`/`toString` con *lazy*. Usa `@Getter/@Setter` + inclusiones explícitas.  
- `@SneakyThrows`: úsalo con criterio; puede ocultar errores de diseño.  
- *Builders* en entidades con invariantes complejas: mejor *factory methods* o validaciones explícitas.  
- Mezclar ModelMapper y MapStruct: define un estándar; evita doble mantenimiento.  
- Olvidar *annotation processing* en IDE/CI: causa errores “fantasma”.  
- No fijar versiones de procesadores de anotaciones: bloqueos raros en CI. Usa versiones explícitas.

---

## 12) Checklist de adopción

- Lombok instalado y *annotation processing* activo.  
- DTOs con `@Builder` + `@Jacksonized` cuando apliquen.  
- Entidades JPA con `@Getter/@Setter` + constructores controlados.  
- Mappers con **MapStruct** (`componentModel = spring`).  
- Errores estandarizados con **Problem Spring Web**.  
- Resiliencia con **Resilience4j** (o **Spring Retry**).  
- Pruebas de integración con **Testcontainers**.  
- Migraciones con **Flyway**.  
- Reglas de arquitectura con **ArchUnit**.  
- Aserciones con **AssertJ**.

---

## 13) Ejercicios propuestos

1. Refactoriza tus DTOs a `record` + `@Builder` + `@Jacksonized`.  
2. Reemplaza ModelMapper por **MapStruct** en un módulo y mide tiempos.  
3. Estandariza errores con **Problem Spring Web** y elimina tu `GlobalExceptionHandler` casero.  
4. Añade un *circuit breaker* con **Resilience4j** a una llamada HTTP y provoca fallos.  
5. Crea un test de integración con **Testcontainers** sobre tu repositorio JPA y ejecuta una migración **Flyway** previa.

---

### Referencias rápidas
- Lombok Annotations: https://projectlombok.org/features/all  
- MapStruct Reference: https://mapstruct.org/documentation/stable/reference  
- Problem Spring Web: https://github.com/zalando/problem-spring-web  
- Resilience4j: https://resilience4j.readme.io  
- Testcontainers: https://www.testcontainers.org  
- Flyway: https://flywaydb.org