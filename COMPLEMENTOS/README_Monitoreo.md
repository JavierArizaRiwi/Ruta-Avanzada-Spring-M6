# Monitorización de una app Spring Boot (H2) con Prometheus y Grafana usando Docker

Este instructivo explica paso a paso cómo configurar y monitorear una aplicación Spring Boot utilizando H2 como base de datos, Prometheus para la recolección de métricas y Grafana para la visualización. Este tutorial está diseñado para principiantes, por lo que no se asume ningún conocimiento previo.

---

## 1. Requisitos previos

Antes de comenzar, asegúrate de tener lo siguiente instalado en tu sistema:

- **Docker**: Herramienta para crear y administrar contenedores.
- **Docker Compose**: Orquestador para definir y ejecutar aplicaciones multicontenedor.
- **Java 17**: Versión requerida para la aplicación Spring Boot.
- **Maven**: Para la gestión de dependencias y construcción del proyecto.

Para verificar si están instalados, ejecuta los siguientes comandos:

```bash
java -version
mvn -version
docker --version
docker compose version
```

---

## 2. Crear la estructura del proyecto

Crea la siguiente estructura de carpetas para organizar tu proyecto:

```text
pagos/
  docker-compose.yml
  Dockerfile
  pom.xml
  prometheus/
    prometheus.yml
  src/
    main/
      java/
      resources/
```

---

## 3. Configurar las dependencias

Edita el archivo `pom.xml` y agrega las siguientes dependencias:

### Spring Boot Actuator

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Micrometer + Prometheus

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### H2 Database

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 4. Configurar Spring Boot

Edita el archivo `application.yml` en la carpeta `resources` con la siguiente configuración:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:pagosdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update
    database-platform: org.hibernate.dialect.H2Dialect

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 5. Configurar Prometheus

Crea un archivo `prometheus.yml` dentro de la carpeta `prometheus` con el siguiente contenido:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'pagos'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
```

---

## 6. Crear el Dockerfile

Crea un archivo `Dockerfile` en la raíz del proyecto con el siguiente contenido:

```dockerfile
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jdk-jammy
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 7. Crear el archivo Docker Compose

Crea un archivo `docker-compose.yml` en la raíz del proyecto con el siguiente contenido:

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
    container_name: pagos-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: h2
      SPRING_DATASOURCE_URL: jdbc:h2:mem:pagosdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
      MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "*"
      MANAGEMENT_ENDPOINT_PROMETHEUS_ENABLED: "true"
      MANAGEMENT_METRICS_EXPORT_PROMETHEUS_ENABLED: "true"

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    container_name: grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
```

---

## 8. Levantar los servicios

Ejecuta los siguientes comandos para iniciar los servicios:

```bash
docker compose down --remove-orphans
docker compose up --build
```

---

## 9. Verificar los servicios

Asegúrate de que los servicios estén funcionando accediendo a las siguientes URLs:

- **Aplicación**: [http://localhost:8080](http://localhost:8080)
- **Métricas Actuator**: [http://localhost:8080/actuator/prometheus](http://localhost:8080/actuator/prometheus)
- **Prometheus**: [http://localhost:9090](http://localhost:9090)
- **Grafana**: [http://localhost:3000](http://localhost:3000)

---

## 10. Configurar Prometheus en Grafana

1. Accede a Grafana en [http://localhost:3000](http://localhost:3000).
2. Ve a **Connections → Data sources → Add data source**.
3. Selecciona **Prometheus**.
4. Ingresa la URL: `http://prometheus:9090`.
5. Guarda la configuración.

---

## 11. Importar un Dashboard en Grafana

1. En Grafana, ve a **Dashboards → Import**.
2. Ingresa el ID del Dashboard: **11378**.
3. Selecciona el data source configurado previamente.
4. Haz clic en **Importar**.

---

¡Listo! Ahora tienes una aplicación Spring Boot completamente monitorizada con Prometheus y Grafana.