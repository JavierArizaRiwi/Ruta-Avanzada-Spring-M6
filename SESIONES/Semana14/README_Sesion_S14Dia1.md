# Día 1 — Spring Cloud: Config Server, Eureka y API Gateway

> **Estado curricular:** contenido avanzado de Semana 14. Eureka/Config Server deben justificarse; Docker Compose ya aporta DNS. Usar Cloud 2025.1.x con Boot 4.1.x.

En esta sesión montarás una **plataforma mínima de microservicios** totalmente gratuita y **gateway‑céntrica** con **Spring Cloud Config Server** (configuración centralizada), **Eureka Server** (descubrimiento de servicios) y **Spring Cloud Gateway** (enrutamiento). Los microservicios mantienen **arquitectura hexagonal**: casos de uso y dominio libres de framework; documentación (springdoc) y observabilidad (actuator) viven en **adapters de infraestructura**.  
Todas las pruebas y accesos externos se realizan **exclusivamente por el API Gateway**.

---

## 1) Objetivos del día

- Centralizar configuración con **Config Server** y un **repo de configuración** (por servicio/perfil).  
- Registrar microservicios en **Eureka** para descubrimiento dinámico.  
- Publicar servicios detrás de **API Gateway** con rutas limpias y versionadas.  
- Exponer **Swagger UI** por el Gateway (y opcionalmente **agregación** en una sola UI).  
- Mantener **hexagonal**: controladores, springdoc y actuator como **adapters**; dominio y casos de uso sin dependencias de Spring.

---

## 2) Arquitectura y flujo (hexagonal + gateway‑céntrico)

```
Cliente / QA / Tests
          │
          ▼
     [ API Gateway ]  ←── (config) ── [ Config Server ] ←── config-repo (git/local)
        /      \
       /        \
 lb://usuarios   lb://pedidos     (descubrimiento)  ←── [ Eureka Server ]
     │                │
  Adapters in/out   Adapters in/out
  (REST, doc,       (REST, doc,
   actuator, JPA,     actuator, JPA,
   Redis, Rabbit)     Redis, Rabbit)
       │                │
   UseCases/Ports   UseCases/Ports
       │                │
      Domain          Domain
```

**Clave:** todo el **tráfico externo** (Swagger, health, APIs) entra por el **Gateway**. Los servicios **no** son accesibles directamente desde el cliente.

---

## 3) Estructura de repositorio (multi‑módulo o multi‑proyecto)

```
/config-repo/                       # Repositorio de configuración (git o carpeta local)
  usuarios-service/
    application.yml
    application-dev.yml
  pedidos-service/
    application.yml
    application-dev.yml
  eureka-server/
    application.yml
  api-gateway/
    application.yml

/platform/
  config-server/
  eureka-server/
  api-gateway/

/services/
  usuarios-service/
  pedidos-service/
```

---

## 4) Config Server (puerto 8888)

### 4.1 Dependencia (pom.xml)
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

### 4.2 Aplicación
```java
// ConfigServerApplication.java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
  public static void main(String[] args) { SpringApplication.run(ConfigServerApplication.class, args); }
}
```

### 4.3 application.yml
```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          # Para taller educativo puedes usar un folder local como repo Git ya clonado:
          uri: file:///${user.home}/workspace/config-repo
          searchPaths: .
          cloneOnStart: true
```

**Prueba rápida:** `GET http://localhost:8888/usuarios-service/dev` debe retornar el YML para ese servicio/perfil.

---

## 5) Eureka Server (puerto 8761)

### 5.1 Dependencias
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### 5.2 Aplicación
```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
  public static void main(String[] args){ SpringApplication.run(EurekaServerApplication.class, args); }
}
```

### 5.3 application.yml
```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

**Prueba rápida:** abre `http://localhost:8761` (inicialmente sin instancias).

---

## 6) API Gateway (puerto 8080)

### 6.1 Dependencias
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!-- (opcional) springdoc en el Gateway para agregar múltiples UIs -->
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
  <version>3.0.3</version>
</dependency>
```

### 6.2 application.yml del Gateway
```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true       # permite lb://usuarios-service, etc.
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials
      routes:
        # Rutas explícitas (recomendado para talleres)
        - id: usuarios
          uri: lb://usuarios-service
          predicates:
            - Path=/usuarios/**
          filters:
            - StripPrefix=1
        - id: pedidos
          uri: lb://pedidos-service
          predicates:
            - Path=/pedidos/**
          filters:
            - StripPrefix=1

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

**Comportamiento esperado:**  
- `GET http://localhost:8080/usuarios/actuator/health` → proxea a usuarios.  
- `GET http://localhost:8080/pedidos/actuator/health` → proxea a pedidos.

---

## 7) Microservicio ejemplo: **usuarios-service** (puerto dinámico)

### 7.1 Dependencias principales
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>3.0.3</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 7.2 Estructura hexagonal mínima
```
com.riwi.usuarios
 ├─ domain/                       # Entidades y reglas puras
 ├─ application/
 │   ├─ ports/                    # Puertos (repos, cache, etc.)
 │   └─ usecase/                  # Casos de uso
 └─ infrastructure/
     ├─ adapters/
     │   ├─ in/rest/UsuarioController.java   # Controlador (REST adapter in)
     │   └─ out/...                          # Persistencia, cache, etc.
     └─ config/                              # springdoc/actuator/eureka
```

### 7.3 bootstrap.yml (lee config centralizada)
```yaml
spring:
  application:
    name: usuarios-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

### 7.4 application.yml (mínimo; idealmente en config-repo)
```yaml
server:
  port: 0   # puerto aleatorio
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
springdoc:
  swagger-ui:
    path: /swagger
  api-docs:
    path: /v3/api-docs
```

### 7.5 Controlador educativo (Adapter in)
```java
// infrastructure/adapters/in/rest/UsuarioController.java
@RestController
@RequestMapping("/api/v1/usuarios")
public class UsuarioController {

  @GetMapping("/ping")
  public Map<String,Object> ping() {
    return Map.of("service","usuarios-service","status","ok");
  }
}
```

**Vía Gateway:** `GET http://localhost:8080/usuarios/api/v1/usuarios/ping`

---

## 8) Microservicio ejemplo: **pedidos-service**

Reutiliza la base del **Día 1** (controlador, servicio, DTOs y manejo de errores) y agrega Eureka + springdoc como **adapters**.

### 8.1 Dependencias clave
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>3.0.3</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 8.2 bootstrap.yml
```yaml
spring:
  application:
    name: pedidos-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

### 8.3 application.yml (mínimo; externo en config-repo)
```yaml
server:
  port: 0
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
springdoc:
  swagger-ui:
    path: /swagger
  api-docs:
    path: /v3/api-docs
```

---

## 9) Documentación Swagger **a través del Gateway**

**Enfoque Simple:** acceder a cada UI por el gateway
- `http://localhost:8080/usuarios/swagger` → UI de usuarios-service
- `http://localhost:8080/pedidos/swagger` → UI de pedidos-service

**Agregación en el Gateway (opcional):**
```yaml
# api-gateway/application.yml (añadir)
springdoc:
  swagger-ui:
    path: /swagger
    urls:
      - name: usuarios
        url: /usuarios/v3/api-docs
      - name: pedidos
        url: /pedidos/v3/api-docs
```

Esto no **fusiona** contratos, pero entrega una **UI con pestañas** por servicio.

---

## 10) Configuración en el repo de configuración (config-repo)

Ejemplo mínimo para `usuarios-service/application-dev.yml`:
```yaml
server:
  port: 0

logging:
  level:
    root: info

management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
```

Repite un archivo similar para `pedidos-service`. Puedes externalizar también `api-gateway/application.yml` y `eureka-server/application.yml` si deseas control centralizado total.

---

## 11) Orden de arranque local (recomendado)

1. **Config Server** (8888)  
2. **Eureka Server** (8761)  
3. **usuarios-service** y **pedidos-service** (puertos aleatorios; verificar registro en Eureka)  
4. **API Gateway** (8080)

Verifica en `http://localhost:8761` que ambos servicios aparezcan **UP**.

---

## 12) Pruebas manuales de ruta (siempre por Gateway)

Usuarios:
```
GET http://localhost:8080/usuarios/api/v1/usuarios/ping
→ 200 {"service":"usuarios-service","status":"ok"}
```

Pedidos (ejemplo):
```
GET http://localhost:8080/pedidos/api/v1/pedidos
→ 200 [ ... ]
```

Swagger:
```
http://localhost:8080/usuarios/swagger
http://localhost:8080/pedidos/swagger
# o UI del gateway (agregación):
http://localhost:8080/swagger
```

---

## 13) Buenas prácticas (hexagonal + cloud)

| Práctica | Beneficio |
|----------|-----------|
| `server.port: 0` en servicios | Multiplica instancias en local para demos |
| `bootstrap.yml` para Config Server | Evita duplicar configs entre proyectos |
| `StripPrefix=1` en rutas del Gateway | URLs limpias y consistentes |
| OpenAPI y Actuator en **adapters** | Dominio y casos de uso limpios |
| `spring.application.name` consistente | Descubrimiento fiable en Eureka |
| Todo tráfico externo por **Gateway** | Simula topología real y simplifica seguridad |

---

## 14) Checklist del día

- Config Server sirviendo YML desde `config-repo` (puerto 8888).  
- Eureka Server mostrando los servicios registrados (8761).  
- Gateway (8080) enruta a `/usuarios/**` y `/pedidos/**`.  
- Swagger accesible vía gateway (simple o agregado).  
- Servicios con estructura **hexagonal** y adapters de doc/health aislados.

---

## 15) Resultado esperado

- Plataforma educativa operativa: **configuración centralizada**, **descubrimiento** y **gateway**.  
- Microservicios base **usuarios** y **pedidos** registrados, documentados y expuestos **solo por Gateway**.  
- Base lista para añadir **cache Redis/RabbitMQ** (Semana 7) y **observabilidad** (Actuator → Prometheus/Grafana).
