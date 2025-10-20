# Día 2 — Principios SOLID aplicados en Spring (con puente desde Java SE y POO)

> Objetivo: comprender **cada principio SOLID** y cómo aplicarlo en **Spring Framework** usando los pilares de la **Programación Orientada a Objetos (POO)**.  
> Aprenderás a identificar malas prácticas comunes en Java SE y refactorizarlas usando la **Inversión de Dependencias, Abstracciones, Composición y Polimorfismo**, con ejemplos comentados, ventajas reales y comparaciones entre SOLID y POO.

---

## 1) Conexión general: de POO a SOLID

| Pilar POO | Principio SOLID relacionado | Descripción combinada |
|------------|-----------------------------|------------------------|
| **Encapsulamiento** | SRP, ISP | Separa responsabilidades y define interfaces pequeñas que ocultan la complejidad. |
| **Abstracción** | DIP | Depender de contratos (interfaces), no de implementaciones concretas. |
| **Herencia** | LSP | Las subclases deben comportarse igual que sus padres sin romper la lógica del sistema. |
| **Polimorfismo** | OCP, DIP | Permite extender comportamientos sin modificar código existente. |

---

## 2) SRP — Single Responsibility Principle (Responsabilidad Única)

**Idea:** una clase debe tener una sola razón para cambiar.

### Antes (Java SE acoplado)
```java
// Esta clase viola SRP: mezcla lógica de negocio, persistencia y notificación
public class FacturaService {
    public void generarFactura(Factura f) {
        // Calcular total
        double total = f.getItems().stream().mapToDouble(Item::getSubtotal).sum();
        f.setTotal(total);

        // Guardar en BD
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/facturas");
        // SQL INSERT ...

        // Enviar correo
        System.out.println("Enviando correo al cliente...");
    }
}
```

###  Después (Spring con SRP)
```java
@Service
public class CalculoFacturaService {
    public void calcular(Factura f) {
        double total = f.getItems().stream().mapToDouble(Item::getSubtotal).sum();
        f.setTotal(total);
    }
}

@Repository
public interface FacturaRepository extends JpaRepository<Factura, Long> {}

@Component
public class NotificadorEmail {
    public void enviar(String destino, String mensaje) {
        System.out.println("Enviando correo a " + destino);
    }
}
```

**Beneficios:**
- Cada clase tiene una sola responsabilidad.  
- Facilita pruebas unitarias por componente.  
- Cumple el principio de **Encapsulamiento** de POO: cada clase controla su propio estado y comportamiento.

**Ejemplo real:** dividir un módulo de “usuarios” en `UsuarioService`, `UsuarioRepository` y `EmailService`.

---

## 3) OCP — Open/Closed Principle (Abierto a extensión, cerrado a modificación)

**Idea:** el código debe poder extenderse sin modificar el existente.  
**Base POO:** *Polimorfismo y Abstracción.*

###  Antes
```java
// Si agrego un nuevo método de pago, debo modificar este switch
public class PagoService {
    public boolean procesar(Pago p) {
        switch (p.getMedio()) {
            case "TARJETA": return procesarTarjeta(p);
            case "PSE": return procesarPse(p);
            default: throw new IllegalArgumentException("Medio no soportado");
        }
    }
}
```

###  Después (Spring + Polimorfismo)
```java
public interface ProcesadorPago {
    boolean procesar(Pago p);
}

@Component("TARJETA")
public class PagoTarjeta implements ProcesadorPago {
    public boolean procesar(Pago p) { 
        System.out.println("Pago con tarjeta procesado.");
        return true;
    }
}

@Component("PSE")
public class PagoPse implements ProcesadorPago {
    public boolean procesar(Pago p) { 
        System.out.println("Pago PSE procesado.");
        return true;
    }
}

@Service
public class PagoService {
    private final Map<String, ProcesadorPago> estrategias;
    public PagoService(Map<String, ProcesadorPago> estrategias) { this.estrategias = estrategias; }

    public boolean pagar(Pago p) {
        return estrategias.get(p.getMedio()).procesar(p);
    }
}
```

**Beneficios:**
- Extiendes el sistema añadiendo nuevas clases (implementaciones) sin tocar el código existente.  
- Basado en el **Polimorfismo**: cada estrategia responde distinto al mismo método.  
- Facilita mantenimiento y escalabilidad.

**Escenario real:** agregar nuevos métodos de pago (Criptomoneda, Transferencia) sin tocar `PagoService`.

---

## 4) LSP — Liskov Substitution Principle (Sustitución de Liskov)

**Idea:** las subclases deben poder reemplazar a su superclase sin alterar el funcionamiento del sistema.  
**Base POO:** *Herencia con comportamiento coherente.*

###  Antes (violando LSP)
```java
class Reporte {
    public void exportar() { System.out.println("Exportando reporte..."); }
}

class ReportePdf extends Reporte {
    @Override
    public void exportar() { System.out.println("Exportando PDF"); }
}

class ReporteEmail extends Reporte {
    @Override
    public void exportar() {
        throw new UnsupportedOperationException("No puedo exportar como email");
    }
}
```

###  Después (cumpliendo LSP con interfaces)
```java
interface Exportador {
    void exportar(Reporte r);
}

@Component("PDF")
class ExportadorPdf implements Exportador {
    public void exportar(Reporte r) { System.out.println("Exportando PDF"); }
}

@Component("CSV")
class ExportadorCsv implements Exportador {
    public void exportar(Reporte r) { System.out.println("Exportando CSV"); }
}
```

**Beneficios:**
- Evita errores en tiempo de ejecución (`UnsupportedOperationException`).  
- Reemplazar una implementación no rompe la lógica de negocio.  
- Refuerza el principio de **Herencia y Polimorfismo** de la POO.

**Escenario real:** múltiples exportadores de reportes o distintos backends de almacenamiento.

---

## 5) ISP — Interface Segregation Principle (Segregación de Interfaces)

**Idea:** una interfaz grande debe dividirse en interfaces más pequeñas y específicas.  
**Base POO:** *Abstracción y Encapsulamiento.*

###  Antes (interfaz inflada)
```java
interface CrudRepository<T> {
    T crear(T t);
    T actualizar(T t);
    void eliminar(Long id);
    List<T> listar();
    void exportarExcel();  // 👈 no aplica a todos los repositorios
}
```

###  Después (interfaces pequeñas)
```java
interface Creador<T> { T crear(T t); }
interface Lector<T, ID> { Optional<T> porId(ID id); List<T> todos(); }
interface Actualizador<T> { T actualizar(T t); }
interface Eliminador<ID> { void eliminar(ID id); }

@Repository
public class ProductoRepository implements Creador<Producto>, Lector<Producto, Long>, Eliminador<Long> {
    // implementaciones concretas
}
```

**Beneficios:**
- Evita implementar métodos innecesarios.  
- Promueve la **Alta Cohesión** y el **Encapsulamiento**.  
- Facilita test unitario por responsabilidad.

**Escenario real:** separar interfaces de repositorio, servicio de negocio, auditoría, etc.

---

## 6) DIP — Dependency Inversion Principle (Inversión de Dependencias)

**Idea:** las clases de alto nivel no deben depender de clases concretas, sino de **abstracciones**.  
**Base POO:** *Abstracción y Polimorfismo.*

###  Antes (acoplado)
```java
public class AlertaService {
    private final EmailNotificador notificador = new EmailNotificador(); // dependencia fija
}
```

###  Después (Spring + DIP)
```java
public interface NotificacionPort {
    void enviar(Notificacion n);
}

@Component
public class EmailAdapter implements NotificacionPort {
    public void enviar(Notificacion n) {
        System.out.println("Email enviado a: " + n.getDestino());
    }
}

@Service
public class AlertaService {
    private final NotificacionPort notificador;
    public AlertaService(NotificacionPort notificador) {
        this.notificador = notificador;
    }
}
```

**Beneficios:**
- Desacopla el servicio del medio de notificación.  
- Permite sustituir adaptadores sin tocar el servicio (Email, SMS, Kafka, etc.).  
- Cumple el **DIP + OCP** y usa **Inversión de Control (IoC)**.  
- Ideal para arquitecturas limpias o hexagonales (puertos y adaptadores).

**Escenario real:** enviar notificaciones por distintos canales (correo, Slack, push).

---

## 7) Asociación con los pilares de la POO

| Pilar | Ejemplo en SOLID | Beneficio | En Spring |
|--------|------------------|------------|------------|
| **Encapsulamiento** | SRP, ISP | Cada clase controla su propia lógica sin exponer detalles | Beans separados con `@Service`, `@Repository`, etc. |
| **Abstracción** | DIP | Servicios dependen de interfaces, no de implementaciones concretas | Interfaces en dominio, adaptadores en infraestructura |
| **Herencia** | LSP | Subclases mantienen comportamiento coherente | Implementaciones múltiples bajo una misma interfaz |
| **Polimorfismo** | OCP, DIP | Se extiende el comportamiento sin modificar código | Inyección de `Map<String, Bean>` o estrategias con Spring |

---

## 8) Buenas prácticas generales

- Prefiere **composición sobre herencia**.  
- Usa **interfaces claras** con responsabilidades únicas.  
- Configura la **inyección de dependencias** por constructor (evita `@Autowired` en campos).  
- Aprovecha los **estereotipos** de Spring para organizar responsabilidades.  
- Usa **tests unitarios** con mocks de interfaces (gracias a DIP).

---

## 9) Mini práctica del día

1. Refactoriza tu `ProductoDAO` en interfaces separadas (CRUD dividido).  
2. Crea un `PrecioService` con estrategias (`OCP`) para aplicar descuentos distintos.  
3. Implementa un `NotificacionPort` con adaptador SMS falso (`DIP`).  
4. Verifica en IntelliJ que el contenedor crea e inyecta todos los beans.  
5. Escribe una prueba unitaria para el `PrecioService` sin depender de la base de datos.

---

## 10) Resumen visual

```
SRP → cada clase hace una sola cosa
OCP → extiende sin modificar
LSP → subclases coherentes
ISP → interfaces pequeñas
DIP → depende de abstracciones
```
**POO + SOLID = código mantenible, flexible y testeable.**