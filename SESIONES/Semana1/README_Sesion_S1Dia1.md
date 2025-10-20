# Día 1 — De Java SE a Spring: IoC, DI, Beans, Estereotipos y Ciclo de Vida

> Objetivo: entender **qué es Spring** y por qué su **contenedor IoC** cambia la forma de estructurar proyectos. Conecta POO, patrones (Factory, Strategy, DAO/Repository) y arquitectura por capas con **Inversión de Control (IoC)** e **Inyección de Dependencias (DI)**, y conoce a fondo los **estereotipos** y el **ciclo de vida de los beans** dentro de un proyecto en IntelliJ IDEA.

---

## 1) Punto de partida: ¿cómo lo hacíamos en Java SE?

### 1.1 Arquitectura por capas clásica
```
UI/Controller  →  Service  →  DAO/Repository  →  JDBC/MySQL
         validación   reglas      SQL/Mapeo        conexiones
```
- Dependencias creadas con `new` dentro de servicios o controladores.  
- Configuración dispersa (URLs, credenciales) y acoplamiento a implementación concreta.  
- Dificultad para testear en aislamiento.

### 1.2 Dolencias que aparecían
- Duplicación de creación de objetos.  
- Cambiar de JDBC a JPA = refactor doloroso.  
- Mocks complicados, pruebas frágiles.

---

## 2) Inversión de Control (IoC) y el contenedor de Spring

**Definición:** delegar en un contenedor la **creación**, **configuración** y **ciclo de vida** de los objetos (beans).  
**Valor:** el código “declara” dependencias; el contenedor las **inyecta**.

### 2.1 Bean y contexto
- **Bean:** objeto gestionado por Spring.  
- **ApplicationContext:** fábrica avanzada de beans + servicios infra (i18n, eventos, perfiles).

### 2.2 Ciclo de vida de un bean (visión)
1. Definición (`@Component`, `@Service`, `@Repository` o `@Bean`).  
2. Instanciación del objeto.  
3. Inyección de dependencias.  
4. Ejecución de post‑procesadores (AOP, validaciones, proxies).  
5. Inicialización (`@PostConstruct` si aplica).  
6. Uso activo del bean.  
7. Destrucción (`@PreDestroy`).

---

## 3) Inyección de Dependencias (DI): formas y buenas prácticas

### 3.1 Tipos
- **Constructor Injection** (preferida): dependencias obligatorias → inmutables (`final`).  
- **Setter Injection:** dependencias opcionales.  
- **Field Injection:** evitar (dificulta test y claridad).

### 3.2 Ejemplo comparativo

**Java SE (acoplado):**
```java
class EstudianteService {
  private final EstudianteRepository repo = new EstudianteRepositoryJdbc(); // acoplamiento
}
```

**Spring (DI por constructor):**
```java
@Service
public class EstudianteService {
  private final EstudianteRepository repo;
  public EstudianteService(EstudianteRepository repo) { this.repo = repo; }
}
```

**JavaConfig explícito (equivale a Factory controlada):**
```java
@Configuration
public class AppConfig {
  @Bean EstudianteRepository estudianteRepository() { return new EstudianteRepositoryJdbc(); }
  @Bean EstudianteService estudianteService(EstudianteRepository repo) { return new EstudianteService(repo); }
}
```

---

## 4) Conexión con POO y patrones de diseño

| Concepto | Cómo se traduce en Spring |
|---|---|
| **Factory / Abstract Factory** | Métodos `@Bean` crean instancias bajo control del contenedor |
| **Strategy** | Inyectar múltiples implementaciones (mapa de estrategias) sin `if/switch` |
| **DAO/Repository** | Contratos (interfaces) desacoplados de la tecnología (JDBC/JPA) |
| **Singleton** | Scope por defecto de los beans (gestionado correctamente por Spring) |

---

## 5) Estereotipos: cómo se categorizan los beans

Los **estereotipos** son anotaciones que indican a Spring *qué tipo de responsabilidad tiene una clase* y habilitan el **descubrimiento automático** (Component Scan).

| Estereotipo | Uso típico | Capa | Valor agregado |
|--------------|------------|------|----------------|
| `@Component` | Componente genérico reutilizable | Infraestructura o dominio | Marca para escaneo general |
| `@Service` | Lógica de negocio, casos de uso | Application | Semántica: "servicio" |
| `@Repository` | Acceso a datos (DAO/JPA/JDBC) | Infraestructura | Traduce excepciones SQL a `DataAccessException` |
| `@Controller` | Controladores web (MVC) | Presentación | Devuelve vistas (Thymeleaf, JSP) |
| `@RestController` | API REST | Presentación (Web/API) | `@Controller` + `@ResponseBody` |

### Ejemplo práctico
```java
@Repository
public class EstudianteRepositoryJdbc implements EstudianteRepository {
    // Acceso a la base de datos con JDBC
}

@Service
public class EstudianteService {
    private final EstudianteRepository repo;
    public EstudianteService(EstudianteRepository repo) { this.repo = repo; }
}

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final EstudianteService service;
    public EstudianteController(EstudianteService service) { this.service = service; }

    @GetMapping
    public List<Estudiante> listar() { return service.listar(); }
}
```

---

## 6) Component Scan y organización del proyecto

```java
@SpringBootApplication
@ComponentScan(basePackages = "com.riwi.academico")
public class AcademicoApp {
  public static void main(String[] args) {
    SpringApplication.run(AcademicoApp.class, args);
  }
}
```

Spring explorará todos los subpaquetes de `com.riwi.academico` y registrará los beans anotados.

**Estructura recomendada:**
```
com.riwi.academico
 ├─ domain/           # Entidades y reglas (POO)
 ├─ application/      # Casos de uso y servicios
 ├─ infrastructure/   # Adaptadores, persistencia, APIs externas
 ├─ web/              # Controladores REST o MVC
 └─ config/           # Beans, perfiles y seguridad
```

---

## 7) Alcance (Scope) de los Beans

| Scope | Descripción | Contexto |
|--------|-------------|----------|
| `singleton` | Uno por contenedor (default) | General |
| `prototype` | Nueva instancia por inyección | General |
| `request` | Nueva instancia por solicitud HTTP | Web |
| `session` | Una por sesión HTTP | Web |

Ejemplo:
```java
@Scope("prototype")
@Component
public class ReporteTemporal { ... }
```

---

## 8) Ciclo de vida y hooks (`@PostConstruct` / `@PreDestroy`)

```java
@Component
public class ConectorDB {

  @PostConstruct
  void init() {
    System.out.println("Conexión inicializada...");
  }

  @PreDestroy
  void close() {
    System.out.println("Conexión cerrada...");
  }
}
```
**En IntelliJ IDEA:** al ejecutar la app desde la configuración de Spring Boot, verás los mensajes en la consola.

---

## 9) IntelliJ IDEA: configuración del proyecto paso a paso

1. **Crea el proyecto:**  
   - File → New → Project → *Spring Initializr* (si usarás Boot) o *Spring* (si es puro).
   - Selecciona dependencias: *Spring Context*, *Spring Web*, *Spring JDBC*, *Spring Data JPA*.
   - GroupId: `com.riwi`; ArtifactId: `academico`.

2. **Estructura de paquetes:**  
   IntelliJ te creará el paquete raíz según el GroupId. Añade subpaquetes `domain`, `application`, `infrastructure`, `web`, `config`.

3. **Configura el `application.yml`:**
   - Ubicación: `src/main/resources/application.yml`
   - Ejemplo:
     ```yaml
     spring:
       datasource:
         url: jdbc:h2:mem:acad
         username: sa
         driver-class-name: org.h2.Driver
       jpa:
         hibernate:
           ddl-auto: update
     ```

4. **Ejecuta desde IntelliJ:**  
   - Abre el archivo principal (`AcademicoApp.java`).  
   - Clic derecho → *Run AcademicoApp* o usa Shift + F10.  
   - Verás en consola los logs de creación de beans (`Creating shared instance of bean...`).

5. **Ver los beans registrados:**  
   Usa en consola de IntelliJ:
   ```bash
   mvn spring-boot:run
   ```
   Y revisa los logs del ApplicationContext o añade un comando dentro de tu clase principal:
   ```java
   @Bean
   CommandLineRunner runner(ApplicationContext ctx) {
     return args -> {
       System.out.println("Beans cargados:");
       Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
     };
   }
   ```

---

## 10) Diagnóstico y resolución de problemas comunes

| Síntoma | Causa posible | Solución |
|---|---|---|
| `NoSuchBeanDefinitionException` | Bean no escaneado o conflicto de tipos | Verifica `@ComponentScan`, `@Qualifier` |
| Ciclo de dependencias | A ↔ B con dependencia mutua | Introduce interfaz/puerto o refactoriza responsabilidades |
| `NullPointer` en bean | Inyección por campo sin proxying o no inicializado | Usa **constructor injection** |
| Rendimiento al iniciar | Escaneo excesivo | Restringe paquetes y evita beans innecesarios |

---

## 11) Resumen visual (mapa mental)

```
POO + Patrones  →  Interfaces (contratos)  →  Beans  →  IoC Container
                                ↑                ↓
                        Implementaciones    Inyección (DI)
```

**Clave:** el dominio depende de **abstracciones**, y Spring se encarga del wiring.  
Con esto se habilita SOLID (DIP/OCP) y arquitecturas limpias.

---

## 12) Mini‑reto para cerrar el día

- Refactoriza tu CRUD JDBC del proyecto anterior:
  1. Crea interfaces `EstudianteRepository` y `CursoRepository`.
  2. Implementa las clases `EstudianteRepositoryJdbc` con `@Repository`.
  3. Crea un `EstudianteService` anotado con `@Service`.
  4. Expón un `EstudianteController` anotado con `@RestController`.
  5. Ejecuta el proyecto en IntelliJ y revisa los logs de inicialización de beans.