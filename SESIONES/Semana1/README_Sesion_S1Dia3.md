# Día 3 — Configuración de Beans, Perfiles y Proyecto Base Limpio en Spring

Esta guía te lleva, paso a paso, desde una organización clásica en Java SE (capas, POO, JDBC) hacia un **proyecto base limpio** en Spring Framework.  
Aquí consolidarás conocimientos sobre el ecosistema de Spring: entenderás **qué es un Bean**, cómo se configuran, cómo funcionan los **perfiles por entorno**, y cómo crear una estructura modular que siente las bases para arquitecturas limpias y escalables.

---

## 1) ¿Qué es un Bean y por qué es importante?

En Java SE, tú creas los objetos manualmente con `new`:
```java
UsuarioService service = new UsuarioService();
```
En Spring, los objetos que forman parte de tu aplicación son **Beans**, y son **creados, configurados y gestionados** por el **contenedor IoC (Inversión de Control)**.  
El contenedor se encarga de su ciclo de vida: instanciación, inyección de dependencias, inicialización y destrucción.

### Definición formal
Un **Bean** es **cualquier objeto administrado por el contenedor de Spring**.  
Puede definirse de tres formas:

1. Mediante anotaciones de estereotipo (`@Component`, `@Service`, `@Repository`, `@Controller`).
2. Mediante métodos anotados con `@Bean` dentro de una clase `@Configuration`.
3. Automáticamente a través del **escaneo de componentes** (`@ComponentScan`).

### Ejemplo simple:
```java
@Component
public class NotificadorEmail {
    public void enviar(String mensaje) {
        System.out.println("Enviando email: " + mensaje);
    }
}
```

Spring detecta este componente al iniciar el contexto y lo convierte en un Bean disponible para inyección en otras clases.

```java
@Service
public class AlertaService {
    private final NotificadorEmail notificador;
    public AlertaService(NotificadorEmail notificador) { this.notificador = notificador; }
}
```

**Conclusión:** ya no necesitas usar `new`, Spring se encarga del wiring.

---

## 2) De Java SE a Spring: una nueva estructura de responsabilidades

En Java SE, la arquitectura suele verse así:
```
ui/console  →  service  →  dao (JDBC)  →  database
```
El problema: las dependencias se crean a mano, y el código está acoplado.

**Con Spring y arquitecturas limpias:**
```
entrypoints → application → domain ← infrastructure
```
- Las dependencias se inyectan.
- El dominio es puro (sin dependencias de frameworks).
- Las configuraciones viven en `infrastructure/config`.

---

## 3) Estructura base del proyecto (Clean-Ready)

```
com.riwi.academico
 ├─ domain/                      # Entidades, reglas de negocio, interfaces (puertos)
 │   ├─ model/
 │   ├─ service/
 │   └─ spi/
 ├─ application/                 # Casos de uso (orquestación, lógica de aplicación)
 │   └─ usecase/
 ├─ infrastructure/              # Adaptadores (JPA, JDBC, Messaging, APIs externas)
 │   ├─ jpa/
 │   │   ├─ entity/
 │   │   ├─ repository/
 │   │   └─ adapter/
 │   ├─ jdbc/
 │   ├─ mapper/
 │   └─ config/
 ├─ entrypoints/                 # Interfaces de entrada (REST, CLI, WebSocket)
 │   └─ rest/
 └─ AcademicoApplication.java
```

**Dependencias:**  
- Los módulos externos dependen de los internos.  
- El dominio no conoce a Spring ni a la base de datos.

---

## 4) Configuración de Beans: `@Configuration` vs Estereotipos

| Enfoque | Ventajas | Cuándo usarlo |
|----------|-----------|----------------|
| `@Configuration` + `@Bean` | Wiring explícito, control total | Casos de uso críticos o configuraciones de infraestructura |
| Estereotipos (`@Component`, `@Service`, `@Repository`) | Simplicidad, menos código | Servicios, adaptadores y controladores |

### Ejemplo práctico:
```java
@Configuration
public class UseCaseConfig {
    @Bean
    public RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo) {
        return new RegistrarEstudianteUseCase(repo);
    }
}
```
Esto define un **Bean manualmente** usando **JavaConfig**.

Mientras que los adaptadores pueden declararse así:
```java
@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    // implementación
}
```

Spring detecta automáticamente los Beans por **component scanning** cuando tu clase principal tiene `@SpringBootApplication`.

---

## 5) Perfiles y configuración externa

### Archivos YAML por entorno

**application.yml**
```yaml
spring:
  application:
    name: academico
  profiles:
    active: dev
```

**application-dev.yml**
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:acad
    username: sa
  jpa:
    hibernate:
      ddl-auto: update
logging:
  level:
    org.hibernate.SQL: debug
```

**application-prod.yml**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/academico
    username: ${DB_USER}
    password: ${DB_PASS}
```

### Activar un perfil
- Desde IntelliJ: `Run > Edit Configurations > Environment Variables` → `SPRING_PROFILES_ACTIVE=dev`
- Desde terminal:  
  ```bash
  mvn spring-boot:run -Dspring-boot.run.profiles=prod
  ```

**Importante:** usa variables de entorno para credenciales y secretos.  
Ejemplo en Linux:
```bash
export DB_USER=root
export DB_PASS=12345
```

---

## 6) Manejo de excepciones por capa

### Dominio
```java
public class NegocioException extends RuntimeException {
    public NegocioException(String mensaje) { super(mensaje); }
}
```

### Infraestructura
```java
@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    public Estudiante guardar(Estudiante e) {
        try {
            // persistencia
        } catch (DataAccessException ex) {
            throw new InfraestructuraException("Error en la base de datos", ex);
        }
    }
}
```

### Presentación (REST)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NegocioException.class)
    public ResponseEntity<?> handleNegocio(NegocioException ex) {
        return ResponseEntity.badRequest().body(Map.of("error", ex.getMessage()));
    }
}
```

---

## 7) Logging en Spring

Spring Boot usa **SLF4J** + **Logback** por defecto.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class EstudianteService {
    private static final Logger log = LoggerFactory.getLogger(EstudianteService.class);

    public void registrar(Estudiante e) {
        log.info("Registrando estudiante {}", e.getNombre());
    }
}
```

Configura el formato del log en `logback-spring.xml`:
```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} %-5level [%thread] %logger - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

---

## 8) JPA vs JDBC: comparativa práctica

| Aspecto | JDBC | JPA |
|----------|------|-----|
| Control SQL | Total | Abstracto |
| Productividad | Media | Alta |
| Transacciones | Manuales (`conn.commit()`) | Automáticas (`@Transactional`) |
| Ideal para | Consultas específicas | CRUD, dominio rico |

Ejemplo **JDBC**:
```java
String sql = "INSERT INTO estudiante (id, nombre) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, e.getId());
    ps.setString(2, e.getNombre());
    ps.executeUpdate();
}
```

Ejemplo **JPA**:
```java
@Repository
public interface EstudianteJpaRepository extends JpaRepository<EstudianteEntity, String> {}
```

---

## 9) Configuración en IntelliJ IDEA

1. **Crear el proyecto:** File → New → Project → *Spring Initializr*.
2. **Seleccionar dependencias:** Spring Web, Spring Data JPA, H2 o MySQL Driver, Lombok.
3. **Configurar perfiles:**  
   - Run → Edit Configurations → Environment Variables → `SPRING_PROFILES_ACTIVE=dev`
4. **Activar procesadores de anotaciones:**  
   - Settings → Build → Compiler → Annotation Processors → “Enable annotation processing”.
5. **Ver los beans cargados:**  
   ```java
   @Bean
   CommandLineRunner runner(ApplicationContext ctx) {
       return args -> Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
   }
   ```

Atajos útiles:  
- Buscar clase: `Ctrl+N`  
- Buscar método: `Ctrl+Shift+Alt+N`  
- Navegar a declaración: `Ctrl+B`  
- Ver jerarquía: `Ctrl+H`

---

## 10) Checklist final

- [x] Beans definidos y detectados correctamente.  
- [x] Perfiles y propiedades por entorno funcionando.  
- [x] Manejo de excepciones por capa implementado.  
- [x] Logging configurado.  
- [x] Proyecto base estructurado con arquitectura limpia.

---

**Resultado esperado:**  
Un proyecto Spring modular, mantenible y escalable, con Beans bien configurados, perfiles activos y separación clara entre dominio, aplicación, infraestructura y presentación.