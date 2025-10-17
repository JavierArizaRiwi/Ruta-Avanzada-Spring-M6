# Historias de Usuario -- Semana 1 (Spring + IntelliJ IDEA)

## Sesión Día 1 -- Fundamentos y Configuración de Proyecto Spring

Este documento guía a personas que nunca han trabajado con **Spring**.
Instalarás las herramientas, crearás un proyecto **Spring Boot** en
**IntelliJ IDEA**, aprenderás atajos para **buscar clases/métodos**, y
expondrás tu primer endpoint.

------------------------------------------------------------------------

## Objetivos del día

1.  Instalar Java 17 y Maven.\
2.  Instalar IntelliJ IDEA Community.\
3.  Crear un proyecto **Spring Boot** (Spring Initializr).\
4.  Entender la estructura por paquetes (controller, service,
    repository, domain).\
5.  Ejecutar la app y probar un endpoint REST.\
6.  Usar **H2** en memoria para persistencia básica.\
7.  Dominar **búsquedas y navegación** en IntelliJ (clases, métodos,
    símbolos, usos).\
8.  Configurar Git/GitHub y flujos de ramas.\
9.  Documentar el proceso.

------------------------------------------------------------------------

## Paso 1 -- Instalar Java 17 y Maven

``` bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
sudo apt install -y maven
mvn -v
```

------------------------------------------------------------------------

## Paso 2 -- Instalar IntelliJ IDEA Community

``` bash
sudo snap install intellij-idea-community --classic
```

------------------------------------------------------------------------

## Paso 3 -- Crear el proyecto Spring Boot

**En IntelliJ:**\
*File → New Project → Spring Initializr*

-   **Group:** com.codeup\
-   **Artifact:** academico-spring\
-   **Dependencies:** Spring Web, Spring Boot DevTools, Lombok,
    Validation, Spring Data JPA, H2 Database

Estructura esperada:

    academico-spring
     ├─ src/main/java/com/codeup/academico
     │   ├─ AcademicoSpringApplication.java
     │   ├─ domain/
     │   ├─ repository/
     │   ├─ service/
     │   └─ web/
     ├─ src/main/resources/
     ├─ pom.xml

------------------------------------------------------------------------

## Paso 4 -- Primer endpoint REST

### domain/Estudiante.java

``` java
@Entity @Table(name = "estudiantes")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Estudiante {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private String nombre;
}
```

### repository/EstudianteRepository.java

``` java
public interface EstudianteRepository extends JpaRepository<Estudiante, Long> {
    boolean existsByNombre(String nombre);
}
```

### service/EstudianteService.java

``` java
public interface EstudianteService {
    Estudiante crear(String nombre);
    List<Estudiante> listar();
    Estudiante buscarPorId(Long id);
}
```

### service/EstudianteServiceImpl.java

``` java
@Service
@Transactional
public class EstudianteServiceImpl implements EstudianteService {
    private final EstudianteRepository repo;
    public EstudianteServiceImpl(EstudianteRepository repo) { this.repo = repo; }

    @Override
    public Estudiante crear(String nombre) {
        if (nombre == null || nombre.isBlank())
            throw new IllegalArgumentException("nombre requerido");
        if (repo.existsByNombre(nombre))
            throw new IllegalArgumentException("ya existe un estudiante con ese nombre");
        return repo.save(Estudiante.builder().nombre(nombre).build());
    }

    @Override
    public List<Estudiante> listar() { return repo.findAll(); }

    @Override
    public Estudiante buscarPorId(Long id) {
        return repo.findById(id)
                   .orElseThrow(() -> new IllegalArgumentException("estudiante no encontrado"));
    }
}
```

### web/EstudianteController.java

``` java
@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final EstudianteService service;
    public EstudianteController(EstudianteService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestParam String nombre) {
        return ResponseEntity.ok(service.crear(nombre));
    }

    @GetMapping
    public ResponseEntity<List<Estudiante>> listar() {
        return ResponseEntity.ok(service.listar());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Estudiante> porId(@PathVariable Long id) {
        return ResponseEntity.ok(service.buscarPorId(id));
    }
}
```

### application.properties

``` properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2
spring.datasource.url=jdbc:h2:mem:acad;DB_CLOSE_DELAY=-1;MODE=MySQL
spring.jpa.hibernate.ddl-auto=update
```

------------------------------------------------------------------------

## Paso 5 -- Ejecutar la aplicación

``` bash
mvn spring-boot:run
```

Prueba:

``` bash
curl -X POST "http://localhost:8080/api/estudiantes?nombre=Laura"
curl "http://localhost:8080/api/estudiantes"
```