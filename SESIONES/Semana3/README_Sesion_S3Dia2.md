
# Día 2 - Relaciones Avanzadas, Consultas Personalizadas y Transacciones con Spring Data JPA

En esta sesión aprenderás a manejar **relaciones entre entidades**, crear **consultas personalizadas** con **JPQL y SQL nativo**, y controlar **transacciones** en Spring Data JPA. Además, verás cómo optimizar el acceso a datos y comprenderás el ciclo de vida de las entidades.

---

## 1) Objetivo del día

- Comprender y aplicar relaciones `@OneToMany`, `@ManyToOne`, `@ManyToMany`.  
- Crear consultas personalizadas con **JPQL** y **Native Query**.  
- Implementar transacciones con `@Transactional`.  
- Comprender `EntityManager` y el ciclo de vida de las entidades.  
- Aplicar buenas prácticas de rendimiento y organización del acceso a datos.  

---

## 2) Repaso: Ciclo de vida de una entidad JPA

| Estado | Descripción | Ejemplo |
|---------|--------------|---------|
| **Transient** | El objeto aún no está en la base de datos. | `new Estudiante("Ana")` |
| **Persistent** | El objeto está siendo gestionado por `EntityManager`. | `repo.save(estudiante)` |
| **Detached** | El objeto fue persistido pero ya no está bajo control del contexto. | Después de `em.clear()` |
| **Removed** | El objeto está marcado para eliminación. | `repo.delete(estudiante)` |

---

## 3) Relaciones entre entidades

### 3.1 `@ManyToOne` (muchos a uno)
```java
@Entity
@Table(name = "cursos")
public class Curso {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;

    @ManyToOne
    @JoinColumn(name = "profesor_id")
    private Profesor profesor;

    // getters y setters
}
```

### 3.2 `@OneToMany` (uno a muchos)
```java
@Entity
@Table(name = "profesores")
public class Profesor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;

    @OneToMany(mappedBy = "profesor", cascade = CascadeType.ALL)
    private List<Curso> cursos = new ArrayList<>();

    // getters y setters
}
```

### 3.3 `@ManyToMany` (muchos a muchos)
```java
@Entity
@Table(name = "estudiantes")
public class Estudiante {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;

    @ManyToMany
    @JoinTable(
        name = "estudiante_curso",
        joinColumns = @JoinColumn(name = "id_estudiante"),
        inverseJoinColumns = @JoinColumn(name = "id_curso")
    )
    private List<Curso> cursos = new ArrayList<>();

    // getters y setters
}
```

> Cada relación debe tener **una parte propietaria** que mantenga la referencia de la relación en la base de datos.

---

## 4) Consultas personalizadas

Spring Data JPA permite crear consultas con el nombre del método o con anotaciones `@Query`.

### 4.1 Query derivada (por nombre del método)
```java
List<Curso> findByNombreContainingIgnoreCase(String nombre);
```

### 4.2 JPQL con `@Query`
```java
@Query("SELECT c FROM Curso c WHERE c.profesor.nombre = :nombre")
List<Curso> buscarPorNombreDeProfesor(@Param("nombre") String nombre);
```

### 4.3 Native Query (SQL real)
```java
@Query(value = "SELECT * FROM cursos WHERE nombre LIKE %:nombre%", nativeQuery = true)
List<Curso> buscarPorNombreSQL(@Param("nombre") String nombre);
```

### 4.4 Query de actualización
```java
@Modifying
@Query("UPDATE Curso c SET c.nombre = :nuevo WHERE c.id = :id")
void actualizarNombre(@Param("id") Long id, @Param("nuevo") String nuevo);
```

---

## 5) Uso de Transacciones

Spring maneja automáticamente las transacciones mediante `@Transactional`.

### 5.1 Ejemplo simple
```java
@Service
public class CursoService {
    private final CursoRepository repo;

    public CursoService(CursoRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public void cambiarNombre(Long id, String nuevo) {
        repo.actualizarNombre(id, nuevo);
    }
}
```

### 5.2 Propagación de transacciones

| Tipo | Descripción |
|-------|--------------|
| `REQUIRED` | Usa una transacción existente o crea una nueva (por defecto). |
| `REQUIRES_NEW` | Siempre crea una nueva transacción. |
| `MANDATORY` | Exige una transacción activa, lanza error si no la hay. |
| `NEVER` | Lanza error si hay una transacción activa. |
| `SUPPORTS` | Ejecuta dentro o fuera de transacción si existe. |

Ejemplo:
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void guardarYNotificar(Curso curso) { ... }
```

---

## 6) EntityManager y consultas dinámicas

```java
@Repository
public class CursoCustomRepository {

    @PersistenceContext
    private EntityManager em;

    public List<Curso> buscarCursosPorProfesor(String nombre) {
        String jpql = "SELECT c FROM Curso c WHERE c.profesor.nombre = :nombre";
        return em.createQuery(jpql, Curso.class)
                 .setParameter("nombre", nombre)
                 .getResultList();
    }
}
```

**Ventajas:**  
- Control total sobre las consultas.  
- Ideal para casos complejos no cubiertos por Spring Data.  

---

## 7) Pruebas con `@DataJpaTest`

```java
@DataJpaTest
class CursoRepositoryTest {

    @Autowired
    private CursoRepository repo;

    @Test
    void debeGuardarYListarCursos() {
        Curso curso = new Curso();
        curso.setNombre("Spring Boot Intermedio");
        repo.save(curso);

        List<Curso> cursos = repo.findByNombreContainingIgnoreCase("spring");
        assertThat(cursos).isNotEmpty();
    }
}
```

---

## 8) Optimización y rendimiento

| Práctica | Descripción |
|-----------|--------------|
| `@FetchType.LAZY` | Carga diferida, mejora el rendimiento. |
| `@FetchType.EAGER` | Carga inmediata (solo cuando es necesario). |
| `@EntityGraph` | Controla qué relaciones se cargan en una query. |
| `@BatchSize(size=10)` | Reduce la cantidad de consultas repetitivas. |
| `JOIN FETCH` | Evita el problema de N+1 queries. |

Ejemplo:
```java
@Query("SELECT c FROM Curso c JOIN FETCH c.profesor")
List<Curso> listarConProfesor();
```

---

## 9) Configuración en IntelliJ IDEA

1. **Plugins requeridos:** Spring Boot, Database Tools, Lombok.  
2. **Atajos:**  
   - Buscar clases: `Ctrl+N` / `⌘O`  
   - Ejecutar test: `Ctrl+Shift+F10` / `⌘⇧R`  
   - Abrir consola SQL: `Alt+Shift+S` / `⌥⇧S`  
3. **Ver relaciones en el diagrama:**  
   - `View → Tool Windows → Persistence → Diagrams → Show Visualisation`

---

## 10) Buenas prácticas y errores comunes

| Error | Causa | Solución |
|-------|--------|-----------|
| `LazyInitializationException` | Acceso fuera del contexto de persistencia | Usa `@Transactional` o `fetch join` |
| `DataIntegrityViolationException` | Violación de FK o not null | Valida datos antes de persistir |
| `MultipleBagFetchException` | Dos colecciones `List` con `EAGER` | Cambiar a `Set` o `LAZY` |
| `N+1 Problem` | Carga de relaciones sin fetch | Usar `JOIN FETCH` o `@EntityGraph` |

---

## 11) Resultado esperado

- Relaciones entre entidades funcionales y persistidas correctamente.  
- Consultas JPQL y nativas en repositorios.  
- Transacciones controladas en servicios.  
- Tests exitosos de persistencia.  
- Código optimizado y libre de acoplamiento con el dominio.  