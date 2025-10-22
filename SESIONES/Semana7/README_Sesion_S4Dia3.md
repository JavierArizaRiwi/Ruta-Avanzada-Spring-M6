# Día 3 - Observabilidad Hexagonal con Actuator, Micrometer/Prometheus y Grafana **a través del API Gateway**

Cerramos la ruta montando **observabilidad completa** para nuestros microservicios siguiendo **arquitectura hexagonal** y consumiendo **todas las rutas vía API Gateway**. Tendrás **healthchecks**, **métricas de Prometheus** (Micrometer) y **dashboards Grafana**, sin costos, reproducible en local con Docker.

> Nota de estilo: todos los ejemplos **exponen/consumen por Gateway**. Los servicios mantienen capas hexagonales (dominio/puertos/adaptadores).

---

## 1) Objetivos del día

- Exponer **health**, **info** y **métricas Prometheus** con Spring Boot Actuator + Micrometer.  
- Diseñar **HealthIndicators** y **métricas de dominio** en la **capa de aplicación** (no en controladores).  
- Configurar **Prometheus** para scrapear **vía Gateway** (rutas `/usuarios/**`, `/pedidos/**`).  
- Publicar métricas y visualizarlas con **Grafana** usando dashboards listos.  
- Mantener el **diseño hexagonal**: métrica/health como adaptadores secundarios, casos de uso limpios.

---

## 2) Arquitectura (hexagonal + gateway centric)

```
Cliente / Prometheus / Grafana
            │
            ▼
       [ API Gateway ]
          /      \
         /        \
   lb://usuarios   lb://pedidos
      │                  │
  dominio + puertos   dominio + puertos
      │                  │
adapters: http/jpa/…  adapters: http/rabbit/…
      │                  │
   Actuator + Micrometer (expuestos vía Gateway)
```
Scrape Prometheus (educativo) **a través del Gateway**:
- `/usuarios/actuator/health`  
- `/usuarios/actuator/prometheus`  
- `/pedidos/actuator/health`  
- `/pedidos/actuator/prometheus`

---

## 3) Dependencias (por servicio)

```xml
<!-- Observabilidad -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Si usas RabbitMQ/Redis (Semana 7 Días 1–2), mantén sus starters; Actuator detecta *binders* y expone métricas.

---

## 4) Configuración Actuator (servicios) – `application.yml` externo (config-repo)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
  endpoint:
    health:
      probes.enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
```

Gateway debe **permitir el paso** de `/actuator/**` hacia cada servicio (con `StripPrefix=1`).

### Gateway (`application.yml`) – recordatorio
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

## 5) Métricas de dominio (hexagonal): **Timer** en caso de uso

Ubica el *cross-cutting* como un **adaptador secundario** que envuelve un **puerto de aplicación** (servicio).

```java
// application/ports/PedidoUseCase.java
package com.riwi.pedidos.application.ports;
public interface PedidoUseCase {
  Long registrar(String producto, int cantidad);
}
```

```java
// application/services/PedidoService.java (implementa el caso de uso)
package com.riwi.pedidos.application.services;

import com.riwi.pedidos.application.ports.PedidoUseCase;
import com.riwi.pedidos.domain.Pedido;
import com.riwi.pedidos.domain.ports.PedidoRepositoryPort;
import org.springframework.stereotype.Service;

@Service
public class PedidoService implements PedidoUseCase {

  private final PedidoRepositoryPort repo;
  public PedidoService(PedidoRepositoryPort repo){ this.repo = repo; }

  @Override
  public Long registrar(String producto, int cantidad) {
    if (cantidad <= 0) throw new IllegalArgumentException("cantidad inválida");
    var saved = repo.save(new Pedido(null, producto, cantidad));
    return saved.getId();
  }
}
```

```java
// infrastructure/metrics/MetricsPedidoUseCase.java (adaptador decorador)
package com.riwi.pedidos.infrastructure.metrics;

import com.riwi.pedidos.application.ports.PedidoUseCase;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

@Component
public class MetricsPedidoUseCase implements PedidoUseCase {

  private final PedidoUseCase delegate;
  private final Timer timer;

  public MetricsPedidoUseCase(PedidoUseCase delegate, MeterRegistry registry) {
    this.delegate = delegate;
    this.timer = Timer.builder("pedidos.usecase.registrar.duration")
        .description("latencia al registrar pedidos")
        .tag("usecase", "registrar")
        .publishPercentileHistogram()
        .register(registry);
  }

  @Override
  public Long registrar(String producto, int cantidad) {
    return timer.record(() -> delegate.registrar(producto, cantidad));
  }
}
```
> Para que el decorador funcione, registra `PedidoService` como `@Primary` o usa configuración explícita con `@Bean` que devuelva el proxy `MetricsPedidoUseCase`. Alternativa simple: inyecta `PedidoUseCase` donde lo uses, **no** la clase concreta.

---

## 6) HealthIndicator por puerto (hexagonal)

Define un **HealthIndicator** que verifique la dependencia del puerto (por ejemplo, RabbitMQ o Redis) sin acoplar al framework.

```java
// infrastructure/health/RabbitHealthIndicator.java
package com.riwi.pedidos.infrastructure.health;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class RabbitHealthIndicator implements HealthIndicator {

  private final RabbitTemplate rabbitTemplate;
  public RabbitHealthIndicator(RabbitTemplate rabbitTemplate){ this.rabbitTemplate = rabbitTemplate; }

  @Override
  public Health health() {
    try {
      rabbitTemplate.execute(channel -> { channel.queueDeclarePassive("notificaciones.queue"); return null; });
      return Health.up().withDetail("queue","notificaciones.queue").build();
    } catch (Exception e) {
      return Health.down(e).withDetail("queue","notificaciones.queue").build();
    }
  }
}
```
Disponible en `/actuator/health` (vía Gateway).

---

## 7) Docker Compose de observabilidad (Prometheus + Grafana)

Crea `observability/docker-compose.yml` y levanta **Prometheus** y **Grafana**.

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: riwi-prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana-oss:latest
    container_name: riwi-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
```

### `observability/prometheus.yml` (scrape **vía Gateway**)
```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "usuarios-service"
    metrics_path: /usuarios/actuator/prometheus
    static_configs:
      - targets: [ "host.docker.internal:8080" ]
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: "gateway->usuarios"

  - job_name: "pedidos-service"
    metrics_path: /pedidos/actuator/prometheus
    static_configs:
      - targets: [ "host.docker.internal:8080" ]
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: "gateway->pedidos"
```
> `host.docker.internal` permite que Prometheus (en contenedor) alcance el Gateway (host). Si usas Linux sin esta resolución, reemplaza por la IP del host.

Arranque:
```bash
cd observability
docker compose up -d
```

Prometheus UI: `http://localhost:9090`  
Grafana UI: `http://localhost:3000` (admin/admin)

---

## 8) Dashboards Grafana (educativo)

1. En Grafana → *Connections* → *Add data source* → **Prometheus** (`http://host.docker.internal:9090`).  
2. Importa dashboard de **Spring Boot / Micrometer** (ID sugerido de la galería: **4701** o **19105**).  
3. Filtra por `application="usuarios-service"` o `application="pedidos-service"` (agregado en `management.metrics.tags`).

Métricas útiles a observar:
- `http_server_requests_seconds_count/sum/max` por `uri` (vía Gateway).  
- `jvm_memory_used_bytes`, `process_cpu_usage`.  
- Métrica de dominio: `pedidos_usecase_registrar_duration_seconds_*` (timer custom).

---

## 9) Pruebas por Gateway (smoke)

```bash
# Health
curl http://localhost:8080/usuarios/actuator/health
curl http://localhost:8080/pedidos/actuator/health

# Métricas
curl http://localhost:8080/usuarios/actuator/prometheus | head
curl http://localhost:8080/pedidos/actuator/prometheus | head
```

En Prometheus, prueba consultas:
```
sum(rate(http_server_requests_seconds_count{uri=~"/api/.*"}[1m])) by (application, method)
histogram_quantile(0.9, sum(rate(pedidos_usecase_registrar_duration_seconds_bucket[5m])) by (le))
```

---

## 10) Laboratorio guiado (45–60 min)

1. Añadir Actuator + micrometer-prometheus a **usuarios** y **pedidos**.  
2. Confirmar que se exponen `/actuator/health` y `/actuator/prometheus` **por el Gateway**.  
3. Levantar `observability` (Prometheus + Grafana).  
4. Importar dashboard y verificar métricas HTTP, JVM y custom de dominio.  
5. Generar carga (curl loop o JMeter) y observar latencias/percentiles en Grafana.  
6. Romper a propósito una dependencia (apaga RabbitMQ/Redis) y validar **HealthIndicator DOWN**.

---

## 11) Buenas prácticas (hexagonal + observabilidad)

| Recomendación | Razón |
|---|---|
| Métricas/health como *adapters* secundarios | Mantiene dominio limpio |
| Etiquetar métricas con `application` y `usecase` | Facilidad de filtrado en Grafana |
| Scrape por Gateway en demos | Simplifica red y puertos (educativo) |
| Scrape directo a servicios (prod) | Evita SPOF y reduce latencia |
| Alertas en Prometheus + notifiers | Detección temprana de fallos |

---

## 12) Checklist del día

- Gateway enruta `/usuarios/**` y `/pedidos/**`.  
- Actuator habilitado con health + prometheus **en ambos servicios**.  
- Prometheus scrappea **vía Gateway**.  
- Grafana operativa con dashboard y métricas custom visibles.  
- HealthIndicators reportan dependencias **UP/DOWN** con detalles.

---

## 13) Resultado esperado

- Observabilidad **integrada** al ecosistema, respetando **hexagonal**.  
- Health y métricas accesibles **por Gateway**.  
- Dashboards Grafana mostrando **HTTP, JVM y métricas de dominio**.  
- Base lista para extender con **tracing distribuido** (OpenTelemetry/Zipkin) si lo deseas.