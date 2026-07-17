
# Día 3 — Testing de persistencia y preparación del refactor arquitectónico

> **Estado curricular:** cierre de Semana 3 y puente hacia arquitectura de Semana 4. Las pruebas usan PostgreSQL Testcontainers, no H2 como sustituto del motor real.
## Persistencia y Testing Integrado con Spring Data JPA

En esta sesión comprenderás cómo **Spring Data JPA** encaja dentro de una **arquitectura limpia o hexagonal**, en contraste con la arquitectura tradicional por capas.
También aprenderás a probar la persistencia y los casos de uso usando **JUnit 5, Mockito** y el principio de **inversión de dependencias (DIP)**.

---

## 1) Objetivo del día

- Probar repositorios, migraciones y consultas sobre PostgreSQL real.
- Comparar arquitectura por capas vs arquitectura hexagonal como preparación de Semana 4.
- Implementar un flujo completo de persistencia con puertos y adaptadores.
- Aislar el dominio del framework mediante mappers y contratos.
- Probar la integración entre capas con `@DataJpaTest` y `@MockitoBean` cuando exista una colaboración Spring que reemplazar.
- Comprender cómo Spring Boot facilita la inyección de dependencias y las transacciones.

---

## 2) Arquitectura por capas vs arquitectura hexagonal

### 2.1 Arquitectura por capas (tradicional)

```
Controller → Service → Repository → Database
```

- **Ventajas:** sencilla de entender, rápida para proyectos pequeños.
- **Desventajas:** alto acoplamiento, difícil de probar y mantener; el dominio depende de la infraestructura.

Ejemplo de dependencia:
```java
@Service
public class EstudianteService {
    @Autowired
    private EstudianteRepository repository; // depende directamente del framework

    public List<Estudiante> listar() {
        return repository.findAll();
    }
}
```

El problema es que el dominio **no puede ser probado ni reutilizado** sin Spring.

---

### 2.2 Arquitectura hexagonal (puertos y adaptadores)

```
       Adaptador de Entrada (REST) → Puerto de Entrada (Caso de Uso)
Dominio (Lógica) ↔ Puerto de Salida (Repositorio) ← Adaptador de Salida (JPA)
```

- El dominio **no depende** del framework.
- Las dependencias apuntan **hacia adentro** (principio DIP).
- Se facilita la prueba y reemplazo de la infraestructura.

```java
// Puerto de salida (dominio)
public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    List<Estudiante> listar();
}

// Adaptador de salida (infraestructura)
@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    private final EstudianteJpaRepository repo;

    public EstudianteJpaAdapter(EstudianteJpaRepository repo){ this.repo = repo; }
    @Override
    public Estudiante guardar(Estudiante e){ return repo.save(e); }
    @Override
    public List<Estudiante> listar(){ return repo.findAll(); }
}
```

```java
// Caso de uso (dominio)
@Service
public class RegistrarEstudianteService {
    private final EstudianteRepositoryPort repo;

    public RegistrarEstudianteService(EstudianteRepositoryPort repo){ this.repo = repo; }

    public Estudiante ejecutar(String nombre){
        return repo.guardar(new Estudiante(nombre));
    }
}
```

**El dominio solo conoce la interfaz, no la implementación.**

---

## 3) Estructura de proyecto recomendada

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/
 │   ├─ ports/          # Interfaces (puertos de salida)
 │   └─ service/        # Casos de uso (puertos de entrada)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/         # REST Controller
 │   │   └─ out/        # JPA Adapter
 │   ├─ jpa/repository/ # Interfaces JpaRepository
 │   └─ config/         # Beans de configuración
 └─ application/
     └─ AcademicoApplication.java
```

---

## 4) Caso práctico: flujo completo con Spring Data JPA

### 4.1 Dominio
```java
// domain/model/Estudiante.java
@Entity
@Table(name = "estudiantes")
public class Estudiante {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    public Estudiante() {}
    public Estudiante(String nombre){ this.nombre = nombre; }
}
```

### 4.2 Puerto del dominio
```java
// domain/ports/EstudianteRepositoryPort.java
public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    List<Estudiante> listar();
}
```

### 4.3 Adaptador JPA
```java
// infrastructure/adapters/out/EstudianteJpaAdapter.java
@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    private final EstudianteJpaRepository repo;
    public EstudianteJpaAdapter(EstudianteJpaRepository repo){ this.repo = repo; }
    @Override
    public Estudiante guardar(Estudiante e){ return repo.save(e); }
    @Override
    public List<Estudiante> listar(){ return repo.findAll(); }
}
```

### 4.4 Repositorio JPA
```java
// infrastructure/jpa/repository/EstudianteJpaRepository.java
public interface EstudianteJpaRepository extends JpaRepository<Estudiante, Long> {}
```

### 4.5 Caso de uso
```java
// domain/service/RegistrarEstudianteService.java
@Service
public class RegistrarEstudianteService {
    private final EstudianteRepositoryPort repo;
    public RegistrarEstudianteService(EstudianteRepositoryPort repo){ this.repo = repo; }
    public Estudiante ejecutar(String nombre){ return repo.guardar(new Estudiante(nombre)); }
    public List<Estudiante> listar(){ return repo.listar(); }
}
```

### 4.6 Adaptador REST
```java
// infrastructure/adapters/in/EstudianteController.java
@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final RegistrarEstudianteService service;
    public EstudianteController(RegistrarEstudianteService service){ this.service = service; }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestParam String nombre){
        return ResponseEntity.ok(service.ejecutar(nombre));
    }

    @GetMapping
    public ResponseEntity<List<Estudiante>> listar(){
        return ResponseEntity.ok(service.listar());
    }
}
```

---

## 5) Testing con arquitectura limpia

### 5.1 Test del dominio (sin Spring)
```java
class RegistrarEstudianteServiceTest {
    @Test
    void debeGuardarEstudianteConMock() {
        EstudianteRepositoryPort mockRepo = Mockito.mock(EstudianteRepositoryPort.class);
        Mockito.when(mockRepo.guardar(Mockito.any())).thenAnswer(i -> i.getArguments()[0]);

        RegistrarEstudianteService service = new RegistrarEstudianteService(mockRepo);
        Estudiante e = service.ejecutar("Ana");

        assertEquals("Ana", e.getNombre());
        Mockito.verify(mockRepo).guardar(Mockito.any());
    }
}
```

### 5.2 Test de integración JPA

Configura `@DataJpaTest` con un `PostgreSQLContainer` y `@DynamicPropertySource`, siguiendo el ejemplo de Día 1. Así se validan dialecto, índices y migraciones reales.
```java
@DataJpaTest
class EstudianteJpaAdapterTest {
    @Autowired
    private EstudianteJpaRepository repo;

    @Test
    void debePersistirEstudiante() {
        Estudiante e = new Estudiante("Carlos");
        repo.save(e);
        assertThat(repo.findAll()).isNotEmpty();
    }
}
```

### 5.3 Test de capa REST con MockMvc
```java
@WebMvcTest(EstudianteController.class)
class EstudianteControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockitoBean
    private RegistrarEstudianteService service;

    @Test
    void debeCrearEstudiante() throws Exception {
        Mockito.when(service.ejecutar("Ana")).thenReturn(new Estudiante("Ana"));

        mvc.perform(post("/api/estudiantes").param("nombre", "Ana"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.nombre").value("Ana"));
    }
}
```

---

## 6) Comparativa directa

| Aspecto | Arquitectura por Capas | Arquitectura Hexagonal |
|----------|------------------------|-------------------------|
| Dependencias | Flujo unidireccional (Controller→Service→Repository) | Inversión de dependencias mediante puertos |
| Acoplamiento | Alto (el dominio depende del framework) | Bajo (dominio independiente) |
| Testing | Difícil de aislar | Testable por capas con mocks |
| Sustitución de tecnología | Costosa | Sencilla (nuevo adaptador) |
| Complejidad inicial | Menor | Mayor pero más escalable |

---

## 7) Configuración IntelliJ IDEA

1. Crear el proyecto desde **Spring Initializr** con dependencias:
   `Spring Web`, `Spring Data JPA`, PostgreSQL Driver, Flyway PostgreSQL, Testcontainers y Spring Boot Test.

2. Activar `Annotation Processing`:
   `Settings → Build → Compiler → Annotation Processors → Enable annotation processing`.

3. Ejecutar pruebas:
   - `Ctrl+Shift+F10` (Windows/Linux) / `⌘⇧R` (Mac).
   - Ver resultados en la pestaña *Run* o *JUnit*.

4. Para visualizar las capas:
   `File → Project Structure → Diagrams → Show Dependencies`.

---

## 8) Buenas prácticas

| Práctica | Beneficio |
|-----------|------------|
| Mantener el dominio libre de anotaciones JPA | Independencia del framework |
| Inyectar interfaces, no implementaciones | Facilita testing y extensión |
| Evitar lógica en los controladores | Mantiene responsabilidad única |
| Usar DTOs en la capa REST | Evita exponer entidades directamente |
| Centralizar la configuración en Beans | Control explícito de dependencias |

---

## 9) Resultado esperado

- Proyecto con arquitectura limpia funcionando con JPA.
- Dominio completamente aislado del framework.
- Pruebas de dominio, infraestructura y REST exitosas.
- Comparativa práctica entre arquitectura tradicional y hexagonal comprendida.
