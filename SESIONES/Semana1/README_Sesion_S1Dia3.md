# Día 3 - Configuración de Beans, Perfiles y Proyecto Base Limpio en Spring

Esta guía te lleva, paso a paso, desde una organización clásica en **Java SE (capas, POO, JDBC)** hacia un **proyecto base limpio y modular** con **Spring Framework**.  
El objetivo es comprender **qué es un Bean**, cómo se configuran, qué son los **perfiles por entorno**, y cómo estructurar un proyecto preparado para crecer hacia arquitecturas limpias.

---

## Contexto educativo

Los estudiantes vienen de trabajar con **arquitectura por capas tradicional**, donde el flujo era:

```
UI / Consola → Servicio → DAO → Base de datos
```

Ahora aprenderán a trasladar esos conceptos al ecosistema Spring, entendiendo cómo **los Beans reemplazan las instancias manuales**, cómo los **perfiles facilitan entornos**, y cómo **configurar un proyecto base** con separación clara de responsabilidades.

---

## 1. ¿Qué es un Bean y por qué es importante?

En Java puro, los objetos se crean manualmente con `new`:

```java
UsuarioService service = new UsuarioService();
```

En Spring, los objetos que forman parte de la aplicación son **Beans**, creados y gestionados por el **contenedor IoC (Inversión de Control)**.  
Este contenedor se encarga de su ciclo de vida: instanciación, inyección de dependencias, inicialización y destrucción.

### Definición formal

Un **Bean** es **cualquier objeto administrado por el contenedor de Spring**.  
Spring se encarga de crearlo, configurarlo y ponerlo a disposición de otras clases que lo necesiten.

---

## 2. Cómo se define un Bean en Spring

Existen tres formas principales de definir Beans.

### 1. Usando anotaciones de estereotipo

Estas anotaciones identifican clases como componentes de Spring:

| Anotación | Propósito | Capa típica |
|------------|------------|-------------|
| `@Component` | Marca un componente genérico | Utilidades o helpers |
| `@Service` | Indica un servicio o caso de uso | Capa de negocio o aplicación |
| `@Repository` | Indica una clase que accede a la base de datos | Capa de persistencia |
| `@Controller` | Controlador web MVC | Capa de presentación |

Ejemplo:

```java
@Component
public class NotificadorEmail {
    public void enviar(String mensaje) {
        System.out.println("Enviando email: " + mensaje);
    }
}

@Service
public class AlertaService {
    private final NotificadorEmail notificador;

    public AlertaService(NotificadorEmail notificador) {
        this.notificador = notificador;
    }

    public void enviarAlerta() {
        notificador.enviar("Nueva alerta generada");
    }
}
```

Spring detecta automáticamente ambas clases, las registra como Beans y se encarga de la inyección.

---

### 2. Usando `@Configuration` + `@Bean`

Permite definir Beans manualmente con control total sobre su creación.  
Ideal para servicios complejos o configuración de infraestructura.

```java
@Configuration
public class ConfiguracionAplicacion {

    @Bean
    public NotificadorEmail notificadorEmail() {
        return new NotificadorEmail();
    }

    @Bean
    public AlertaService alertaService(NotificadorEmail notificador) {
        return new AlertaService(notificador);
    }
}
```

Esto es equivalente a usar `new`, pero dentro del contexto Spring y con gestión completa del ciclo de vida.

---

### 3. Escaneo automático de componentes

Cuando la clase principal tiene `@SpringBootApplication`, se activa el **component scan**, que busca Beans en los subpaquetes.

```java
@SpringBootApplication
public class AcademicoApplication {
    public static void main(String[] args) {
        SpringApplication.run(AcademicoApplication.class, args);
    }
}
```

Por defecto, escaneará todo el paquete `com.riwi.academico` y sus subpaquetes.

---

## 3. Inversión de Control e Inyección de Dependencias

### Antes (Java SE)

```java
ServicioNotificacion servicio = new ServicioNotificacion();
Controlador controlador = new Controlador(servicio);
```

### Ahora (Spring IoC)

Spring crea los objetos y los inyecta automáticamente.

```java
@RestController
public class Controlador {
    private final ServicioNotificacion servicio;

    public Controlador(ServicioNotificacion servicio) {
        this.servicio = servicio;
    }
}
```

Spring detecta que `ServicioNotificacion` es un Bean (`@Service`) y lo inyecta sin necesidad de `new`.

Ventaja: el código está desacoplado y las dependencias se gestionan desde el contenedor.

---

## 4. Estructura base del proyecto (Clean-Ready)

```
com.riwi.academico
 ├─ domain/                  # Entidades, reglas de negocio, interfaces (puertos)
 │   ├─ model/
 │   ├─ service/
 │   └─ spi/
 ├─ application/             # Casos de uso (servicios de aplicación)
 │   └─ usecase/
 ├─ infrastructure/          # Adaptadores (JPA, JDBC, APIs externas)
 │   ├─ jpa/
 │   ├─ jdbc/
 │   ├─ mapper/
 │   └─ config/
 ├─ entrypoints/             # Controladores REST o CLI
 │   └─ rest/
 └─ AcademicoApplication.java
```

En esta estructura:
- **domain:** no conoce Spring ni frameworks externos.  
- **application:** coordina los casos de uso.  
- **infrastructure:** implementa detalles técnicos.  
- **entrypoints:** expone endpoints (controladores).

---

## 5. Configuración de Beans: cuándo usar cada enfoque

| Enfoque | Cuándo usarlo | Ventajas |
|----------|----------------|-----------|
| `@Configuration` + `@Bean` | Beans de infraestructura o librerías externas | Control explícito sobre instanciación |
| Estereotipos (`@Component`, `@Service`, `@Repository`) | Clases de negocio o aplicación | Sencillez y autodescubrimiento |

Ejemplo combinado:

```java
@Configuration
public class UseCaseConfig {
    @Bean
    public RegistrarEstudianteUseCase registrarEstudianteUseCase(EstudianteRepositoryPort repo) {
        return new RegistrarEstudianteUseCase(repo);
    }
}

@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    // Implementación CRUD
}
```

---

## 6. Perfiles y configuración externa

Los perfiles permiten tener propiedades distintas según el entorno (desarrollo, pruebas, producción).

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
    url: jdbc:h2:mem:academico
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

Activación del perfil:

Terminal:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

IntelliJ IDEA:

```
Run → Edit Configurations → Environment Variables → SPRING_PROFILES_ACTIVE=dev
```

---

## 7. Manejo de excepciones por capa

Cada capa debe manejar sus errores de forma coherente.

**Dominio**

```java
public class NegocioException extends RuntimeException {
    public NegocioException(String mensaje) { super(mensaje); }
}
```

**Infraestructura**

```java
@Repository
public class EstudianteJpaAdapter {
    public Estudiante guardar(Estudiante e) {
        try {
            // persistencia
        } catch (DataAccessException ex) {
            throw new InfraestructuraException("Error al acceder a la base de datos", ex);
        }
    }
}
```

**Presentación**

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

## 8. Logging en Spring

Spring Boot usa SLF4J + Logback por defecto.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class EstudianteService {
    private static final Logger log = LoggerFactory.getLogger(EstudianteService.class);

    public void registrar(Estudiante e) {
        log.info("Registrando estudiante: {}", e.getNombre());
    }
}
```

Configuración personalizada (`logback-spring.xml`):

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

## 9. JPA vs JDBC

| Característica | JDBC | JPA |
|----------------|------|-----|
| Control SQL | Total | Abstracto |
| Productividad | Media | Alta |
| Transacciones | Manuales (`conn.commit()`) | Automáticas (`@Transactional`) |
| Ideal para | Consultas personalizadas | CRUD y dominio rico |

**Ejemplo JDBC**

```java
String sql = "INSERT INTO estudiante (id, nombre) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, e.getId());
    ps.setString(2, e.getNombre());
    ps.executeUpdate();
}
```

**Ejemplo JPA**

```java
@Repository
public interface EstudianteJpaRepository extends JpaRepository<EstudianteEntity, String> {}
```

---

## 10. Configuración en IntelliJ IDEA

1. Crear proyecto con Spring Initializr.  
2. Dependencias: Spring Web, Spring Data JPA, H2 o MySQL, Lombok.  
3. Activar perfil: `SPRING_PROFILES_ACTIVE=dev`.  
4. Habilitar Annotation Processing.  
5. Ver Beans cargados:

```java
@Bean
CommandLineRunner runner(ApplicationContext ctx) {
    return args -> Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
}
```

Atajos:  
- Buscar clase: `Ctrl+N`  
- Buscar método: `Ctrl+Shift+Alt+N`  
- Ver jerarquía: `Ctrl+H`  
- Ir a definición: `Ctrl+B`

---

## 11. Checklist final

- [x] Beans configurados correctamente  
- [x] Perfiles por entorno activos  
- [x] Excepciones por capa gestionadas  
- [x] Logging funcional  
- [x] Proyecto modular y listo para escalar  

---

**Resultado esperado:**  
Un proyecto Spring modular, mantenible y limpio, con Beans bien configurados, perfiles activos, excepciones controladas y una arquitectura base lista para evolucionar hacia Clean Architecture o Hexagonal.