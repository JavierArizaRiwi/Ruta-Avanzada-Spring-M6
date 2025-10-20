
# Día 2 – Clean Architecture aplicada en Spring Boot 

En esta sesión profundizarás en cómo estructurar aplicaciones Spring Boot usando **Arquitectura Limpia (Clean Architecture)**. Conectarás lo aprendido en Java SE (capas, patrones, POO, JDBC) con una arquitectura **más mantenible, escalable y testable**.

---

## 1) De la arquitectura por capas a la arquitectura limpia

### 1.1 Recordatorio: arquitectura por capas clásica
```
UI/Controller → Service → Repository → Database
```
Ventajas:
- Simple de entender.  
- Adecuada para proyectos pequeños.

Desventajas:
- Acoplamiento entre capas.  
- Dificultad para testear el dominio aislado.  
- Las dependencias fluyen **de afuera hacia adentro**, lo que rompe el principio de independencia del dominio.

---

### 1.2 Clean Architecture (Arquitectura Limpia)
Propone **invertir las dependencias**: el dominio no conoce detalles externos.  
Cada capa tiene su propósito claro.

```
           +-----------------------------+
           |     Entrypoints (REST)      |  -> Controladores
           +-----------------------------+
           |     Application (Use Cases) |  -> Lógica de orquestación
           +-----------------------------+
           |     Domain (Core Business)  |  -> Entidades, Reglas de negocio
           +-----------------------------+
           | Infrastructure (Adapters)   |  -> DB, API, Kafka, Email
           +-----------------------------+
```

**Regla de oro:** las dependencias siempre apuntan hacia el **dominio**.

---

## 2) Estructura del proyecto base (Spring Boot)

```
com.riwi.academico
 ├─ domain/                        # Entidades, servicios de dominio, puertos (interfaces)
 │   ├─ model/
 │   ├─ service/
 │   └─ spi/                       # Ports - interfaces que definen contratos
 ├─ application/                   # Casos de uso (Application Services)
 │   ├─ usecase/
 │   └─ mapper/
 ├─ infrastructure/
 │   ├─ jpa/                       # Adaptadores de persistencia (Spring Data JPA)
 │   ├─ messaging/                 # Kafka, Email (otros adaptadores)
 │   └─ config/                    # Configuración de beans, perfiles, seguridad
 └─ entrypoints/
     └─ rest/                      # Controladores REST
```

---

## 3) Dominios, puertos y adaptadores

### 3.1 Puerto (definición de contrato)
```java
// domain/spi/EstudianteRepositoryPort.java
package com.riwi.academico.domain.spi;

import com.riwi.academico.domain.model.Estudiante;
import java.util.List;
import java.util.Optional;

public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    Optional<Estudiante> buscarPorId(String id);
    List<Estudiante> listarTodos();
}
```

### 3.2 Adaptador (implementa el puerto)
```java
// infrastructure/jpa/adapter/EstudianteJpaAdapter.java
package com.riwi.academico.infrastructure.jpa.adapter;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import com.riwi.academico.infrastructure.jpa.repository.EstudianteJpaRepository;
import com.riwi.academico.infrastructure.mapper.EstudianteMapper;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {
    private final EstudianteJpaRepository jpa;

    public EstudianteJpaAdapter(EstudianteJpaRepository jpa) { this.jpa = jpa; }

    @Override
    public Estudiante guardar(Estudiante e) {
        EstudianteEntity saved = jpa.save(EstudianteMapper.toEntity(e));
        return EstudianteMapper.toDomain(saved);
    }

    @Override
    public Optional<Estudiante> buscarPorId(String id) {
        return jpa.findById(id).map(EstudianteMapper::toDomain);
    }

    @Override
    public List<Estudiante> listarTodos() {
        return jpa.findAll().stream().map(EstudianteMapper::toDomain).collect(Collectors.toList());
    }
}
```

---

## 4) Caso de uso (Application Layer)
```java
// application/usecase/RegistrarEstudianteUseCase.java
package com.riwi.academico.application.usecase;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.spi.EstudianteRepositoryPort;
import org.springframework.stereotype.Service;

@Service
public class RegistrarEstudianteUseCase {
    private final EstudianteRepositoryPort repo;

    public RegistrarEstudianteUseCase(EstudianteRepositoryPort repo) { this.repo = repo; }

    public Estudiante ejecutar(String id, String nombre) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
        if (nombre == null || nombre.isBlank()) throw new IllegalArgumentException("nombre requerido");
        return repo.guardar(new Estudiante(id, nombre));
    }
}
```

---

## 5) Entrypoint REST (controlador)
```java
// entrypoints/rest/EstudianteController.java
package com.riwi.academico.entrypoints.rest;

import com.riwi.academico.application.usecase.RegistrarEstudianteUseCase;
import com.riwi.academico.domain.model.Estudiante;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final RegistrarEstudianteUseCase registrar;

    public EstudianteController(RegistrarEstudianteUseCase registrar){ this.registrar = registrar; }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestParam String id, @RequestParam String nombre){
        return ResponseEntity.ok(registrar.ejecutar(id, nombre));
    }
}
```

---

## 6) Comparativa: Arquitectura en capas vs Arquitectura limpia

| Concepto | Por capas | Clean Architecture |
|-----------|------------|--------------------|
| Flujo de dependencias | Hacia abajo | Hacia el dominio |
| Acoplamiento | Alto | Bajo |
| Pruebas unitarias | Difíciles | Sencillas |
| Dominio conoce framework | Sí | No |
| Sustituir DB o API | Requiere refactor | Solo otro adaptador |

---

## 7) Principios SOLID en acción

| Principio | Aplicación en Clean Architecture |
|------------|----------------------------------|
| SRP | Cada capa tiene una única responsabilidad |
| OCP | Nuevos adaptadores sin modificar lógica central |
| LSP | Sustitución de interfaces (puertos) sin romper el flujo |
| ISP | Puertos específicos en vez de interfaces genéricas enormes |
| DIP | Casos de uso dependen de puertos, no de implementaciones concretas |

---

## 8) Configuración de IntelliJ IDEA

1. **Crear proyecto Spring Boot**  
   - `File → New → Project → Spring Initializr`.  
   - Dependencias iniciales: *Spring Web*, *Spring Data JPA*, *H2 Database*, *Lombok*.

2. **Estructurar paquetes según Clean Architecture**  
   - `src/main/java/com/riwi/academico/domain/...`  
   - `application`, `infrastructure`, `entrypoints` como se mostró.

3. **Configurar perfiles**  
   - Crea `application-dev.yml` y activa con VM Option `-Dspring.profiles.active=dev`.

4. **Plugins necesarios**  
   - Lombok, SonarLint, Spring Tools.

5. **Annotation Processing**  
   - `Settings → Build → Compiler → Annotation Processors → Enable annotation processing`.

6. **Atajos clave**  
   - Buscar clase: `Ctrl+N` / `⌘O`  
   - Buscar método/símbolo: `Ctrl+Alt+Shift+N` / `⌘⌥O`  
   - Navegar entre capas: `Ctrl+B` / `⌘B` sobre interfaces o implementaciones.  
   - Reorganizar imports: `Ctrl+Alt+O` / `⌘⌥O`

7. **Inspecciones y SonarLint**  
   - Activa reglas para detectar clases grandes, métodos anidados, dependencias cíclicas.

---

## 9) Diagnóstico de problemas comunes

| Problema | Causa | Solución |
|-----------|--------|-----------|
| `NoSuchBeanDefinitionException` | Adaptador o caso de uso no escaneado | Verifica `@ComponentScan` |
| `Circular dependency` | A ↔ B se inyectan mutuamente | Introduce interfaz intermedia o separa responsabilidades |
| Error `Entity not found` | Falta repositorio JPA o mapeo | Verifica entidad y `@Repository` |
| Dificultad para testear casos de uso | Dependencias acopladas | Introduce mocks del puerto (Mockito) |

---

## 10) Buenas prácticas

- El **dominio no usa anotaciones de Spring**.  
- Los **puertos** solo definen contratos.  
- Los **adaptadores** implementan esos contratos.  
- Los **controladores** solo traducen peticiones a casos de uso.  
- Evita lógica de negocio en controladores o adaptadores.  
- Configura tests unitarios por capa (sin cargar el contexto completo).

---

## 11) Ejercicio de análisis conceptual

1. Analiza qué pasaría si cambias Spring Data JPA por JDBC en este proyecto.  
2. ¿Qué capas tendrías que tocar?  
3. ¿Dónde debería ubicarse la lógica de validación del estudiante?  
4. ¿Qué ventajas aporta tener el caso de uso como un bean explícito (`@Service` o `@Bean`)?

---

## 12) Resumen visual
```
REST Controller → UseCase (Application) → Port (Domain) ← Adapter (Infrastructure)
```

- El dominio **manda**.  
- Spring solo es el medio para conectar los componentes.

---

**Resultado esperado:**  
Proyecto Spring Boot modular, con dominio independiente, casos de uso explícitos y adaptadores limpios, listo para evolucionar hacia microservicios o DDD.