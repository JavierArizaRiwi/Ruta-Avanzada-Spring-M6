# Guía Profesional: Arquitecturas Limpias, Hexagonales, Microservicios y DDD en Java con Spring

## Introducción

Las arquitecturas modernas en Java y Spring Boot permiten construir sistemas robustos, escalables y mantenibles. Esta guía cubre desde la evolución de las arquitecturas por capas y MVC, hasta Clean Architecture, Hexagonal, Microservicios y Domain-Driven Design (DDD), con teoría, diagramas, ejemplos y buenas prácticas.

---

## 1. Evolución: Capas y MVC

### Arquitectura por capas
Separación clásica en proyectos Java SE:
```
com.mycompany.academico
 ├─ domain/           → Entidades
 ├─ repository/       → Acceso a datos
 ├─ service/          → Lógica de negocio
 └─ ui/console        → Interfaz/controladores
```
**Ventajas:** claridad, reutilización, mantenimiento.
**Limitaciones:** dependencias rígidas, difícil de probar, poco escalable.

### MVC (Model-View-Controller)
Modelo: datos y entidades.
Vista: interfaz (HTML, JSP, Thymeleaf).
Controlador: recibe peticiones y delega en servicios.
En Spring Boot: `@Controller`, `@RestController`, `@Service`, `@Repository`, `@Entity`.

---


## 2. Arquitectura Limpia (Clean Architecture)

Propuesta por Robert C. Martin (Uncle Bob): independencia total entre capas. El dominio no depende de frameworks, bases de datos ni detalles externos.

### Estructura típica
```
com.company.proyecto
 ├─ domain/                → Entidades y lógica de negocio pura
 ├─ application/           → Casos de uso
 ├─ infrastructure/        → Adaptadores de frameworks, persistencia, APIs externas
 └─ entrypoints/           → Interfaces de entrada (REST, CLI, etc.)
```
**Flujo de dependencias:**
Entrypoint → Application → Domain ← Infrastructure
Las dependencias solo apuntan hacia adentro.

### Principios clave
- Independencia del framework y la UI.
- Independencia de la base de datos.
- Pruebas fáciles y aisladas.

---

------------------------------------------------------------------------


## 3. Arquitectura Hexagonal (Ports & Adapters)

Separar el núcleo de negocio de los detalles externos mediante puertos (interfaces) y adaptadores (implementaciones).

### Diagrama
```
┌───────────────────────────┐
│   Entradas (Driving)      │
│ REST, CLI, Schedulers     │
└─────────────┬─────────────┘
        │ llama casos de uso
 ┌───────▼────────┐
 │  Aplicación    │
 └───────┬────────┘
        │ usa puertos
 ┌───────▼────────┐
 │   Dominio      │
 └───────┬────────┘
        │ define puertos de salida
┌─────────────▼─────────────┐
│  Salidas (Driven)         │
│  JPA, Kafka, Email, etc.  │
└────────────────────────────┘
```

### Paquetes recomendados
```
com.company.proyecto
├─ domain
│  ├─ model/               # Entidades, Value Objects
│  ├─ service/             # Servicios de dominio
│  └─ spi/                 # Puertos de salida (interfaces)
├─ application
│  ├─ usecase/             # Casos de uso
│  └─ dto/                 # DTOs
├─ infrastructure
│  ├─ persistence/         # Adaptadores JPA/JDBC
│  ├─ messaging/           # Kafka/Rabbit
│  ├─ restclient/          # APIs externas
│  └─ config/              # Beans
└─ entrypoints
   ├─ rest/                # Controladores REST
   └─ scheduler/           # Jobs
```

### Ejemplo
**Dominio:**
```java
public class Estudiante {
    private final String id;
    private String nombre;
    // ... reglas de negocio
}
public interface EstudianteRepositoryPort {
    Estudiante save(Estudiante e);
    Optional<Estudiante> findById(String id);
    boolean existsByNombre(String nombre);
}
```
**Aplicación:**
```java
public class CrearEstudianteUseCase {
    private final EstudianteRepositoryPort repo;
    public CrearEstudianteUseCase(EstudianteRepositoryPort repo) { this.repo = repo; }
    public Estudiante execute(String id, String nombre) {
        // ... lógica
        return repo.save(new Estudiante(id, nombre));
    }
}
```
**Infraestructura:**
```java
@Repository
class JpaEstudianteRepositoryAdapter implements EstudianteRepositoryPort {
    // ... implementación JPA
}
```
**Entrypoint:**
```java
@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final CrearEstudianteUseCase crear;
    public EstudianteController(CrearEstudianteUseCase crear) { this.crear = crear; }
    @PostMapping
    public ResponseEntity<?> crear(@RequestParam String id, @RequestParam String nombre) {
        return ResponseEntity.ok(crear.execute(id, nombre));
    }
}
```

### Buenas prácticas
- El dominio no debe importar clases de Spring ni JPA.
- Define puertos en dominio, implementa en infraestructura.
- Casos de uso orquestan reglas y dependencias.
- Usa mappers para aislar modelos.
- Testing: pruebas puras en dominio, mocks en use cases, integración en adaptadores, @WebMvcTest en entrypoints.

---

---

## 2) Arquitectura de Microservicios

### 2.1. Principios
- **Servicios pequeños y autónomos**, desplegables de forma independiente.  
- **Base de datos por servicio** (evitar esquema compartido).  
- **Comunicación**: síncrona (REST/gRPC) o asíncrona (mensajería).  
- **Observabilidad**: logs centralizados, métricas, trazas distribuidas.  
- **Resiliencia**: timeouts, reintentos, circuito abierto, bulkheads.  
- **Automatización**: CI/CD, feature toggles, blue/green, canary.  

### 2.2. Topología típica
```
[Client] → [API Gateway] → [Service A] → [DB A]
                         → [Service B] → [DB B]
                         → [Service C] → [DB C]
             ↘  Message Broker (Kafka/Rabbit)  ↙
                (eventos, sagas, outbox)
```

### 2.3. Patrones clave
- **API Gateway**: entrada única, routing, throttling, auth, CORS.  
- **Service Discovery** (cuando aplica): registro/descubrimiento de instancias.  
- **Circuit Breaker/Retry/Rate Limit**: resiliencia en llamadas remotas.  
- **Strangler Fig**: migrar desde monolito a microservicios gradualmente.  
- **Saga** (orquestación o coreografía): consistencia distribuida.  
- **Outbox Pattern**: garantizar publicación de eventos junto con transacciones locales.  
- **CQRS/Event Sourcing** (cuando convenga): lectura/escritura separadas, histórico de eventos.

### 2.4. Datos y consistencia
- **Base de datos por servicio**: evita acoplamiento.  
- **Transacciones locales** en cada servicio; **eventual consistency** entre servicios.  
- **Idempotencia** en consumidores (reprocesos).  
- **Esquemas versionados** (Flyway/Liquibase por servicio).

### 2.5. Comunicación
- REST/gRPC para **query/command** directos.  
- Mensajería (Kafka/Rabbit) para **eventos del dominio** y **sagas**.  
- **Contrato**: usar **tests de contrato** (p.ej., Pact) para evitar rupturas.

### 2.6. Observabilidad y seguridad
- **Logs** estructurados (correlation-id, trace-id).  
- **Métricas** (Micrometer/Prometheus).  
- **Tracing** (OpenTelemetry).  
- **Seguridad**: OAuth2/OIDC, mTLS, secretos cifrados, política de mínimos privilegios.  

### 2.7. Testing en microservicios
- **Unit tests**: dominio y casos de uso.  
- **Slice tests**: @WebMvcTest, @DataJpaTest.  
- **Contract tests**: producer/consumer.  
- **Integration tests**: con Testcontainers para DB/brokers.  
- **E2E**: flujos entre servicios (idealmente en staging).

### 2.8. Ejemplo mínimo de servicio
Estructura del microservicio **catalogo**:
```
com.company.catalogo
├─ domain/
├─ application/
├─ infrastructure/
└─ entrypoints/
```
Cada servicio tiene su **propio** `pom.xml`, Dockerfile y pipeline CI. La comunicación con **ventas** o **inventario** sucede vía REST o eventos.

---

## 3) Domain-Driven Design (DDD)

### 3.1. Conceptos esenciales
- **Ubiquitous Language**: lenguaje compartido entre negocio y equipo técnico.  
- **Bounded Context**: límites claros del modelo; cada contexto tiene su propio Ubiquitous Language.  
- **Context Map**: relaciones entre contextos (Partnership, Customer/Supplier, Conformist, Anticorruption Layer).  
- **Entidades**: identidad estable.  
- **Value Objects**: inmutables, se comparan por valor.  
- **Agregados**: conjunto coherente de entidades/VO con invariantes y **Aggregate Root**.  
- **Repositorios**: colecciones persistentes de agregados.  
- **Servicios de Dominio**: lógica que no encaja en una entidad/VO.  
- **Eventos de Dominio**: hechos que ocurren en el dominio (“EstudianteCreado”).

### 3.2. Diseño tático (ejemplo)

**Value Object**
```java
// domain/model/Nombre.java
public class Nombre {
    private final String valor;
    public Nombre(String valor) {
        if (valor == null || valor.isBlank() || valor.length() < 3) throw new IllegalArgumentException("Nombre inválido");
        this.valor = valor;
    }
    public String valor() { return valor; }
}
```

**Agregado**
```java
// domain/model/Estudiante.java
public class Estudiante {
    private final String id;           // identidad
    private Nombre nombre;             // VO

    public Estudiante(String id, Nombre nombre) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
        this.id = id;
        this.nombre = nombre;
        // invariantes de agregado
    }

    public void renombrar(Nombre nuevo) { this.nombre = nuevo; }
    // getters (exponer VO o su primitivo con cuidado)
}
```

**Repositorio de agregados (puerto)**
```java
// domain/spi/EstudianteRepositoryPort.java
public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    Optional<Estudiante> porId(String id);
}
```

**Servicio de dominio**
```java
// domain/service/MatriculaService.java
public class MatriculaService {
    public void validarCupo(int cupo, int inscritos) {
        if (inscritos >= cupo) throw new IllegalStateException("Sin cupo para matrícula");
    }
}
```

**Caso de uso (aplicación) publicando evento**
```java
// application/usecase/CrearEstudiante.java
public class CrearEstudiante {
    private final EstudianteRepositoryPort repo;
    private final DomainEventPublisher eventPublisher;

    public CrearEstudiante(EstudianteRepositoryPort repo, DomainEventPublisher eventPublisher) {
        this.repo = repo; this.eventPublisher = eventPublisher;
    }

    public Estudiante ejecutar(String id, String nombre) {
        Estudiante e = new Estudiante(id, new Nombre(nombre));
        Estudiante guardado = repo.guardar(e);
        eventPublisher.publish(new EstudianteCreado(guardado.getId())); // evento de dominio
        return guardado;
    }
}
```

**Evento de dominio**
```java
// domain/event/EstudianteCreado.java
public record EstudianteCreado(String estudianteId) {}
```

**Publicador de eventos (puerto + adaptador)**
```java
// domain/spi/DomainEventPublisher.java
public interface DomainEventPublisher {
    void publish(Object domainEvent);
}

// infrastructure/messaging/KafkaDomainEventPublisher.java
@Component
class KafkaDomainEventPublisher implements DomainEventPublisher {
    private final KafkaTemplate<String, Object> kafka;
    public KafkaDomainEventPublisher(KafkaTemplate<String, Object> kafka) { this.kafka = kafka; }

    @Override public void publish(Object event) {
        kafka.send("dominio.estudiantes", event);
    }
}
```

### 3.3. Bounded Contexts y Context Map
- **Admissions**, **Academics**, **Billing** podrían ser contextos separados.  
- Define relaciones (ej.: Billing es **Conformist** respecto a Academics) y **ACL** (Anti-Corruption Layer) para integraciones con modelos distintos.

### 3.4. Transacciones y consistencia en DDD
- **Invariantes dentro del agregado** (transacción local).  
- Entre agregados/servicios: **eventual consistency** mediante eventos/sagas.  
- **Outbox** para publicar eventos de manera confiable.

---

## 4) Integrando todo con Spring Boot

### 4.1. Mapeo y persistencia
- Entidades de JPA **no son** tu agregado si necesitas invariantes estrictas; usa mappers.  
- `@Entity` en **infraestructura** (no en `domain`) y convierte a/desde objetos de dominio.  
- Repositorios Spring Data como **adaptadores** de puertos de salida.

### 4.2. Configuración de beans
- Crea **@Configuration** para instanciar casos de uso con sus puertos.  
- Evita `@Autowired` en campos; usa **inyección por constructor**.

### 4.3. Validación y DTOs
- Valida en **borde** (entrypoints) con Bean Validation (`@Valid`, `@NotBlank`, etc.).  
- Mantén DTOs de entrada/salida **fuera** del dominio.

### 4.4. Mensajería y transacciones
- Usa **Transacción local + Outbox**:  
  1) Guardas agregado y evento en tabla *outbox* en la misma TX.  
  2) Un *relay* publica el evento desde *outbox* a Kafka/Rabbit.

---

## 5) Patrones de diseño útiles
- **Factory** para crear agregados válidos.  
- **Specification** para reglas complejas de negocio.  
- **Strategy** para políticas variables.  
- **Decorator** para cross-cutting (p.ej., auditoría) en casos de uso.  
- **Anti-Corruption Layer** para acoplar contextos.

---

## 6) Estrategia de pruebas (pirámide)
1. **Dominio (rápidas, sin Spring)**: invariantes, VO, servicios de dominio.  
2. **Use cases (unit con mocks)**: comportamiento esperado usando puertos simulados.  
3. **Adaptadores (integración)**: JPA, mensajería, REST clients (Testcontainers).  
4. **Contratos**: Pact u otros para asegurar compatibilidad entre servicios.  
5. **E2E**: flujos completos, menor cantidad.

---

## 7) Checklist de implantación
- [ ] Dominio sin dependencias a Spring/JPA.  
- [ ] Puertos definidos en `domain.spi`.  
- [ ] Casos de uso en `application.usecase`.  
- [ ] Adaptadores en `infrastructure.*` con mappers.  
- [ ] Entradas en `entrypoints.*`.  
- [ ] Outbox + mensajería para consistencia entre servicios.  
- [ ] Observabilidad (tracing, métricas, logs) y resiliencia (timeouts, circuit breaker).  
- [ ] CI/CD por servicio y DB **por servicio**.  

---

## 8) Conclusión
- **Hexagonal** te da aislamiento del dominio respecto a infraestructura.  
- **DDD** te guía a modelar el negocio con foco en reglas, lenguaje ubicuo y límites claros.  
- **Microservicios** te permiten escalar equipos y despliegues, pero requieren disciplina en datos, observabilidad y resiliencia.  
Juntos, forman una base sólida para construir sistemas empresariales mantenibles y escalables con **Spring Boot**.

---

**Autor:** Javier Junior Ariza Montenegro  
**Versión:** 1.1 (detalle extendido)  
**Licencia:** MIT