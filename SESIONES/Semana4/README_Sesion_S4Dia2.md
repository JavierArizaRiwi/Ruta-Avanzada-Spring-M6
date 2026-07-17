# Día 2 — Arquitectura Hexagonal pragmática, mapeo y ADR

> **Estado curricular:** aplicación profunda de arquitectura en Semana 4. Comparar siempre con una alternativa más simple y documentar el costo.

En esta sesión aprenderás a diseñar aplicaciones **altamente desacopladas** siguiendo el enfoque de **Arquitectura Hexagonal (Ports & Adapters)**.
Este modelo fue introducido por *Alistair Cockburn* y es la base de muchas implementaciones modernas de **Clean Architecture** y **Domain-Driven Design (DDD)**.

---

## Contexto educativo

Este día marca la **transición natural** desde la **arquitectura por capas clásica** (que tus coders ya dominan) hacia un modelo **más limpio, modular y mantenible**.
El objetivo es entender que lo aprendido sobre *services*, *repositories* y *controllers* sigue siendo válido, pero ahora se organiza bajo principios de **independencia del dominio**, **inversión de dependencias** y **testabilidad aislada**.

---

## 1) ¿Qué es la Arquitectura Hexagonal?

La arquitectura hexagonal busca **separar el núcleo del sistema (el dominio y sus reglas)** de todo lo externo: bases de datos, APIs, interfaces gráficas, etc.
Su nombre proviene de la representación visual de un **hexágono**, donde cada lado simboliza un tipo de interfaz o conector externo.

```
               +-----------------------------+
               |      Adaptador de Entrada   |  → REST, CLI, Mensajería
               +-----------------------------+
                         ↓          ↑
+----------------------------------------------------------+
|                     Dominio / Aplicación                 |
|  ┌────────────────────────────────────────────────────┐  |
|  |   Puerto de Entrada (Input Port)  →  Casos de Uso   |  |
|  |   Puerto de Salida (Output Port)  ←  Adaptadores    |  |
|  └────────────────────────────────────────────────────┘  |
+----------------------------------------------------------+
                         ↑          ↓
               +-----------------------------+
               |     Adaptador de Salida     |  → BD, APIs externas
               +-----------------------------+
```

El dominio queda **aislado del framework y la infraestructura**, lo que permite **probarlo y reemplazar dependencias fácilmente**.

---

## 2) ¿Por qué se llaman “Gateways” o “Puertos”?

El término *gateway* o *port* hace referencia a un **punto de comunicación controlado** entre el dominio y el mundo exterior.
- Un **Input Port (Puerto de entrada)** define **qué operaciones ofrece el dominio** (casos de uso).
- Un **Output Port (Puerto de salida)** define **qué necesita el dominio del exterior** (por ejemplo, guardar datos o enviar notificaciones).

**Analogía:**
El dominio es una fortaleza, y los *ports* son las puertas por donde entran o salen las solicitudes.
Los *adapters* son los soldados o mensajeros que transforman los mensajes entre el mundo externo y el interno.

---

## 3) Estructura recomendada de proyecto

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/                # Entidades y reglas de negocio
 │   ├─ ports/                # Interfaces (gateways / puertos)
 │   └─ service/              # Servicios del dominio (casos de uso)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/               # Adaptadores de entrada (REST, CLI, etc.)
 │   │   └─ out/              # Adaptadores de salida (DB, Kafka, etc.)
 │   └─ config/               # Configuración de beans
 └─ application/
     └─ AcademicoApplication.java
```

---

## 4) Ejemplo práctico con Estudiantes

### 4.1 Dominio
```java
// domain/model/Estudiante.java
package com.riwi.academico.domain.model;

public class Estudiante {
    private final String id;
    private String nombre;

    public Estudiante(String id, String nombre) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id requerido");
        if (nombre == null || nombre.isBlank()) throw new IllegalArgumentException("nombre requerido");
        this.id = id;
        this.nombre = nombre;
    }

    public String getId() { return id; }
    public String getNombre() { return nombre; }
    public void renombrar(String nuevoNombre) {
        if (nuevoNombre == null || nuevoNombre.isBlank())
            throw new IllegalArgumentException("nombre requerido");
        this.nombre = nuevoNombre;
    }
}
```

### 4.2 Puerto de salida (Gateway)
```java
// domain/ports/EstudianteRepositoryPort.java
package com.riwi.academico.domain.ports;

import com.riwi.academico.domain.model.Estudiante;
import java.util.Optional;
import java.util.List;

public interface EstudianteRepositoryPort {
    Estudiante guardar(Estudiante e);
    Optional<Estudiante> buscarPorId(String id);
    List<Estudiante> listar();
}
```

### 4.3 Puerto de entrada (Caso de uso)
```java
// domain/service/RegistrarEstudianteService.java
package com.riwi.academico.domain.service;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.ports.EstudianteRepositoryPort;
import java.util.List;

public class RegistrarEstudianteService {
    private final EstudianteRepositoryPort repo;

    public RegistrarEstudianteService(EstudianteRepositoryPort repo) {
        this.repo = repo;
    }

    public Estudiante registrar(String id, String nombre) {
        return repo.guardar(new Estudiante(id, nombre));
    }

    public List<Estudiante> listar() { return repo.listar(); }
}
```

### 4.4 Adaptador de salida (Infraestructura)
```java
// infrastructure/adapters/out/EstudianteJpaAdapter.java
package com.riwi.academico.infrastructure.adapters.out;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.ports.EstudianteRepositoryPort;
import com.riwi.academico.infrastructure.jpa.entity.EstudianteEntity;
import com.riwi.academico.infrastructure.jpa.repository.EstudianteJpaRepository;
import com.riwi.academico.infrastructure.mapper.EstudianteMapper;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

@Repository
public class EstudianteJpaAdapter implements EstudianteRepositoryPort {

    private final EstudianteJpaRepository repo;

    public EstudianteJpaAdapter(EstudianteJpaRepository repo){ this.repo = repo; }

    @Override
    public Estudiante guardar(Estudiante e) {
        return EstudianteMapper.toDomain(repo.save(EstudianteMapper.toEntity(e)));
    }

    @Override
    public Optional<Estudiante> buscarPorId(String id) {
        return repo.findById(id).map(EstudianteMapper::toDomain);
    }

    @Override
    public List<Estudiante> listar() {
        return repo.findAll().stream().map(EstudianteMapper::toDomain).collect(Collectors.toList());
    }
}
```

### 4.5 Adaptador de entrada (REST Controller)
```java
// infrastructure/adapters/in/EstudianteController.java
package com.riwi.academico.infrastructure.adapters.in;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.service.RegistrarEstudianteService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {
    private final RegistrarEstudianteService service;

    public EstudianteController(RegistrarEstudianteService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestParam String id, @RequestParam String nombre){
        return ResponseEntity.ok(service.registrar(id, nombre));
    }

    @GetMapping
    public ResponseEntity<List<Estudiante>> listar(){
        return ResponseEntity.ok(service.listar());
    }
}
```

---

## 5) Wiring con Spring (JavaConfig)

```java
// infrastructure/config/BeanConfig.java
package com.riwi.academico.infrastructure.config;

import com.riwi.academico.domain.ports.EstudianteRepositoryPort;
import com.riwi.academico.domain.service.RegistrarEstudianteService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BeanConfig {

    @Bean
    public RegistrarEstudianteService registrarEstudianteService(EstudianteRepositoryPort repo) {
        return new RegistrarEstudianteService(repo);
    }
}
```

---

## 6) Beneficios concretos del enfoque Hexagonal

| Beneficio | Explicación |
|------------|-------------|
| **Desacoplamiento** | El dominio no depende de frameworks ni bases de datos. |
| **Sustituibilidad** | Cambiar JPA por Mongo o REST sin tocar el dominio. |
| **Testabilidad** | Se pueden crear mocks de los puertos fácilmente. |
| **Extensibilidad** | Nuevos adaptadores sin modificar el núcleo. |
| **Claridad** | Separa lo esencial (reglas) de lo accesorio (infraestructura). |

---

## 7) Testing de dominio con JUnit y Mockito

```java
// domain/service/RegistrarEstudianteServiceTest.java
package com.riwi.academico.domain.service;

import com.riwi.academico.domain.model.Estudiante;
import com.riwi.academico.domain.ports.EstudianteRepositoryPort;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class RegistrarEstudianteServiceTest {

    @Test
    void debeRegistrarEstudianteCorrectamente() {
        EstudianteRepositoryPort mockRepo = Mockito.mock(EstudianteRepositoryPort.class);
        Mockito.when(mockRepo.guardar(Mockito.any())).thenAnswer(i -> i.getArguments()[0]);

        RegistrarEstudianteService service = new RegistrarEstudianteService(mockRepo);
        Estudiante e = service.registrar("1", "Ana");

        assertEquals("Ana", e.getNombre());
        Mockito.verify(mockRepo).guardar(Mockito.any());
    }

    @Test
    void debeListarEstudiantes() {
        EstudianteRepositoryPort mockRepo = Mockito.mock(EstudianteRepositoryPort.class);
        Mockito.when(mockRepo.listar()).thenReturn(List.of(new Estudiante("1", "Carlos")));

        RegistrarEstudianteService service = new RegistrarEstudianteService(mockRepo);
        assertEquals(1, service.listar().size());
    }
}
```

---

## 8) Comparativa con Arquitectura por Capas

| Concepto | En arquitectura por capas | En arquitectura hexagonal |
|-----------|----------------------------|-----------------------------|
| Orquestación de reglas | `Service` (capa de aplicación) | **Caso de uso** (*Input Port*) |
| Acceso a datos | `Repository` (Spring Data) | **Puerto de salida** + **Adaptador (JPA)** |
| Exposición HTTP | `Controller` | **Adaptador de entrada (REST)** |
| Modelo de negocio | Entidades JPA | **Modelo de dominio puro (sin dependencias)** |
| Dependencias | Controller → Service → Repository | Adaptadores → Puertos → Dominio |
| Sustituir BD/API | Difícil, rompe varias capas | Solo se cambia el adaptador |
| Testabilidad | A menudo depende del contexto de Spring | Casos de uso aislados, fáciles de mockear |

**Resumen:**
El *Service* clásico se convierte en un **caso de uso**; los *repositories* en **puertos**; y las **implementaciones concretas** (JPA, REST, Kafka, etc.) en **adaptadores**.
El dominio no conoce ni Spring, ni JPA, ni HTTP.

---

## 9) Flujo visual de dependencias

```
Arquitectura por Capas:        Arquitectura Hexagonal:

Controller → Service → Repo    Adaptadores → Puertos → Dominio
(Depende de infra)             (Independiente del framework)
```

---

## 10) Conclusión

- La **Arquitectura Hexagonal** separa el dominio de la infraestructura.
- Los **puertos/gateways** definen los contratos del dominio.
- Los **adaptadores** conectan el mundo real con esos contratos.
- Con **Spring**, los beans se configuran para unir ambos mundos.
- Con **JUnit + Mockito**, se puede probar el dominio sin cargar el framework.

**Resultado esperado:**
Aplicación Spring Boot estructurada bajo principios hexagonales, con puertos y adaptadores claros, dominio independiente y casos de uso testeables con mocks.
