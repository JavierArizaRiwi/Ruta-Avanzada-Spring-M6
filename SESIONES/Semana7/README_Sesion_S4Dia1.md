# Día 1 - Comunicación Asíncrona con RabbitMQ **en Arquitectura Hexagonal** (Publisher–Subscriber educativo, acceso vía API Gateway)

En esta sesión integrarás **RabbitMQ** como *message broker* dentro de un ecosistema de **microservicios hexagonales**.  
Los endpoints HTTP para publicar eventos se **consumen a través del API Gateway**, mientras que la mensajería entre servicios se realiza **directamente por RabbitMQ** (no pasa por el gateway).  
Mantendremos el **dominio** y **casos de uso** libres de Spring y de RabbitMQ usando **puertos** y **adaptadores**.

---

## 1) Objetivos del día

- Diseñar **publisher** y **subscriber** con **puertos/adaptadores** (hexagonal).  
- Exponer un endpoint de **pedidos-service** (publisher) y llamarlo **vía Gateway**.  
- Consumir eventos en **notificaciones-service** usando un **adaptador de entrada** RabbitMQ que delega en un caso de uso.  
- Configurar **Docker** para **RabbitMQ** (gratuito) y verificar el flujo end-to-end.  
- Implementar **reintentos** y **manejo de errores** en el listener.

---

## 2) Arquitectura (hexagonal + gateway)

```
Cliente → API Gateway → pedidos-service (REST in adapter)
                          │
                          ▼
               (Aplicación / Caso de uso)
               ┌────────────────────────┐
               │ PublicarPedidoUseCase │
               │  └─ MessageBusPort ──────────┐
               └────────────────────────┘      │  (adapter out)
                        ▲                      │
                        │ Rest Controller      ▼
                                              RabbitMQ  ←───┐
                                                             │
notificaciones-service (adapter in: RabbitListener) ─────────┘
            │
            ▼
  ProcesarNotificacionUseCase (aplicación)
       └─ NotifierPort (adapter out: email/log, etc.)
```

**Reglas clave:**  
- Los **casos de uso** dependen solo de **puertos**.  
- **RabbitMQ** se usa como **adaptador** (salida en publisher, entrada en consumer).  
- El **API Gateway** se usa solo para **invocar endpoints HTTP** (p. ej., crear pedido).

---

## 3) Infraestructura local gratuita

Crea un contenedor con **RabbitMQ Management**:

```bash
docker run -d --hostname riwi-rabbit --name riwi-rabbit   -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```
UI: <http://localhost:15672> (user: `guest`, pass: `guest`).

---

## 4) Dependencias (en ambos servicios)

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
</dependency>
<!-- Para exponer/consumir endpoints y health -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

## 5) Configuración básica (`application.yml`)

### pedidos-service (publisher)
```yaml
spring:
  application:
    name: pedidos-service
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

app:
  rabbit:
    exchange: pedidos.exchange
    routingkey: pedidos.nuevo
    queue: notificaciones.queue
```

### notificaciones-service (consumer)
```yaml
spring:
  application:
    name: notificaciones-service
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

app:
  rabbit:
    queue: notificaciones.queue
```

---

## 6) Estructura de paquetes (ambos servicios con hexagonal)

```
com.riwi.pedidos
 ├─ domain/
 │   └─ model/PedidoEvent.java
 ├─ application/
 │   └─ ports/MessageBusPort.java
 │   └─ usecase/PublicarPedidoUseCase.java
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rest/PedidoController.java        # invocado vía Gateway
 │   │   └─ out/rabbit/RabbitMessageBusAdapter.java
 │   └─ config/RabbitConfig.java

com.riwi.notificaciones
 ├─ domain/
 │   └─ model/PedidoEvent.java
 ├─ application/
 │   ├─ ports/NotifierPort.java
 │   └─ usecase/ProcesarNotificacionUseCase.java
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/rabbit/PedidoEventListener.java   # RabbitListener → delega al use case
 │   │   └─ out/console/ConsoleNotifierAdapter.java
 │   └─ config/RabbitConfig.java
```

---

## 7) pedidos-service (publisher)

### 7.1 Dominio (evento)
```java
// domain/model/PedidoEvent.java
package com.riwi.pedidos.domain.model;

public class PedidoEvent {
  private Long id;
  private String producto;
  private int cantidad;

  public PedidoEvent() {}
  public PedidoEvent(Long id, String producto, int cantidad){
    this.id = id; this.producto = producto; this.cantidad = cantidad;
  }
  public Long getId(){ return id; }
  public String getProducto(){ return producto; }
  public int getCantidad(){ return cantidad; }
}
```

### 7.2 Puerto de mensajería
```java
// application/ports/MessageBusPort.java
package com.riwi.pedidos.application.ports;

import com.riwi.pedidos.domain.model.PedidoEvent;

public interface MessageBusPort {
  void publishPedidoCreado(PedidoEvent event);
}
```

### 7.3 Caso de uso
```java
// application/usecase/PublicarPedidoUseCase.java
package com.riwi.pedidos.application.usecase;

import com.riwi.pedidos.application.ports.MessageBusPort;
import com.riwi.pedidos.domain.model.PedidoEvent;

public class PublicarPedidoUseCase {

  private final MessageBusPort bus;
  public PublicarPedidoUseCase(MessageBusPort bus) { this.bus = bus; }

  public void ejecutar(PedidoEvent event) {
    if (event.getCantidad() <= 0) throw new IllegalArgumentException("cantidad inválida");
    bus.publishPedidoCreado(event);
  }
}
```

### 7.4 Adaptador Rabbit (salida)
```java
// infrastructure/adapters/out/rabbit/RabbitMessageBusAdapter.java
package com.riwi.pedidos.infrastructure.adapters.out.rabbit;

import com.riwi.pedidos.application.ports.MessageBusPort;
import com.riwi.pedidos.domain.model.PedidoEvent;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class RabbitMessageBusAdapter implements MessageBusPort {

  private final RabbitTemplate template;
  @Value("${app.rabbit.exchange}") private String exchange;
  @Value("${app.rabbit.routingkey}") private String routingKey;

  public RabbitMessageBusAdapter(RabbitTemplate template){ this.template = template; }

  @Override
  public void publishPedidoCreado(PedidoEvent event) {
    template.convertAndSend(exchange, routingKey, event);
  }
}
```

### 7.5 Configuración Rabbit (exchange/queue/binding)
```java
// infrastructure/config/RabbitConfig.java
package com.riwi.pedidos.infrastructure.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

  @Bean
  public TopicExchange pedidosExchange() { return new TopicExchange("pedidos.exchange"); }

  @Bean
  public Queue notificacionesQueue() { return new Queue("notificaciones.queue", true); }

  @Bean
  public Binding binding(Queue notificacionesQueue, TopicExchange pedidosExchange) {
    return BindingBuilder.bind(notificacionesQueue).to(pedidosExchange).with("pedidos.nuevo");
  }
}
```

### 7.6 Adaptador de entrada (REST) — invocado **vía Gateway**
```java
// infrastructure/adapters/in/rest/PedidoController.java
package com.riwi.pedidos.infrastructure.adapters.in.rest;

import com.riwi.pedidos.application.usecase.PublicarPedidoUseCase;
import com.riwi.pedidos.domain.model.PedidoEvent;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/pedidos")
public class PedidoController {

  private final PublicarPedidoUseCase useCase;

  public PedidoController(PublicarPedidoUseCase useCase) { this.useCase = useCase; }

  @PostMapping
  public ResponseEntity<String> crearPedido(@RequestBody PedidoEvent pedido) {
    useCase.ejecutar(pedido);
    return ResponseEntity.ok("Evento publicado");
  }
}
```

### 7.7 Wiring del caso de uso
```java
// infrastructure/config/AppConfig.java
package com.riwi.pedidos.infrastructure.config;

import com.riwi.pedidos.application.ports.MessageBusPort;
import com.riwi.pedidos.application.usecase.PublicarPedidoUseCase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {
  @Bean
  public PublicarPedidoUseCase publicarPedidoUseCase(MessageBusPort bus){
    return new PublicarPedidoUseCase(bus);
  }
}
```

---

## 8) notificaciones-service (subscriber)

### 8.1 Dominio (evento compatible)
```java
// domain/model/PedidoEvent.java
package com.riwi.notificaciones.domain.model;

public class PedidoEvent {
  private Long id;
  private String producto;
  private int cantidad;
  public PedidoEvent() {}
  public Long getId(){ return id; }
  public String getProducto(){ return producto; }
  public int getCantidad(){ return cantidad; }
}
```

### 8.2 Puerto de notificación (salida)
```java
// application/ports/NotifierPort.java
package com.riwi.notificaciones.application.ports;

public interface NotifierPort {
  void notify(String message);
}
```

### 8.3 Caso de uso (aplicación)
```java
// application/usecase/ProcesarNotificacionUseCase.java
package com.riwi.notificaciones.application.usecase;

import com.riwi.notificaciones.application.ports.NotifierPort;
import com.riwi.notificaciones.domain.model.PedidoEvent;

public class ProcesarNotificacionUseCase {

  private final NotifierPort notifier;
  public ProcesarNotificacionUseCase(NotifierPort notifier){ this.notifier = notifier; }

  public void ejecutar(PedidoEvent event){
    notifier.notify("Nuevo pedido: " + event.getProducto() + " x" + event.getCantidad());
  }
}
```

### 8.4 Adaptador de salida (consola / email simulado)
```java
// infrastructure/adapters/out/console/ConsoleNotifierAdapter.java
package com.riwi.notificaciones.infrastructure.adapters.out.console;

import com.riwi.notificaciones.application.ports.NotifierPort;
import org.springframework.stereotype.Component;

@Component
public class ConsoleNotifierAdapter implements NotifierPort {
  @Override public void notify(String message) { System.out.println("📣 " + message); }
}
```

### 8.5 Adaptador de entrada (RabbitListener) — delega al caso de uso
```java
// infrastructure/adapters/in/rabbit/PedidoEventListener.java
package com.riwi.notificaciones.infrastructure.adapters.in.rabbit;

import com.riwi.notificaciones.application.usecase.ProcesarNotificacionUseCase;
import com.riwi.notificaciones.domain.model.PedidoEvent;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class PedidoEventListener {

  private final ProcesarNotificacionUseCase useCase;
  public PedidoEventListener(ProcesarNotificacionUseCase useCase){ this.useCase = useCase; }

  @RabbitListener(queues = "notificaciones.queue")
  public void onMessage(PedidoEvent event) {
    useCase.ejecutar(event);
  }
}
```

### 8.6 Config y wiring
```java
// infrastructure/config/RabbitConfig.java
package com.riwi.notificaciones.infrastructure.config;

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {
  @Bean public Queue notificacionesQueue(){ return new Queue("notificaciones.queue", true); }
}
```

```java
// infrastructure/config/AppConfig.java
package com.riwi.notificaciones.infrastructure.config;

import com.riwi.notificaciones.application.ports.NotifierPort;
import com.riwi.notificaciones.application.usecase.ProcesarNotificacionUseCase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {
  @Bean
  public ProcesarNotificacionUseCase procesarNotificacionUseCase(NotifierPort notifier){
    return new ProcesarNotificacionUseCase(notifier);
  }
}
```

---

## 9) Pruebas **vía Gateway** (end-to-end educativo)

Asegúrate de tener el **API Gateway** con la ruta:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos
          uri: lb://pedidos-service
          predicates: [ Path=/pedidos/** ]
          filters: [ StripPrefix=1 ]
```

1) Levanta **RabbitMQ**, **notificaciones-service** y **pedidos-service**, más el **Gateway**.  
2) Publica un evento **por Gateway**:
```bash
curl -X POST http://localhost:8080/pedidos/api/v1/pedidos   -H "Content-Type: application/json"   -d '{"id":101,"producto":"Tablet","cantidad":1}'
```
3) Observa la consola de **notificaciones-service** (deberías ver el mensaje `📣 Nuevo pedido: Tablet x1`).  
4) Verifica en la UI de RabbitMQ el **exchange**, **cola** y **bindings**.

---

## 10) Manejo de errores y reintentos

Habilita reintentos simples en el listener:
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000ms
```

Simula un error controlado en el listener para ver los reintentos:
```java
@RabbitListener(queues = "notificaciones.queue")
public void onMessage(PedidoEvent event) {
  if ("ERROR".equalsIgnoreCase(event.getProducto())) {
    throw new RuntimeException("Error simulado");
  }
  useCase.ejecutar(event);
}
```

> Para flujos más avanzados: **DLQ** (Dead Letter Queue) y **manual acks** son el siguiente paso (fuera del alcance del día).

---

## 11) Observabilidad mínima

Expón health y métricas (por Gateway):
```
GET http://localhost:8080/pedidos/actuator/health
GET http://localhost:8080/notificaciones/actuator/health
```
En el **Día 3** verás cómo scrapear métricas con Prometheus y visualizarlas en Grafana.

---

## 12) Buenas prácticas (hexagonal + mensajería)

| Práctica | Beneficio |
|---|---|
| Puertos en aplicación; adapters para Rabbit | Dominios libres de framework |
| Endpoints invocados por Gateway | Simula tráfico real |
| Eventos **simples/compatibles** | Menor acoplamiento |
| Reintentos limitados + DLQ | Resiliencia sin bucles infinitos |
| Log de correlación (request-id) | Trazabilidad entre servicios |

---

## 13) Checklist del día

- ✅ `MessageBusPort` y `PublicarPedidoUseCase` en **publisher**.  
- ✅ `ProcesarNotificacionUseCase` y `NotifierPort` en **consumer**.  
- ✅ Adaptadores Rabbit y REST (entrada/salida) implementados.  
- ✅ Flujo probado **por API Gateway**.  
- ✅ Reintentos básicos comprobados.

---

## 14) Resultado esperado

- Ecosistema **asíncrono** funcionando con **RabbitMQ** bajo **arquitectura hexagonal**.  
- Publicación por endpoint **vía Gateway** y consumo con RabbitListener que **delegan en casos de uso**.  
- Base lista para **cache Redis** (Día 2) y **observabilidad** (Día 3).