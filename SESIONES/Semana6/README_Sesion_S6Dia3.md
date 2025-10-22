# Día 3 - Documentación centralizada **vía API Gateway** + Pruebas de Contrato y Healthchecks **en Arquitectura Hexagonal**

En esta sesión consolidarás la plataforma de microservicios del Día 2 respetando **arquitectura hexagonal**: los contratos OpenAPI, healthchecks y pruebas se organizan como **adaptadores** alrededor de casos de uso y puertos. **Todo el tráfico HTTP se consume por el API Gateway**; los servicios no exponen controladores “públicos” directos a clientes externos.

---

## 1) Objetivos del día

- Exponer **Swagger UI** de cada microservicio **a través del Gateway** (springdoc, sin pagos).  
- Habilitar **Spring Boot Actuator** y health probes (`liveness`, `readiness`) como **adaptadores secundarios**.  
- Implementar **pruebas de contrato** (OpenAPI JSON) y **smoke tests** contra el Gateway, no contra los servicios internos.  
- Mantener **hexagonal**: contratos y observabilidad viven en la **capa de infraestructura/adapters**, el **dominio** sigue limpio.  

---

## 2) Arquitectura (hexagonal + gateway-centric)

```
Cliente / Tests / Prometheus
            │
            ▼
       [ API Gateway ]
          /      \
         /        \
   lb://usuarios   lb://pedidos
      │                  │
 ┌───────────────┐  ┌───────────────┐
 │  Application  │  │  Application  │
 │  Ports/UseCases│  │ Ports/UseCases│
 └───────▲───────┘  └───────▲───────┘
         │                  │
   (adapters in/out)   (adapters in/out)
     REST (springdoc)     REST (springdoc)
     Actuator/Health      Actuator/Health
     JPA/Redis/Rabbit     JPA/Redis/Rabbit
```

**Clave:** Swagger (springdoc) y Actuator son **adaptadores de salida** (infraestructura). Casos de uso y dominio **no** dependen de ellos.

---

## 3) Dependencias a añadir (por servicio)

```xml
<!-- Observabilidad y documentación (adapters) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.6.0</version>
</dependency>
```

*(Opcional)* En Gateway puedes añadir springdoc para **agregar** UIs (solo referencia URLs), o dejar que el Gateway rote hacia cada servicio.

---

## 4) Configuración Actuator (servicios) – `application.yml` (mejor en Config Server)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
  endpoint:
    health:
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
```

> Esto permite health y métricas (para Semana 7 Día 3). Recuerda que se consumen **por Gateway**.

### Gateway – rutas necesarias (recordatorio)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: usuarios
          uri: lb://usuarios-service
          predicates: [ Path=/usuarios/** ]
          filters: [ StripPrefix=1 ]
        - id: pedidos
          uri: lb://pedidos-service
          predicates: [ Path=/pedidos/** ]
          filters: [ StripPrefix=1 ]
```

---

## 5) Swagger centralizado vía Gateway (sin plugins pagos)

**Opción A (simple):** acceder a cada UI del servicio por el Gateway

```
http://localhost:8080/usuarios/swagger
http://localhost:8080/pedidos/swagger
```

**Opción B (agregación en el Gateway):** el Gateway presenta una sola UI con varias fuentes

```yaml
# api-gateway/application.yml
springdoc:
  swagger-ui:
    path: /swagger
    urls:
      - name: usuarios
        url: /usuarios/v3/api-docs
      - name: pedidos
        url: /pedidos/v3/api-docs
```

> No fusiona contratos, pero los **agrega** en una interfaz única. Hexagonalmente, esto vive en **infraestructura del Gateway**.

---

## 6) Organización de código (hexagonal)

Ejemplo en `usuarios-service`:
```
com.riwi.usuarios
 ├─ domain/                  # Modelo, lógicas puras, sin Spring
 ├─ application/
 │   ├─ ports/               # Puertos de repos, cache, etc.
 │   └─ usecase/             # Casos de uso (servicios de aplicación)
 └─ infrastructure/
     ├─ adapters/
     │   ├─ in/rest/         # Controladores REST (springdoc anota aquí)
     │   └─ out/...          # JPA/Redis/Rabbit
     └─ config/              # Config YML/springdoc/actuator (adapters)
```

- **OpenAPI annotations** (`@Operation`, `@Schema`) se colocan en **adapters/in/rest**.  
- **Actuator** es puramente **infraestructura** (no mezclar con dominio).

---

## 7) Pruebas de contrato (OpenAPI) **contra Gateway**

Crea un módulo de pruebas de plataforma (p. ej. `platform-tests/`) con JUnit. Valida que los **docs del Gateway** expongan paths esperados.

```java
// platform-tests/src/test/java/com/riwi/platform/ContractOpenApiTest.java
package com.riwi.platform;

import org.json.JSONObject;
import org.junit.jupiter.api.Test;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.junit.jupiter.api.Assertions.*;

class ContractOpenApiTest {

  private final HttpClient client = HttpClient.newHttpClient();
  private final String gateway = "http://localhost:8080";

  @Test
  void usuarios_openapi_existe_y_contiene_ping() throws Exception {
    var res = client.send(
      HttpRequest.newBuilder(URI.create(gateway + "/usuarios/v3/api-docs")).GET().build(),
      HttpResponse.BodyHandlers.ofString());
    assertEquals(200, res.statusCode());
    var api = new JSONObject(res.body());
    assertTrue(api.getJSONObject("paths").has("/api/v1/usuarios/ping"),
      "Debe existir el path /api/v1/usuarios/ping");
    var info = api.getJSONObject("info");
    assertTrue(info.has("title") && info.has("version"));
  }

  @Test
  void pedidos_openapi_existe_y_tiene_endpoints() throws Exception {
    var res = client.send(
      HttpRequest.newBuilder(URI.create(gateway + "/pedidos/v3/api-docs")).GET().build(),
      HttpResponse.BodyHandlers.ofString());
    assertEquals(200, res.statusCode());
    var api = new JSONObject(res.body());
    assertTrue(api.getJSONObject("paths").length() > 0);
  }
}
```

> Para validaciones **estrictas** de esquemas: considera **openapi4j** más adelante (no requerido para el taller).

---

## 8) Smoke tests de health **por Gateway**

```java
// platform-tests/src/test/java/com/riwi/platform/HealthcheckGatewayTest.java
package com.riwi.platform;

import org.json.JSONObject;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.junit.jupiter.api.Assertions.assertEquals;

class HealthcheckGatewayTest {

  private final HttpClient client = HttpClient.newHttpClient();
  private final String gateway = "http://localhost:8080";

  @Test
  void health_usuarios() throws Exception {
    var res = client.send(
      HttpRequest.newBuilder(URI.create(gateway + "/usuarios/actuator/health")).GET().build(),
      HttpResponse.BodyHandlers.ofString());
    assertEquals(200, res.statusCode());
    var json = new JSONObject(res.body());
    assertEquals("UP", json.getString("status"));
  }

  @Test
  void health_pedidos() throws Exception {
    var res = client.send(
      HttpRequest.newBuilder(URI.create(gateway + "/pedidos/actuator/health")).GET().build(),
      HttpResponse.BodyHandlers.ofString());
    assertEquals(200, res.statusCode());
    var json = new JSONObject(res.body());
    assertEquals("UP", json.getString("status"));
  }
}
```

> Estos tests garantizan que la **plataforma completa** (rutas, gateway, servicios) está arriba antes de tus talleres.

---

## 9) Tips de configuración (taller)

- Usa `server.port: 0` en servicios; descubre los puertos con Eureka, pero **siempre prueba por Gateway**.  
- En el Gateway, `StripPrefix=1` para mapear `/usuarios/**` y `/pedidos/**`.  
- Expón solo endpoints necesarios de Actuator; agrega `prometheus` y `metrics` si seguirás con observabilidad (Semana 7).  
- Mantén **OpenAPI** (anotaciones de springdoc) en **adapters/in** y **nunca** en dominio/usecases.

---

## 10) Laboratorio guiado (30–45 min)

1. Agregar `actuator` y `springdoc` a **usuarios** y **pedidos** (infraestructura).  
2. Verificar rutas del Gateway (`/usuarios/**`, `/pedidos/**`).  
3. Probar UIs:  
   - `http://localhost:8080/usuarios/swagger`  
   - `http://localhost:8080/pedidos/swagger`  
   - (opcional) `http://localhost:8080/swagger` (agregado en Gateway).
4. Crear el módulo `platform-tests` y copiar las clases de prueba.  
5. Ejecutar `mvn -q -Dtest=*platform* test`.  
6. Romper a propósito un contrato (renombra un path o elimina una operación) y observar cómo cae el test.

---

## 11) Checklist del día

- Swagger UI accesible **por Gateway** para **usuarios** y **pedidos**.  
- Actuator habilitado; `health` responde `UP` por Gateway.  
- Pruebas de contrato OpenAPI **contra el Gateway**.  
- Smoke tests de plataforma (healthchecks) **por Gateway**.  
- Código organizado por **hexagonal** (contratos/observabilidad en adapters).

---

## 12) Resultado esperado

- Plataforma **documentada y testeada** desde el **API Gateway**.  
- Casos de uso y dominio **intactos** (sin dependencias de framework).  
- Contrato OpenAPI **controlado** por pruebas de plataforma; cambios no acordados **rompen el pipeline**.