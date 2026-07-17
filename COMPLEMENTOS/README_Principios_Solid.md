
# Guía detallada — De POO en Java SE a Spring + Principios SOLID

> **Estado curricular:** vigente como referencia de Semanas 1 y 5. SOLID se evalúa con reglas del caso conductor, no por cantidad de interfaces o patrones.

> Objetivo: construir una **línea de transición** clara desde los principios de **Programación Orientada a Objetos (POO) en Java SE** hacia **Spring** y **SOLID**, con ejemplos prácticos, anti‑patrones comunes y ejercicios. Material pensado para coders que ya hicieron proyectos por capas/MVC en Java SE y ahora migran a **Spring Boot**.

---

## 0) Mapa mental rápido

- **POO**: Abstracción, Encapsulamiento, Herencia, Polimorfismo (pilares).  
- **Problema**: proyectos crecen, aparece alto acoplamiento, duplicación y fragilidad.  
- **Spring**: Inversión de Control (IoC), Inyección de Dependencias (DI), contenedor de beans, AOP.  
- **SOLID**: cinco principios para diseño mantenible: SRP, OCP, LSP, ISP, DIP.  
- **Resultado**: código **modular** y **probable** en unidades, con dependencias explícitas y bajo acoplamiento.

---

## 1) POO en Java SE — recap y trampas frecuentes

### 1.1 Pilares de la POO
- **Abstracción**: modelar lo esencial (interfaces, clases abstractas).  
- **Encapsulamiento**: ocultar detalles internos (modificadores, getters/setters con invariantes).  
- **Herencia**: reutilizar comportamiento (cuidado: puede aumentar el acoplamiento).  
- **Polimorfismo**: cambiar comportamiento por interfaz/jerarquía.

### 1.2 Trampas que aparecen en Java SE
- **Dios/Manager Class**: una clase hace “de todo” (violación SRP).  
- **Dependencias ocultas**: crear objetos “new” dispersos en la lógica (dificulta pruebas).  
- **Herencia excesiva**: cuando composición sería más clara.  
- **Servicios estáticos**: facilitan el uso, pero hacen difícil el testeo/mocks.  
- **Acoplamiento duro a implementaciones concretas**: imposible intercambiar dependencias.

---

## 2) De Java SE a Spring — IoC, DI y el contenedor

**Idea central**: **tú** no creas las dependencias a mano con `new`. En su lugar, declaras **qué** necesitas y el **contenedor de Spring** se encarga de **inyectarlo**.

### 2.1 Conceptos clave
- **Bean**: objeto gestionado por Spring (ciclo de vida controlado).  
- **IoC (Inversión de Control)**: el contenedor invoca y arma dependencias.  
- **DI (Inyección de Dependencias)**: recibir colaboraciones por **constructor** (preferido), setter o campo.  
- **Contexto**: `ApplicationContext` que conoce y provee beans.

### 2.2 Cómo se ve en código
```java
// Java SE (acoplamiento duro)
PedidoService service = new PedidoService(new PedidoRepository());

// Spring (DI por constructor, test-friendly)
@Service
public class PedidoService {
    private final PedidoRepository repo;
    public PedidoService(PedidoRepository repo) { this.repo = repo; }
}
```
```java
// Configuración explícita (opcional, preferida en Clean/Hexagonal)
@Configuration
public class BeanConfig {
    @Bean
    public PedidoService pedidoService(PedidoRepository repo) {
        return new PedidoService(repo);
    }
}
```

**Beneficio**: pruebas más sencillas, sustitución de implementaciones, wiring coherente.

---

## 3) Principios SOLID — teoría, olfatos de código y ejemplos Spring

### 3.1 SRP — **Single Responsibility Principle**
**Cada clase/módulo debe tener una única razón para cambiar.**

**Olor**: una clase escribe logs, valida, llama a la DB y formatea respuestas.  
**Refactor**:
- Separa **validación**, **persistencia**, **formateo** en colaboraciones.
- Usa **servicios de dominio** y **adaptadores** en infraestructura.

**Ejemplo (Spring)**
```java
// Anti‑ejemplo (todo en uno)
@Service
public class FacturacionService {
  // calcula totales, guarda en DB, arma PDF, envía email...
}

// Solución SRP
@Service class CalculoFacturaService { /* reglas */ }
@Repository interface FacturaRepository extends JpaRepository<Factura, Long> {}
@Component class GeneradorPdf { /* solo PDF */ }
@Component class NotificadorEmail { /* solo email */ }
```
**Tip**: en Spring, usar componentes pequeños con responsabilidades nítidas.

---

### 3.2 OCP — **Open/Closed Principle**
**Abierto a extensión, cerrado a modificación.**

**Olor**: cada vez que aparece un nuevo medio de pago editas un `switch` gigante.  
**Refactor**: polimorfismo vía **estrategias** registradas como beans.

```java
public interface ProcesadorPago { boolean procesar(Pago p); }

@Component("TARJETA") class PagoTarjeta implements ProcesadorPago { /* ... */ }
@Component("PSE")     class PagoPse implements ProcesadorPago { /* ... */ }

@Service
public class PagoService {
  private final Map<String, ProcesadorPago> estrategias;
  public PagoService(Map<String, ProcesadorPago> estrategias) { this.estrategias = estrategias; }
  public boolean pagar(Pago p) { return estrategias.get(p.medio()).procesar(p); }
}
```
**Spring** inyecta el `Map<String, ProcesadorPago>` con todas las implementaciones.

---

### 3.3 LSP — **Liskov Substitution Principle**
**Una subclase debe poder usarse donde se espera su superclase** sin romper invariantes.

**Olor**: subclase que lanza `UnsupportedOperationException` en métodos obligatorios, o cambia los contratos (pre/post-condiciones).  
**Refactor**: diseña jerarquías alineadas con comportamientos reales o usa **composición**.

```java
interface Exportador { byte[] exportar(Reporte r); } // contrato claro
class ExportadorCsv implements Exportador { /* respeta contrato */ }
class ExportadorPdf implements Exportador { /* respeta contrato */ }
// Evita subclases que “rompen” requisitos del contrato.
```

---

### 3.4 ISP — **Interface Segregation Principle**
**Prefiere muchas interfaces pequeñas y específicas** a interfaces “gordas”.

**Olor**: `RepositorioMega<T>` con 20 métodos que la mayoría de implementaciones no usa.  
**Refactor**: separa contratos por **capacidad**.

```java
interface Creador<T> { T crear(T t); }
interface Lector<T,ID> { Optional<T> porId(ID id); List<T> todos(); }
interface Actualizador<T> { T actualizar(T t); }
interface Eliminador<ID> { void eliminar(ID id); }
```
En Spring Data, compón lo necesario o crea repositorios específicos por agregado.

---

### 3.5 DIP — **Dependency Inversion Principle**
**Los módulos de alto nivel NO deben depender de módulos de bajo nivel, sino de **abstracciones**.**

**Olor**: `Service` dependiente de `JdbcTemplate` o `RestTemplate` directamente en el dominio.  
**Refactor**: define **puertos** (interfaces) y **adaptadores** (implementaciones) —ideal con **Hexagonal/Clean**—.

```java
// Dominio/Aplicación
public interface NotificacionPort { void enviar(Notificacion n); }

// Infraestructura
@Component
class EmailAdapter implements NotificacionPort { /* usa JavaMailSender */ }

// Orquestación
@Service
class AlertaService {
  private final NotificacionPort notificador;
  public AlertaService(NotificacionPort notificador) { this.notificador = notificador; }
}
```

---

## 4) Cómo SOLID se apoya en Spring (y viceversa)

- **DI (@Service/@Repository/@Component)** → facilita **DIP**.  
- **Profiles/Beans múltiples** → habilita **OCP** a través de estrategias intercambiables.  
- **Validación/Handlers** (Bean Validation, `@ControllerAdvice`) → favorece **SRP** al separar cross‑cutting.  
- **Interfaces finas** + **Repos/Services específicos** → promueve **ISP**.  
- **Contratos claros en servicios** + pruebas de sustitución → cuidan **LSP**.

---

## 5) Anti‑patrones habituales al migrar a Spring

1. **“Anémico con repositorio gordo”**: toda la lógica en el repositorio. → Sube reglas a **servicios de dominio** o **casos de uso**.  
2. **“Todo es @Service”**: clases monolíticas. → Segmentar en componentes especializados.  
3. **“Entidades JPA = modelo de dominio”** (a fuerza). → Usar **mappers** si necesitas invariantes y VO ricos.  
4. **“New por todos lados”**: rompe DI. → Solicita dependencias por **constructor**.  
5. **“Tests integrados para todo”**: lentos y frágiles. → Pirámide de pruebas (unit > slice > integration > e2e).

---

## 6) Mini‑ejemplos de refactor (antes → después)

### 6.1 Crear pedido (Java SE “rápido”)
```java
class PedidoController {
  void crear() {
    PedidoRepository repo = new PedidoRepository();
    Logger log = Logger.getLogger("pedido");
    // validar, crear, persistir, enviar email...
  }
}
```
**Problemas**: SRP, DIP, testabilidad.

**Spring + SOLID**
```java
@RestController
@RequestMapping("/api/pedidos")
class PedidoController {
  private final CrearPedidoUseCase crear;
  public PedidoController(CrearPedidoUseCase crear) { this.crear = crear; }

  @PostMapping
  ResponseEntity<?> crear(@Valid @RequestBody CrearPedidoDTO dto) {
    return ResponseEntity.ok(crear.ejecutar(dto));
  }
}

@Service
class CrearPedidoUseCase {
  private final PedidoRepositoryPort repo;
  private final NotificacionPort notificador;
  public CrearPedidoUseCase(PedidoRepositoryPort repo, NotificacionPort notificador) {
    this.repo = repo; this.notificador = notificador;
  }
  public Pedido ejecutar(CrearPedidoDTO dto) {
    // validar reglas -> dominio
    // guardar -> repo
    // notificar -> puerto
    return /* ... */;
  }
}
```

---

## 7) Checklist de diseño (aplícalo a cada historia de usuario)

- [ ] ¿Cada clase tiene **una** responsabilidad clara? (SRP)  
- [ ] ¿Puedo extender el comportamiento **sin editar** código estable? (OCP)  
- [ ] ¿La jerarquía respeta contratos y substitución? (LSP)  
- [ ] ¿Las interfaces son **pequeñas y enfocadas**? (ISP)  
- [ ] ¿Dependo de **abstracciones**, no de detalles? (DIP)  
- [ ] ¿El wiring de dependencias lo hace Spring (DI por constructor)?  
- [ ] ¿Las reglas de negocio viven fuera de infraestructura?  
- [ ] ¿Hay pruebas unitarias del dominio y de los casos de uso?  
- [ ] ¿Cross‑cutting (logs, validación, manejo de excepciones) está aislado?  
- [ ] ¿El código es legible y con lenguaje ubicuo del negocio?

---

## 8) Ejercicios guiados

1. **SRP**: Partir un `FacturaService` “todo en uno” en 3 componentes.  
2. **OCP**: Agregar un nuevo medio de pago **sin** modificar `PagoService`.  
3. **ISP**: Dividir una interfaz “mega‑repo” en 3 interfaces pequeñas y actualizar clases.  
4. **DIP**: Extraer un acceso directo a `RestTemplate` a un **puerto** e implementar 2 adaptadores (real y mock).  
5. **Tests**: Escribir unit tests de dominio (sin Spring) y slice tests de controlador (`@WebMvcTest`).

---

## 9) Buenas prácticas de implementación

- **Inyección por constructor** + `final` para dependencias.  
- **No uses `@Autowired` en campos**; evita setters salvo necesidad.  
- **Reglas en el dominio o casos de uso**, no en controladores/repositorios.  
- **DTOs** en bordes (web/mensajería), **no** en el dominio.  
- **Validación** con Bean Validation (`@NotNull`, `@Size`, `@Valid`) en DTOs.  
- **Mappers** (MapStruct o manuales) entre DTO ↔ dominio ↔ entidad JPA.  
- **Config explícita** (`@Configuration`) para casos de uso si sigues Clean/Hexagonal.  
- **Logs y excepciones controladas** (`@ControllerAdvice` con handlers).

---

## 10) Conclusión

- POO te da los **bloques básicos**; Spring te aporta **IoC/DI** para ensamblarlos de forma flexible.  
- **SOLID** es el “GPS” de diseño para que tu código se mantenga **mantenible, escalable y testeable**.  
- Juntos, te permiten pasar de un proyecto Java SE “correcto” a una base **profesional** lista para crecer (arquitecturas limpias, hexagonal, DDD y microservicios).

---

**Autor:** Javier Junior Ariza Montenegro  
**Versión:** 1.0  
**Licencia:** MIT
