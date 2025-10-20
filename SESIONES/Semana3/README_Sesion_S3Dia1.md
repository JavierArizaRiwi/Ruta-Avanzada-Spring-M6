
# Día 1  3 – Persistencia con Spring Data JPA y MySQL  

En esta sesión aprenderás a **persistir datos de manera profesional** utilizando **Spring Data JPA** y **MySQL**. Comprenderás cómo integrar esta capa dentro de una arquitectura limpia o hexagonal sin violar la independencia del dominio.

---

## 1) ¿Qué es JPA y por qué usarlo?

**JPA (Java Persistence API)** es una especificación que define cómo mapear clases Java a tablas de bases de datos relacionales mediante ORM (Object Relational Mapping).  
Spring Data JPA implementa JPA y simplifica el acceso a los datos mediante **repositorios automáticos**.

### Ventajas sobre JDBC

| Aspecto | JDBC tradicional | JPA / Spring Data JPA |
|----------|------------------|-----------------------|
| Verbosidad | Alta (PreparedStatement, ResultSet) | Baja (solo interfaces) |
| Mantenimiento | Difícil | Fácil (repositorios autogenerados) |
| Consultas | SQL manual | JPQL / Query Methods |
| Transacciones | Manuales | Automáticas con `@Transactional` |
| Ciclo de vida | No gestionado | Gestionado por `EntityManager` |

---

## 2) Dependencias en `pom.xml`

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## 3) Configuración del DataSource (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/academico_db?useSSL=false&serverTimezone=UTC
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
        show_sql: true
```

---

## 4) Entidades con JPA  

```java
// domain/model/Estudiante.java
package com.riwi.academico.domain.model;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "estudiantes")
public class Estudiante {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 80)
    private String nombre;

    @OneToMany(mappedBy = "estudiante", cascade = CascadeType.ALL)
    private Set<Inscripcion> inscripciones = new HashSet<>();

    public Estudiante() {}
    public Estudiante(String nombre) { this.nombre = nombre; }

    public Long getId() { return id; }
    public String getNombre() { return nombre; }
}
```

```java
// domain/model/Curso.java
package com.riwi.academico.domain.model;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "cursos")
public class Curso {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;

    @OneToMany(mappedBy = "curso", cascade = CascadeType.ALL)
    private Set<Inscripcion> inscripciones = new HashSet<>();

    public Curso() {}
    public Curso(String nombre) { this.nombre = nombre; }
}
```

```java
// domain/model/Inscripcion.java
package com.riwi.academico.domain.model;

import jakarta.persistence.*;

@Entity
@Table(name = "inscripciones")
public class Inscripcion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne @JoinColumn(name = "id_estudiante")
    private Estudiante estudiante;

    @ManyToOne @JoinColumn(name = "id_curso")
    private Curso curso;

    public Inscripcion() {}
    public Inscripcion(Estudiante estudiante, Curso curso) {
        this.estudiante = estudiante;
        this.curso = curso;
    }
}
```

---

## 5) Repositorios orientados al dominio

```java
// infrastructure/jpa/repository/EstudianteJpaRepository.java
package com.riwi.academico.infrastructure.jpa.repository;

import com.riwi.academico.domain.model.Estudiante;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface EstudianteJpaRepository extends JpaRepository<Estudiante, Long> {
    List<Estudiante> findByNombreContainingIgnoreCase(String nombre);
}
```

**Consultas personalizadas**
```java
@Query("SELECT e FROM Estudiante e WHERE e.nombre LIKE %:nombre%")
List<Estudiante> buscarPorNombre(@Param("nombre") String nombre);
```

---

## 6) Integración con el dominio (Mapper)

Para mantener el dominio libre de dependencias JPA:

```java
// infrastructure/mapper/EstudianteMapper.java
package com.riwi.academico.infrastructure.mapper;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;

public class EstudianteMapper {
    public static Estudiante toDomain(EstudianteEntity entity){
        return new Estudiante(entity.getNombre());
    }

    public static EstudianteEntity toEntity(Estudiante model){
        EstudianteEntity e = new EstudianteEntity();
        e.setNombre(model.getNombre());
        return e;
    }
}
```

---

## 7) Pruebas con H2 y `@DataJpaTest`

```java
// test/java/com/riwi/acad/jpa/EstudianteRepositoryTest.java
package com.riwi.acad.jpa;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.infrastructure.jpa.repository.EstudianteJpaRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class EstudianteRepositoryTest {

    @Autowired
    private EstudianteJpaRepository repo;

    @Test
    void debeGuardarYRecuperarEstudiante() {
        Estudiante e = new Estudiante("María");
        repo.save(e);
        assertThat(repo.findByNombreContainingIgnoreCase("mar")).isNotEmpty();
    }
}
```

---

## 8) Configuración IntelliJ IDEA

1. **Plugins necesarios:**  
   Spring Boot, Database Tools, Lombok, SonarLint.

2. **Configurar conexión MySQL:**  
   - `View → Tool Windows → Database → + → Data Source → MySQL`
   - Verifica conexión `jdbc:mysql://localhost:3306/academico_db`

3. **Atajos útiles:**  
   - Buscar clases: `Ctrl+N` / `⌘O`  
   - Ejecutar test: `Ctrl+Shift+F10` / `⌘⇧R`  
   - Ver logs SQL: `Run → Edit Configurations → VM Options → -Dspring.jpa.show-sql=true`

4. **Annotation Processing:**  
   - `Settings → Build, Execution, Deployment → Compiler → Annotation Processors → Enable annotation processing`.

---

## 9) Beneficios del uso de JPA

| Componente | Propósito |
|-------------|------------|
| JPA | Simplifica la persistencia con ORM y reduce código repetido |
| Spring Data JPA | Genera repositorios automáticamente |
| H2 | Permite testear sin depender de MySQL |
| Mapper | Evita acoplar el dominio al framework |

---

## 10) Resultado esperado

- Base de datos configurada y sincronizada con las entidades JPA.  
- Entidades persistentes y relaciones funcionales.  
- Pruebas unitarias exitosas con `@DataJpaTest`.  
- Repositorios funcionales listos para integrarse a los puertos del dominio.  