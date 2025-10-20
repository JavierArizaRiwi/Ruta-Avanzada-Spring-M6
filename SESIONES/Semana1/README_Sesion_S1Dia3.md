
# Día 3 - Configuración de Beans, Perfiles y Proyecto Base Limpio en Spring

Esta guía te lleva, paso a paso y con ejemplos, desde una organización clásica en Java SE (capas, POO, JDBC) hacia un **proyecto base limpio** en Spring. Vas a dominar:
- Estrategias de configuración de beans (JavaConfig vs estereotipos).
- Estructura de paquetes alineada con arquitectura limpia.
- Perfiles y configuración externa por entorno (dev/test/prod).
- Propiedades, variables de entorno y secretos.
- Manejo de excepciones por capa y logging.
- Decisiones entre JPA y JDBC y su impacto arquitectónico.
- Configuración detallada en IntelliJ IDEA para un flujo de trabajo fluido.

---

## 1) De Java SE a Spring: ¿por qué reconfigurar el proyecto?
En Java SE solemos tener:
```
ui/console  →  service  →  dao (JDBC)  →  database
```
- Las dependencias se crean con `new` en cada clase.
- La configuración (URL de BD, credenciales) queda “dispersa” o embebida en código.
- Cambiar una implementación (JDBC → JPA) exige refactors amplios.

**Con Spring**, externalizamos configuración, inyectamos dependencias y movemos el wiring a una **capa de configuración**.
El dominio se mantiene **puro**, y los detalles (JPA/JDBC) viven en **adaptadores**.

---

## 2) Estructura de proyecto base (evolutiva a Clean/Hexagonal)

Recomendación inicial (monolito modular con capas limpias):

```
com.riwi.academico
 ├─ domain/                      # entidades/VO/servicios de dominio (sin Spring/JPA)
 │   ├─ model/
 │   ├─ service/
 │   └─ spi/                     # puertos (interfaces) que el dominio necesita
 ├─ application/                 # casos de uso (orquestación)
 │   └─ usecase/
 ├─ infrastructure/              # detalles técnicos (adaptadores)
 │   ├─ jpa/                     # persistencia con Spring Data JPA
 │   │   ├─ entity/
 │   │   ├─ repository/          # Spring Data repos
 │   │   └─ adapter/             # implementación de puertos
 │   ├─ jdbc/                    # alternativa: JDBC (si la comparas con JPA)
 │   ├─ messaging/               # Kafka/Email/etc. (más adelante)
 │   ├─ mapper/                  # conversión entidad JPA ↔ dominio
 │   └─ config/                  # @Configuration de beans y perfiles
 ├─ entrypoints/                 # interfaces de entrada
 │   └─ rest/                    # controladores/DTOs/validación
 └─ AcademicoApplication.java    # arranque (si usas Spring Boot)
```

**Regla de dependencias**: todas las dependencias apuntan hacia dentro (entrypoints → application → domain). `infrastructure` implementa los **puertos** definidos en `domain.spi`.

---

## 3) Configuración de Beans: JavaConfig vs Estereotipos

### 3.1 Dos estrategias complementarias
| Aspecto | `@Configuration` + `@Bean` | Estereotipos (`@Component`, `@Service`, `@Repository`) |
|---|---|---|
| Control del wiring | Explícito y visible | Conveniente/automático |
| Uso recomendado | Casos de uso y ensamblajes críticos | Adaptadores y servicios simples |
| Testabilidad | Alta (fácil reemplazo de beans) | Alta |
| Dependencias | Constructor injection | Constructor injection |

### 3.2 Ejemplo: wiring explícito de un caso de uso
```java
// domain/spi/EstudianteRepositoryPort.java (puerto)
package com.riwi.academico.domain.spi;

import com.riwi.academico.domain.model.Estudiante;
import java.util.Optional;

public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    Optional<Estudiante> porId(String id);
}
```

```java
// application/usecase/RegistrarEstudianteUseCase.java (orquestación)
package com.riwi.academico.application.usecase;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;

public class RegistrarEstudianteUseCase {
    private final EstudianteRepositoryPort repo;
    public RegistrarEstudianteUseCase(EstudianteRepositoryPort repo) { this.repo = repo; }

    public Estudiante ejecutar(String id, String nombre) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
        if (nombre == null || nombre.isBlank()) throw new IllegalArgumentException("nombre requerido");
        return repo.guardar(new Estudiante(id, nombre));
    }
}
```

```java
// infrastructure/jpa/adapter/EstudianteJpaAdapter.java (adaptador JPA)
package com.riwi.academico.infrastructure.jpa.adapter;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import com.riwi.academico.infrastructure.jpa.repository.EstudianteJpaRepository;
import com.riwi.academico.infrastructure.mapper.EstudianteMapper;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    private final EstudianteJpaRepository jpa;
    public EstudianteJpaAdapter(EstudianteJpaRepository jpa){ this.jpa = jpa; }

    @Override
    public Estudiante guardar(Estudiante e) {
        EstudianteEntity saved = jpa.save(EstudianteMapper.toEntity(e));
        return EstudianteMapper.toDomain(saved);
    }

    @Override
    public Optional<Estudiante> porId(String id) {
        return jpa.findById(id).map(EstudianteMapper::toDomain);
    }
}
```

```java
// infrastructure/config/UseCaseConfig.java (JavaConfig explícito)
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class UseCaseConfig {
    @Bean
    public RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo) {
        return new RegistrarEstudianteUseCase(repo);
    }
}
```

**Por qué así**: el caso de uso es parte de la **arquitectura** y conviene que el wiring sea explícito y visible; los adaptadores van con estereotipos.

---

## 4) Perfiles y configuración externa por entorno

### 4.1 Archivos por perfil
- `application.yml` base.
- `application-dev.yml`, `application-test.yml`, `application-prod.yml` para sobreescritura.

**Ejemplo `application.yml`:**
```yaml
spring:
  application:
    name: academico
logging:
  level:
    root: info
```

**Ejemplo `application-dev.yml`:**
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:acad;MODE=MySQL;DB_CLOSE_DELAY=-1
    username: sa
    password: 
  jpa:
    hibernate:
      ddl-auto: update
  profiles:
    group:
      local: dev
server:
  port: 8080
logging:
  level:
    org.hibernate.SQL: debug
```

**Ejemplo `application-prod.yml`:**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/academico
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate
server:
  port: 8080
```

### 4.2 Activar perfil
- VM option: `-Dspring.profiles.active=dev`
- Variable de entorno: `SPRING_PROFILES_ACTIVE=dev`

### 4.3 Secretos y variables
- Nunca codifiques credenciales.
- Usa variables de entorno o proveedores seguros (cuando escales: Vault/Config Server).

---

## 5) Manejo de excepciones y validaciones por capa

### 5.1 Dominio
- Crea excepciones significativas (por ejemplo, `NegocioException`).
- Lanza `IllegalArgumentException`/`NegocioException` al violar invariantes.

```java
package com.riwi.academico.domain.exception;
public class NegocioException extends RuntimeException {
    public NegocioException(String message){ super(message); }
}
```

### 5.2 Infraestructura
- Traduce errores de proveedor (SQL, HTTP) a excepciones propias de infraestructura.
- Evita filtrar detalles sensibles hacia arriba.

### 5.3 Entrypoints (REST)
- Manejo global con `@ControllerAdvice` para producir un **contrato uniforme de error**.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<?> handleIllegal(IllegalArgumentException ex){
        return ResponseEntity.badRequest().body(Map.of("message", ex.getMessage()));
    }
}
```

### 5.4 Validaciones
- En el borde (DTOs): `@Valid`, `@NotBlank`, `@Size`, etc.
- No mezclar validación de DTO con reglas de dominio (usa Value Objects cuando aplique).

---

## 6) Logging y trazabilidad

- SLF4J + Logback por defecto en Spring Boot.
- Estructura sugerida de log: `timestamp`, `nivel`, `servicio`, `traceId`, `mensaje`.
- En dev: elevar nivel de `org.hibernate.SQL` para ver queries.
- Añadir `traceId`/`correlationId` por request cuando expongas endpoints.

**Ejemplo `logback-spring.xml` (básico):**
```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

---

## 7) JPA vs JDBC: matriz de decisión

| Criterio | JDBC | JPA |
|---|---|---|
| Control fino de SQL | Alto | Medio (JPQL/Native) |
| Productividad CRUD | Media | Alta |
| Curva de aprendizaje | Baja/Media | Media/Alta |
| Portabilidad | Alta (estándar SQL) | Alta (JPA), detalles pueden variar |
| Rendimiento | Excelente si dominas SQL | Bueno; requiere tuning (fetch, batching) |

**Recomendación inicial**: usa JPA para la mayoría de CRUD; emplea JDBC/Native Query para consultas críticas y reporting.

---

## 8) Configuración detallada en IntelliJ IDEA

1. Abrir Settings/Preferences.  
2. Build Tools (Maven/Gradle): habilitar Auto-Import.  
3. Plugins: Spring, Lombok, SonarLint.  
4. Compiler → Annotation Processors: habilitar.  
5. Run/Debug: tipo Spring Boot/Application, VM options `-Dspring.profiles.active=dev`.  
6. Environment vars (si aplica): `DB_USER`, `DB_PASS`, `DB_URL`.  
7. Code Style/Inspections: activa inspecciones de Spring/Java.  
8. Atajos:  
   - Buscar clases: `Ctrl+N` / `⌘O`  
   - Buscar símbolos: `Ctrl+Alt+Shift+N` / `⌘⌥O`  
   - Buscar en todo: `Ctrl+Shift+F` / `⌘⇧F`  
   - Ir a declaración/uso: `Ctrl+B` / `⌘B`, `Alt+F7` / `⌥F7`

---

## 9) Ejemplo completo mínimo (unir piezas)

```java
// domain/model/Estudiante.java
package com.riwi.academico.domain.model;
public class Estudiante {
    private final String id;
    private String nombre;
    public Estudiante(String id, String nombre){
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
        if (nombre == null || nombre.isBlank()) throw new IllegalArgumentException("nombre requerido");
        this.id = id; this.nombre = nombre;
    }
    public String getId(){ return id; }
    public String getNombre(){ return nombre; }
    public void renombrar(String nuevo){ if(nuevo==null||nuevo.isBlank()) throw new IllegalArgumentException("nombre requerido"); this.nombre = nuevo; }
}
```

```java
// infrastructure/jpa/entity/EstudianteEntity.java
package com.riwi.academico.infrastructure.jpa.entity;

import jakarta.persistence.*;
@Entity @Table(name="estudiantes")
public class EstudianteEntity {
    @Id private String id;
    @Column(nullable=false) private String nombre;
    public EstudianteEntity() {}
    public EstudianteEntity(String id, String nombre){ this.id=id; this.nombre=nombre; }
    public String getId(){ return id; } public String getNombre(){ return nombre; }
}
```

```java
// infrastructure/jpa/repository/EstudianteJpaRepository.java
package com.riwi.academico.infrastructure.jpa.repository;

import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import org.springframework.data.jpa.repository.JpaRepository;
public interface EstudianteJpaRepository extends JpaRepository<EstudianteEntity, String> {}
```

```java
// infrastructure/mapper/EstudianteMapper.java
package com.riwi.academico.infrastructure.mapper;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;

public class EstudianteMapper {
    public static EstudianteEntity toEntity(Estudiante d){ return new EstudianteEntity(d.getId(), d.getNombre()); }
    public static Estudiante toDomain(EstudianteEntity e){ return new Estudiante(e.getId(), e.getNombre()); }
}
```

```java
// entrypoints/rest/EstudianteController.java
package com.riwi.academico.entrypoints.rest;

import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.model.Estudiante;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final RegistrarEstudianteUseCase registrar;
    public EstudianteController(RegistrarEstudianteUseCase registrar){ this.registrar = registrar; }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestParam String id, @RequestParam String nombre){
        return ResponseEntity.ok(registrar.ejecutar(id, nombre));
    }
}
```

```java
// infrastructure/config/UseCaseConfig.java (ya mostrado)
```

---

## 10) Problemas comunes y soluciones

| Problema | Causa probable | Solución |
|---|---|---|
| `NoSuchBeanDefinitionException` | Falta `@Component`/`@Repository`/`@Configuration` o paquete fuera de `@ComponentScan` | Revisa paquetes y anotaciones |
| Conflicto de beans del mismo tipo | Varias implementaciones | Usa `@Qualifier` o `@Primary` |
| `Failed to configure a DataSource` | Datos de conexión inválidos | Verifica `application-*.yml` y variables de entorno |
| Errores de validación de JPA | Entidades mal mapeadas | Revisa `@Id`, `@Column`, nombres de tabla |
| `Circular dependency` | A ↔ B se inyectan mutuamente | Introduce puertos o refactoriza responsabilidades |

---

## 11) Checklist del día
- Estructura base creada con `domain/application/infrastructure/entrypoints`.
- Casos de uso inyectados con **JavaConfig** (`@Bean`).  
- Perfiles configurados: `dev`, `test`, `prod`.  
- Propiedades externas y secretos sin hardcode.  
- Manejo de excepciones por capa y logging configurado.  
- Decisiones JPA/JDBC documentadas.

---

## 12) Próximos pasos
A partir de este proyecto base, en la próxima sesión pasarás a **Spring Boot**, autoconfiguración, y a reforzar la **arquitectura limpia** con pruebas por capas y documentación de dependencias.