# Semana 7 – Día 2  
## Cache y Rendimiento con Redis en Spring Boot **con Arquitectura Hexagonal y acceso vía API Gateway**

En esta sesión integrarás **Redis** como **adaptador secundario** de caché dentro de una **arquitectura hexagonal**.  
El **dominio** y los **casos de uso** no dependen de Spring ni de Redis; en cambio usan **puertos** que luego implementamos con un **adaptador Redis**.  
Todas las rutas de prueba se consumen **a través del API Gateway**.

---

## 1) Objetivos del día

- Aplicar **cache distribuido** con Redis **sin acoplar** el dominio (puertos/adaptadores).  
- Introducir un **CachePort** genérico y su **RedisCacheAdapter**.  
- Cachear lecturas idempotentes desde el **caso de uso** (no en el controlador).  
- Exponer y consumir endpoints **vía Gateway**.  
- Mantener pruebas y monitoreo con Actuator (`/actuator/caches`, `/actuator/metrics`).

---

## 2) Arquitectura y flujo (hexagonal + gateway)

```
Cliente → API Gateway → usuarios-service (adaptador de entrada: REST Controller)
                         │
                         ▼
                 (Aplicación/Caso de uso)
                ┌──────────────────────────┐
                │ UsuarioQueryUseCase      │
                │  └─ usa UsuarioRepoPort  │──→ Adaptador JPA/H2 (salida)
                │  └─ usa CachePort        │──→ Adaptador Redis (salida)
                └──────────────────────────┘
                         │
                         ▼
                     Dominio puro
```

**Regla:** el dominio y la aplicación **no** conocen ni a Spring ni a Redis. Redis es un **adapter out** que implementa `CachePort`.

---

## 3) Repositorios y contenedores locales gratuitos

### Redis con Docker
```bash
docker run -d --name riwi-redis -p 6379:6379 redis:latest
```

> *Tip:* en Linux puedes usar `redis/redis-stack:latest` si deseas UI. Para el taller, la imagen oficial es suficiente.

### Gateway (recordatorio de rutas)
```yaml
# api-gateway/application.yml (fragmento)
spring:
  cloud:
    gateway:
      routes:
        - id: usuarios
          uri: lb://usuarios-service
          predicates: [ Path=/usuarios/** ]
          filters: [ StripPrefix=1 ]
```
Probaremos por **Gateway**: `http://localhost:8080/usuarios/api/v1/usuarios/{id}`

---

## 4) Dependencias (usuarios-service)

```xml
<!-- Web + Validación + Cache (anotaciones) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Redis como adaptador de salida -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- (opcional) Persistencia educativa -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>runtime</scope>
</dependency>

<!-- Observabilidad opcional -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

## 5) Configuración (`application.yml`)

```yaml
spring:
  application:
    name: usuarios-service
  redis:
    host: localhost
    port: 6379
  cache:
    type: redis

management:
  endpoints:
    web:
      exposure:
        include: "health,info,caches,metrics"
```

> Si tu config está centralizada (Config Server), ubica estas propiedades en `config-repo/usuarios-service`.

---

## 6) Paquetes (estructura impecable)

```
com.riwi.usuarios
 ├─ domain/
 │   └─ model/Usuario.java
 ├─ application/
 │   ├─ ports/
 │   │   ├─ UsuarioRepositoryPort.java
 │   │   └─ CachePort.java
 │   └─ service/UsuarioQueryService.java   # Caso de uso
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/UsuarioController.java
 │   │   └─ out/
 │   │       ├─ jpa/… (opcional educativo)
 │   │       └─ redis/RedisCacheAdapter.java
 │   └─ config/RedisConfig.java
 └─ UsuariosServiceApplication.java
```

---

## 7) Dominio puro

```java
// domain/model/Usuario.java
package com.riwi.usuarios.domain.model;

public class Usuario {
  private Long id;
  private String nombre;

  public Usuario(Long id, String nombre) { this.id = id; this.nombre = nombre; }
  public Long getId() { return id; }
  public String getNombre() { return nombre; }
}
```

---

## 8) Puertos de aplicación (contrato)

```java
// application/ports/UsuarioRepositoryPort.java
package com.riwi.usuarios.application.ports;

import com.riwi.usuarios.domain.model.Usuario;
import java.util.Optional;

public interface UsuarioRepositoryPort {
  Optional<Usuario> findById(Long id);
}
```

```java
// application/ports/CachePort.java
package com.riwi.usuarios.application.ports;

import java.util.Optional;

public interface CachePort {
  <T> Optional<T> get(String cacheName, String key, Class<T> type);
  void put(String cacheName, String key, Object value);
  void evict(String cacheName, String key);
}
```

> `CachePort` es **agnóstico** de Redis/Spring. Permite cambiar la tecnología de cache sin tocar el caso de uso.

---

## 9) Caso de uso (aplicación) — **lógica de cache aquí**

```java
// application/service/UsuarioQueryService.java
package com.riwi.usuarios.application.service;

import com.riwi.usuarios.application.ports.CachePort;
import com.riwi.usuarios.application.ports.UsuarioRepositoryPort;
import com.riwi.usuarios.domain.model.Usuario;

import java.util.NoSuchElementException;

public class UsuarioQueryService {

  private final UsuarioRepositoryPort repo;
  private final CachePort cache;
  private final String CACHE = "usuarios";

  public UsuarioQueryService(UsuarioRepositoryPort repo, CachePort cache) {
    this.repo = repo; this.cache = cache;
  }

  public Usuario obtenerPorId(Long id) {
    String key = String.valueOf(id);

    // 1) Buscar en cache (no depende de Spring)
    var cached = cache.get(CACHE, key, Usuario.class);
    if (cached.isPresent()) return cached.get();

    // 2) Ir al repositorio y popular cache
    var usuario = repo.findById(id).orElseThrow(() -> new NoSuchElementException("usuario no encontrado"));
    cache.put(CACHE, key, usuario);
    return usuario;
  }

  public void invalidar(Long id) {
    cache.evict(CACHE, String.valueOf(id));
  }
}
```

> Observa que usamos `CachePort` directamente en la **aplicación**; así evitamos anotar con `@Cacheable` en el servicio. Totalmente **hexagonal**.

---

## 10) Adaptador Redis (salida)

```java
// infrastructure/adapters/out/redis/RedisCacheAdapter.java
package com.riwi.usuarios.infrastructure.adapters.out.redis;

import com.riwi.usuarios.application.ports.CachePort;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.Optional;

@Component
public class RedisCacheAdapter implements CachePort {

  private final RedisTemplate<String, Object> redis;

  public RedisCacheAdapter(RedisTemplate<String, Object> redis) { this.redis = redis; }

  private String namespaced(String cacheName, String key) { return cacheName + "::" + key; }

  @Override
  public <T> Optional<T> get(String cacheName, String key, Class<T> type) {
    Object raw = redis.opsForValue().get(namespaced(cacheName, key));
    return Optional.ofNullable(type.cast(raw));
  }

  @Override
  public void put(String cacheName, String key, Object value) {
    // TTL educativa de 60s; podrías parametrizarla
    redis.opsForValue().set(namespaced(cacheName, key), value, Duration.ofSeconds(60));
  }

  @Override
  public void evict(String cacheName, String key) {
    redis.delete(namespaced(cacheName, key));
  }
}
```

### Configuración de `RedisTemplate` y serialización
```java
// infrastructure/config/RedisConfig.java
package com.riwi.usuarios.infrastructure.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

  @Bean
  public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);
    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
    template.afterPropertiesSet();
    return template;
  }
}
```

---

## 11) Adaptador de entrada (REST) — **siempre por Gateway**

```java
// infrastructure/adapters/in/rest/UsuarioController.java
package com.riwi.usuarios.infrastructure.adapters.in.rest;

import com.riwi.usuarios.application.service.UsuarioQueryService;
import com.riwi.usuarios.domain.model.Usuario;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/usuarios")
public class UsuarioController {

  private final UsuarioQueryService useCase;

  public UsuarioController(UsuarioQueryService useCase) { this.useCase = useCase; }

  @GetMapping("/{id}")
  public ResponseEntity<Usuario> obtener(@PathVariable Long id) {
    return ResponseEntity.ok(useCase.obtenerPorId(id));
  }

  @DeleteMapping("/{id}/cache")
  public ResponseEntity<Void> invalidar(@PathVariable Long id) {
    useCase.invalidar(id);
    return ResponseEntity.noContent().build();
  }
}
```

> Asegúrate de exponer y consumir **por Gateway**:  
> `GET http://localhost:8080/usuarios/api/v1/usuarios/1`  
> `DELETE http://localhost:8080/usuarios/api/v1/usuarios/1/cache`

---

## 12) Wiring (beans) del caso de uso

Si tu `UsuarioQueryService` no es un `@Service` (para mantenerlo puro), crea el bean en configuración:

```java
// infrastructure/config/AppConfig.java
package com.riwi.usuarios.infrastructure.config;

import com.riwi.usuarios.application.ports.CachePort;
import com.riwi.usuarios.application.ports.UsuarioRepositoryPort;
import com.riwi.usuarios.application.service.UsuarioQueryService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

  @Bean
  public UsuarioQueryService usuarioQueryService(UsuarioRepositoryPort repo, CachePort cache) {
    return new UsuarioQueryService(repo, cache);
  }
}
```

---

## 13) Repositorio educativo (opcional)

```java
// infrastructure/adapters/out/jpa/JpaUsuarioRepository.java
package com.riwi.usuarios.infrastructure.adapters.out.jpa;

import com.riwi.usuarios.application.ports.UsuarioRepositoryPort;
import com.riwi.usuarios.domain.model.Usuario;
import org.springframework.stereotype.Repository;

import java.util.Map;
import java.util.Optional;

@Repository
public class JpaUsuarioRepository implements UsuarioRepositoryPort {
  // Simulación in-memory; reemplaza por Spring Data JPA real si lo deseas
  private static final Map<Long,String> DATA = Map.of(1L,"Ana",2L,"Luis",3L,"María");
  @Override
  public Optional<Usuario> findById(Long id) {
    return Optional.ofNullable(DATA.get(id)).map(n -> new Usuario(id, n));
  }
}
```

---

## 14) Prueba educativa (local, vía Gateway)

1. Levanta Redis (`docker run …`).  
2. Levanta `usuarios-service`, el **Gateway**, y asegúrate de tener la ruta `/usuarios/**`.  
3. Realiza dos consultas iguales por Gateway:

```bash
curl http://localhost:8080/usuarios/api/v1/usuarios/1
curl http://localhost:8080/usuarios/api/v1/usuarios/1
```

**Salida esperada (logs del servicio):** la primera consulta va al repositorio; la segunda viene de cache (no verás el acceso al repo).

4. Verifica claves en Redis:
```bash
docker exec -it riwi-redis redis-cli
> keys *
> get "usuarios::1"
```

5. Invalida cache y vuelve a consultar:
```bash
curl -X DELETE http://localhost:8080/usuarios/api/v1/usuarios/1/cache
curl http://localhost:8080/usuarios/api/v1/usuarios/1
```

---

## 15) Observabilidad básica

Habilita Actuator (ya en `application.yml`) y consulta **por Gateway**:
```
http://localhost:8080/usuarios/actuator/caches
http://localhost:8080/usuarios/actuator/metrics
```
En Semana 7 Día 3 configuraste Prometheus/Grafana para ver **hits/misses** y latencias.

---

## 16) Buenas prácticas (hexagonal + cache)

| Práctica | Beneficio |
|---|---|
| Cache en **caso de uso** (no en controlador) | Mantiene dominio limpio y reutilizable |
| TTL razonable (ej. 60–300s) | Datos frescos y respuestas rápidas |
| Evitar cachear `null`/errores | Menos inconsistencias |
| Invalidación explícita en flujos de escritura | Coherencia del dato |
| Prefijos por servicio `{cache}::` | Evita colisiones entre microservicios |
| Probar siempre **vía Gateway** | Mismo path de tráfico que en prod |

---

## 17) Checklist del día

- ✅ `CachePort` y `UsuarioRepositoryPort` definidos.  
- ✅ `UsuarioQueryService` usa cache sin depender de Spring.  
- ✅ `RedisCacheAdapter` implementa el puerto.  
- ✅ Endpoints accedidos **vía Gateway**.  
- ✅ Actuator muestra `caches` y `metrics`.  

---

## 18) Resultado esperado

- **Cache distribuido** operando con **Redis** como adaptador secundario.  
- Arquitectura **hexagonal** respetada (dominio limpio, puertos/adaptadores).  
- Endpoints consumidos **por API Gateway**.  
- Proyecto listo para integrarse con **Prometheus/Grafana** y políticas de invalidación más avanzadas.