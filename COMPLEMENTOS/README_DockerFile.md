build:
up:
down:
logs:
ps:

# Guía Profesional: Dockerfile y Despliegue de Aplicaciones Spring Boot

> **Estado curricular:** referencia para `SESIONES/Semana12`. Los laboratorios usan Maven Wrapper, usuario no-root, healthcheck e imágenes fijadas.

Esta guía explica cómo crear, optimizar y desplegar imágenes Docker para aplicaciones Java Spring Boot, incluyendo buenas prácticas, ejemplos, recomendaciones y automatización con Docker Compose y Makefile.

---

## 1. Conceptos Clave de Docker

- **Imagen:** Paquete inmutable con todo lo necesario para ejecutar una app.
- **Contenedor:** Instancia aislada de una imagen en ejecución.
- **Dockerfile:** Script para construir imágenes personalizadas.
- **Docker Compose:** Orquestador para definir y ejecutar múltiples contenedores.

---

## 2. Estructura Recomendada del Proyecto

```
/mi-proyecto
├─ pom.xml
├─ src/
│  └─ main/...
├─ Dockerfile
├─ docker-compose.yml
└─ .dockerignore
```

### Ejemplo de `.dockerignore`
```
target/
.git/
.idea/
*.iml
*.log
.DS_Store
```

---

## 3. Dockerfile Óptimo para Spring Boot

```dockerfile
# Etapa 1: Build con Maven y JDK 17
FROM maven:3.9.0-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Etapa 2: Imagen final ligera
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Buenas prácticas:**
- Usa multi-stage para imágenes pequeñas y seguras.
- Expón solo los puertos necesarios.
- No incluyas credenciales ni archivos sensibles.

---

## 4. Automatización con Makefile

```makefile
APP_IMAGE?=miapp:1.0.0
build:
	docker build -t $(APP_IMAGE) .
up:
	docker compose up -d --build
down:
	docker compose down
logs:
	docker compose logs -f miapp
ps:
	docker compose ps
```

---

## 5. Ejemplo de `docker-compose.yml`

```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: miapp:1.0.0
    container_name: miapp
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
    restart: unless-stopped
```

**Con base de datos H2:**
```yaml
services:
  app:
    ... # igual que arriba
    environment:
      SPRING_DATASOURCE_URL: jdbc:h2:mem:testdb
      SPRING_DATASOURCE_DRIVER_CLASS_NAME: org.h2.Driver
      SPRING_DATASOURCE_USERNAME: sa
      SPRING_DATASOURCE_PASSWORD: password
```

---

## 6. Comandos Esenciales

| Acción                | Comando                                 |
|---------------------- |-----------------------------------------|
| Construir imagen      | `docker build -t miapp:1.0.0 .`         |
| Levantar contenedor   | `docker compose up -d --build`          |
| Ver logs              | `docker compose logs -f miapp`          |
| Apagar servicios      | `docker compose down`                   |

---

## 7. Checklist y Recomendaciones

- [x] Usa multi-stage en Dockerfile.
- [x] Define variables de entorno en Compose.
- [x] Excluye archivos innecesarios con `.dockerignore`.
- [x] Automatiza con Makefile.
- [x] Versiona tus imágenes.
- [x] Prueba local antes de subir a producción.

**Recomendaciones:**
- Mantén tu Dockerfile simple y explícito.
- Usa imágenes oficiales y actualizadas.
- Revisa los logs y el estado de los contenedores.
- Documenta los comandos y configuraciones.

---

## 8. Preguntas Frecuentes (FAQ)

**¿Por qué usar multi-stage?**
Reduce el tamaño y mejora la seguridad de la imagen.

**¿Cómo depurar errores de build?**
Revisa los logs, verifica rutas y dependencias.

**¿Cómo agregar una base de datos externa?**
Agrega otro servicio en `docker-compose.yml` (ejemplo: PostgreSQL, MySQL).

---

**Autor:** Javier Ariza  
**Versión:** 2.0  
**Descripción:** Guía profesional para crear, optimizar y desplegar imágenes Docker de aplicaciones Spring Boot.
