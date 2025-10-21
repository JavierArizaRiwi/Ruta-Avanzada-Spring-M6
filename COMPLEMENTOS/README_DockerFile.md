# Guía paso a paso: Crear imagen Docker y desplegar con Docker Compose

Esta guía explica cómo construir una **imagen Docker** de una aplicación Java Spring Boot (JDK 17 y Maven) y cómo desplegarla usando **Docker Compose**.

---

## 0) Estructura mínima del proyecto

```
/tu-proyecto
├─ pom.xml
├─ src/
│  └─ main/...
└─ Dockerfile
```

### Archivo opcional `.dockerignore`

Crea un archivo `.dockerignore` para acelerar los builds y evitar archivos innecesarios:

```
target/
.git/
.gitignore
.github/
.idea/
*.iml
*.log
.DS_Store
```

---

## 1) Configuración del `pom.xml`

Ejemplo completo del archivo de configuración Maven para tu aplicación:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.5.6</version>
		<relativePath/>
	</parent>

	<groupId>com.riwitienda</groupId>
	<artifactId>pagos</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>pagos</name>
	<description>Demo project for Spring Boot</description>

    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
	</dependencies>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <excludes>
                    <exclude>application.properties</exclude>
                </excludes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <includes>
                    <include>application.properties</include>
                </includes>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.3.1</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 2) Dockerfile (multi-stage)

Guarda el siguiente archivo como `Dockerfile` en la raíz del proyecto:

```dockerfile
# Etapa 1: Construcción con Maven y JDK 17
FROM maven:3.9.0-eclipse-temurin-17 AS build

WORKDIR /app

# Copiamos el pom.xml para aprovechar el cache de dependencias
COPY pom.xml .

# Descargamos las dependencias para que quede en caché
RUN mvn dependency:go-offline

# Copiamos el código fuente
COPY src ./src

# Construimos el paquete (saltamos los tests para agilizar)
RUN mvn clean package -DskipTests

# Etapa 2: Imagen final con JRE Temurin 17 Alpine (más ligera)
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copiamos el jar construido desde la etapa de build
COPY --from=build /app/target/*.jar app.jar

# Exponemos el puerto 8080 (por defecto de Spring Boot)
EXPOSE 8080

# Comando para ejecutar la aplicación
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 3) Construcción y ejecución de la imagen

```bash
# Construir la imagen
docker build -t pagos:1.0.0 .

# Ejecutar el contenedor
docker run --rm -p 8080:8080 --name pagos pagos:1.0.0
```

---

## 4) Docker Compose (solo la aplicación)

```yaml
services:
  pagos:
    build:
      context: .
      dockerfile: Dockerfile
    image: pagos:1.0.0
    container_name: pagos
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
    restart: unless-stopped
```

Comando para levantar:
```bash
docker compose up -d --build
docker compose logs -f pagos
```

---

## 5) Docker Compose con base de datos H2

```yaml
services:
  pagos:
    build:
      context: .
      dockerfile: Dockerfile
    image: pagos:1.0.0
    container_name: pagos
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:h2:mem:testdb
      SPRING_DATASOURCE_DRIVER_CLASS_NAME: org.h2.Driver
      SPRING_DATASOURCE_USERNAME: sa
      SPRING_DATASOURCE_PASSWORD: password
    restart: unless-stopped
```

---

## 6) Makefile (opcional)

```makefile
APP_IMAGE?=pagos:1.0.0

build:
	docker build -t $(APP_IMAGE) .

up:
	docker compose up -d --build

down:
	docker compose down

logs:
	docker compose logs -f pagos

ps:
	docker compose ps
```

---

## 7) Comandos principales

| Acción | Comando |
|--------|----------|
| Construir imagen | `docker build -t pagos:1.0.0 .` |
| Levantar contenedor | `docker compose up -d --build` |
| Ver logs | `docker compose logs -f pagos` |
| Apagar servicios | `docker compose down` |

---

**Autor:** Javier Ariza  
**Versión:** 1.0  
**Descripción:** Guía práctica para construir imágenes Docker de aplicaciones Spring Boot y desplegarlas con Docker Compose.