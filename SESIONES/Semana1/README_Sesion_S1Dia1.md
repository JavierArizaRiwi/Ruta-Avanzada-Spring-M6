# Día 1 - De Java SE a Spring: IoC, DI, Beans, Estereotipos y Ciclo de Vida

## Introducción general

Este primer día marca la transición desde la **programación estructurada en Java SE** hacia el **ecosistema Spring**, uno de los frameworks empresariales más potentes del mundo Java.  
Comprenderás **por qué nació Spring**, cómo soluciona los problemas de las aplicaciones monolíticas clásicas y cómo introduce los conceptos de **Inversión de Control (IoC)** y **Inyección de Dependencias (DI)** para lograr sistemas más modulares, mantenibles y escalables.

---

## 1. Origen y propósito de Spring Framework

### 1.1 Problemas del enfoque Java EE clásico
Antes de Spring, el desarrollo empresarial usaba **EJB (Enterprise JavaBeans)**, con mucha complejidad, XML y acoplamiento fuerte.

Problemas típicos:
- Configuraciones extensas y repetitivas.  
- Dificultad para probar componentes.  
- Dependencias rígidas entre clases.  
- Uso excesivo de recursos (instancias, transacciones).

### 1.2 Solución propuesta por Spring
Spring nació en 2003 con un objetivo claro:
> “Simplificar el desarrollo empresarial en Java mediante la gestión automática de dependencias y configuraciones.”

Principios fundamentales:
- **Inversión de Control (IoC)**: el contenedor crea e inyecta objetos.  
- **Programación orientada a interfaces**.  
- **Desacoplamiento entre capas y frameworks**.  
- **Modularidad y reutilización.**

---

## 2. Arquitectura por capas tradicional en Java SE

Un proyecto típico se estructuraba así:

```
UI/Consola → Servicio → DAO → Base de datos (JDBC)
```

Ejemplo clásico en Java SE:

```java
public class EstudianteService {
    private EstudianteRepository repo = new EstudianteRepository(); // acoplado
}
```

Desventajas:
- Acoplamiento fuerte (`new` en cada clase).  
- Dificultad para cambiar implementaciones (de JDBC a JPA).  
- Imposible testear aisladamente sin la base de datos.

---

## 3. Inversión de Control (IoC): concepto clave

### 3.1 Definición
**Inversión de Control (IoC)** significa **entregar el control de creación y administración de objetos a un contenedor**.  
En lugar de que tu código cree dependencias, el framework lo hace por ti.

**Antes (control directo):**
```java
NotificadorEmail notificador = new NotificadorEmail();
AlertaService alerta = new AlertaService(notificador);
```

**Con Spring (control invertido):**
```java
@Service
public class AlertaService {
    private final NotificadorEmail notificador;
    public AlertaService(NotificadorEmail notificador) {
        this.notificador = notificador;
    }
}
```
El contenedor **crea** e **inyecta** `NotificadorEmail` cuando detecta que es un Bean.

### 3.2 ApplicationContext y BeanFactory

- **BeanFactory**: versión básica del contenedor (gestiona creación de beans).  
- **ApplicationContext**: versión extendida (maneja eventos, perfiles, i18n, etc.).

Ejemplo de uso manual:
```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
AlertaService alerta = context.getBean(AlertaService.class);
alerta.enviarAlerta();
```

---

## 4. Inyección de Dependencias (DI)

### 4.1 Concepto
**Dependency Injection** es el mecanismo mediante el cual el contenedor IoC **inyecta automáticamente** los objetos (dependencias) que una clase necesita.

### 4.2 Formas de inyección

| Tipo | Descripción | Recomendación |
|------|--------------|---------------|
| Constructor Injection | Dependencia obligatoria | ✅ Mejor práctica |
| Setter Injection | Dependencia opcional | Usar solo si aplica |
| Field Injection | Inyección directa en atributos | Evitar (dificulta testeo) |

Ejemplo con inyección por constructor:
```java
@Service
public class EstudianteService {
    private final EstudianteRepository repo;
    public EstudianteService(EstudianteRepository repo) { this.repo = repo; }
}
```

---

## 5. Qué es un Bean

Un **Bean** es un **objeto gestionado por el contenedor Spring**, creado a partir de anotaciones o configuraciones explícitas.  
Spring administra su **instanciación, ciclo de vida y dependencia**.

### 5.1 Definición de Beans

Tres formas principales:

1. **Anotaciones de estereotipo**
```java
@Component
public class NotificadorEmail { }
```
2. **Métodos `@Bean` dentro de una clase `@Configuration`**
```java
@Configuration
public class AppConfig {
    @Bean
    public NotificadorEmail notificadorEmail() {
        return new NotificadorEmail();
    }
}
```
3. **Escaneo automático (`@ComponentScan`)**
```java
@SpringBootApplication
public class AcademicoApplication { }
```

---

## 6. Ciclo de vida de un Bean

1. **Definición** → anotaciones o configuración.  
2. **Instanciación** → el contenedor crea el objeto.  
3. **Inyección** → se resuelven las dependencias.  
4. **Inicialización** → se ejecuta `@PostConstruct` (si existe).  
5. **Uso** → el bean opera dentro de la aplicación.  
6. **Destrucción** → al cerrar el contexto, se llama a `@PreDestroy`.

Ejemplo:
```java
@Component
public class ConexionDB {

    @PostConstruct
    void iniciar() {
        System.out.println("Conexión inicializada...");
    }

    @PreDestroy
    void cerrar() {
        System.out.println("Conexión cerrada.");
    }
}
```

---

## 7. Estereotipos y sus responsabilidades

| Anotación | Capa | Rol | Descripción |
|------------|------|-----|--------------|
| `@Component` | Infraestructura | General | Marca un bean genérico |
| `@Service` | Aplicación | Negocio | Define lógica de servicio o caso de uso |
| `@Repository` | Persistencia | Datos | Traduce excepciones SQL a `DataAccessException` |
| `@Controller` | Presentación | Web MVC | Controlador de vistas |
| `@RestController` | Presentación | API REST | `@Controller` + `@ResponseBody` |

Ejemplo completo:
```java
@Repository
public class EstudianteRepositoryJdbc implements EstudianteRepository { }

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

## 8. Scopes (alcance de los Beans)

| Scope | Descripción | Contexto |
|--------|--------------|----------|
| `singleton` | Única instancia por contenedor | Por defecto |
| `prototype` | Nueva instancia en cada inyección | General |
| `request` | Una instancia por solicitud HTTP | Web |
| `session` | Una por sesión de usuario | Web |

Ejemplo:
```java
@Scope("prototype")
@Component
public class ReporteTemporal { }
```

---

## 9. Configuración inicial en IntelliJ IDEA

1. **Crear proyecto:**  
   - `File → New → Project → Spring Initializr`  
   - Dependencias: *Spring Context*, *Spring Web*, *Spring Data JPA*, *H2 Database*, *Lombok*.

2. **Estructurar paquetes:**
   ```
   com.riwi.academico
    ├─ domain/
    ├─ application/
    ├─ infrastructure/
    ├─ web/
    └─ config/
   ```

3. **Archivo principal:**
   ```java
   @SpringBootApplication
   public class AcademicoApplication {
       public static void main(String[] args) {
           SpringApplication.run(AcademicoApplication.class, args);
       }
   }
   ```

4. **Ver Beans registrados:**
   ```java
   @Bean
   CommandLineRunner runner(ApplicationContext ctx) {
       return args -> Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
   }
   ```

5. **Ejecutar aplicación:**
   - `Shift + F10` o clic derecho → Run.  
   - Observa la creación automática de Beans en consola.

---

## 10. Ejemplo completo: de Java SE a Spring

**Antes (Java SE):**
```java
public class App {
    public static void main(String[] args) {
        EstudianteRepository repo = new EstudianteRepositoryJdbc();
        EstudianteService service = new EstudianteService(repo);
        service.registrar("Ana");
    }
}
```

**Después (Spring):**
```java
@SpringBootApplication
public class AcademicoApp {
    public static void main(String[] args) {
        SpringApplication.run(AcademicoApp.class, args);
    }
}
```

Spring crea automáticamente los Beans `EstudianteRepository` y `EstudianteService`, y los conecta.

---

## 11. Diagnóstico de errores comunes

| Error | Causa | Solución |
|-------|--------|-----------|
| `NoSuchBeanDefinitionException` | Falta anotación o escaneo | Añadir `@Component` o ajustar `@ComponentScan` |
| `NullPointerException` en bean | Inyección por campo | Usar constructor injection |
| Ciclo de dependencias | Clases A ↔ B se referencian mutuamente | Introducir interfaz o refactorizar |
| Bean duplicado | Dos definiciones con mismo tipo | Usar `@Qualifier` o `@Primary` |

---

## 12. Reto práctico

1. Crea un proyecto Spring simple.  
2. Implementa:
   - `@Repository` EstudianteRepository (lista en memoria).  
   - `@Service` EstudianteService con métodos CRUD.  
   - `@RestController` EstudianteController con endpoints `/api/estudiantes`.  
3. Ejecuta y revisa los Beans en consola.  
4. Analiza cómo el contenedor IoC se encarga de todo el wiring.

---

**Resultado esperado:**  
Comprender el funcionamiento del contenedor de Spring, el ciclo de vida de los beans y cómo IoC/DI reemplazan la creación manual de objetos, estableciendo la base para arquitecturas limpias y desacopladas.