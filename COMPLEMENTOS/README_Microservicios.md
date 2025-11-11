
# Guía Profesional y Funcional: Microservicios con Spring Boot, Node.js y Docker Compose

> **Propósito:** Esta guía te lleva paso a paso, con comentarios y explicaciones, para crear dos microservicios interoperables (Java y Node.js), orquestados con Docker Compose y conectados a MySQL. Ideal para principiantes y para enseñar en clase.

---

## 1. Requisitos Previos

- Docker y Docker Compose instalados ([descarga aquí](https://docs.docker.com/get-docker/)).
- Java 17+ y Maven ([descarga aquí](https://adoptium.net/)).
- Node.js v16+ ([descarga aquí](https://nodejs.org/)).
- Postman o curl para pruebas.

---

## 2. Estructura del Proyecto

```text
micro-edtech/
  catalog-service/        # Microservicio Java Spring Boot
  lms-service/            # Microservicio Node.js Express
  docker-compose.yml      # Orquestador de servicios
```

---

## 3. Microservicio 1: catalog-service (Spring Boot)

### 3.1. Crear el proyecto
Usa [Spring Initializr](https://start.spring.io) con:
- Spring Web
- Spring Data JPA
- MySQL Driver
- Lombok (opcional)
Nombre: `catalog-service`  
Group: `com.edtech`  
Package: `com.edtech.catalog`

Descomprime el zip en `micro-edtech/catalog-service`.

### 3.2. Dominio y arquitectura hexagonal
**Dominio:**
```java
// src/main/java/com/edtech/catalog/domain/Curso.java
package com.edtech.catalog.domain;

public class Curso {
    private Long id;
    private String nombre;
    private String estado; // ACTIVO / INACTIVO

    public Curso(Long id, String nombre, String estado) {
        this.id = id;
        this.nombre = nombre;
        this.estado = estado;
    }

    // Getters
    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public String getEstado() { return estado; }
}
```

**Puerto del dominio:**
```java
// src/main/java/com/edtech/catalog/domain/CursoRepositoryPort.java
package com.edtech.catalog.domain;

import java.util.List;

public interface CursoRepositoryPort {
    List<Curso> listarCursos();
}
```

**Caso de uso:**
```java
// src/main/java/com/edtech/catalog/application/ListarCursosService.java
package com.edtech.catalog.application;

import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;
import java.util.List;

public class ListarCursosService {
    private final CursoRepositoryPort repository;
    public ListarCursosService(CursoRepositoryPort repository) {
        this.repository = repository;
    }
    public List<Curso> ejecutar() {
        return repository.listarCursos();
    }
}
```

**Adaptador en memoria:**
```java
// src/main/java/com/edtech/catalog/infrastructure/InMemoryCursoAdapter.java
package com.edtech.catalog.infrastructure;

import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
public class InMemoryCursoAdapter implements CursoRepositoryPort {
    @Override
    public List<Curso> listarCursos() {
        return List.of(
            new Curso(1L, "Módulo 5 - Persistencia", "ACTIVO"),
            new Curso(2L, "Módulo 6 - Frameworks", "ACTIVO")
        );
    }
}
```

**Controlador REST:**
```java
// src/main/java/com/edtech/catalog/infrastructure/rest/CursoController.java
package com.edtech.catalog.infrastructure.rest;

import com.edtech.catalog.application.ListarCursosService;
import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
public class CursoController {
    private final ListarCursosService listarCursosService;
    public CursoController(CursoRepositoryPort repositoryPort) {
        this.listarCursosService = new ListarCursosService(repositoryPort);
    }
    @GetMapping("/cursos")
    public List<Curso> listar() {
        return listarCursosService.ejecutar();
    }
}
```

**Dockerfile:**
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 4. Microservicio 2: lms-service (Node.js)

### 4.1. Crear el proyecto
```bash
mkdir lms-service
cd lms-service
npm init -y
npm install express
```

### 4.2. Código del servicio
```javascript
// lms-service/index.js
const express = require('express');
const app = express();
app.use(express.json());

const entregas = [];

// Endpoint para crear una entrega
app.post('/entregas', (req, res) => {
  const entrega = {
    id: entregas.length + 1,
    coderId: req.body.coderId,
    moduloId: req.body.moduloId,
    descripcion: req.body.descripcion
  };
  entregas.push(entrega);
  res.status(201).json(entrega);
});

// Endpoint para listar entregas
app.get('/entregas', (req, res) => {
  res.json(entregas);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`lms-service escuchando en puerto ${PORT}`);
});
```

**Dockerfile:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

---

## 5. Orquestación con docker-compose.yml

En la raíz `micro-edtech/` crea el archivo:
```yaml
version: '3.8'
services:
  catalog-service:
    build: ./catalog-service
    container_name: catalog-service
    ports:
      - "8080:8080"

  lms-service:
    build: ./lms-service
    container_name: lms-service
    ports:
      - "3000:3000"

  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - "3306:3306"
```

**Comentarios:**
- Cada servicio está en su propio contenedor y red interna.
- Puedes agregar variables de entorno y volúmenes según necesidad.

---

## 6. Levantar y probar los microservicios

### 6.1. Levantar todo
```bash
docker-compose up --build
```

### 6.2. Probar los endpoints
- Spring Boot: `GET http://localhost:8080/cursos`
- Node.js: `POST http://localhost:3000/entregas` con JSON:
  ```json
  {
    "coderId": 1,
    "moduloId": 2,
    "descripcion": "Mi entrega de prueba"
  }
  ```

### 6.3. Verificar comunicación
- Ambos servicios pueden comunicarse por red interna de Docker.
- Puedes agregar llamadas HTTP entre servicios usando la URL del servicio (ejemplo: `http://catalog-service:8080/cursos` desde Node).

---

## 7. Checklist y recomendaciones

- [x] Cada microservicio tiene su propio Dockerfile.
- [x] docker-compose.yml orquesta todos los servicios.
- [x] Prueba los endpoints con Postman/curl.
- [x] Documenta cada paso y agrega comentarios en el código.
- [x] Usa nombres claros y consistentes.

**Recomendaciones:**
- Mantén los servicios independientes y desacoplados.
- Usa variables de entorno para configuración sensible.
- Agrega tests unitarios y de integración.
- Documenta el flujo de datos y dependencias.

---

## 8. Preguntas Frecuentes (FAQ)

**¿Puedo agregar más servicios?** Sí, solo añade más bloques en docker-compose.yml.

**¿Cómo conecto los servicios a MySQL?** Configura la URL de conexión en cada microservicio usando el nombre del servicio `mysql` como host.

**¿Cómo escalo los servicios?** Usa la opción `scale` en Docker Compose o Kubernetes para producción.

---

**Autor:** Javier Ariza  
**Versión:** 2.0  
**Descripción:** Guía profesional, funcional y comentada para crear microservicios interoperables con Spring Boot, Node.js y Docker Compose.

---

## 0. Requisitos previos

- Tener **Docker** y **Docker Compose** instalados.
- Tener **Java 17** (o el que use tu Spring Boot) y **Maven**.
- Tener **Node.js** (v16+).
- Saber usar una terminal (cd, ls, etc.).
- Postman o curl para probar.

La idea es que al final puedas pararte frente a tus coders y decir: “esto que ven son **dos servicios distintos** escritos en lenguajes distintos, pero orquestados por Docker”.

---

## 1. Estructura del proyecto

Vamos a crear una carpeta de trabajo:

```text
micro-edtech/
  catalog-service/        <-- Spring Boot
  lms-service/            <-- Node.js
  docker-compose.yml
```

Trabajaremos dentro de `micro-edtech/`.

---

## 2. Microservicio 1 (Spring Boot): catalog-service

### 2.1. ¿Qué va a hacer?
Será el servicio que expone **cursos** o **módulos**. Por ahora solo tendrá:
- `GET /cursos` → devuelve una lista fija (luego la conectamos a MySQL)
- Arquitectura “tipo hexagonal”: tendremos dominio y adaptador REST claro.

### 2.2. Crear el proyecto
Puedes crear el proyecto desde https://start.spring.io (Spring Initializr) con estas dependencias:

- Spring Web
- Spring Data JPA
- MySQL Driver
- Lombok (opcional)

Nombre: `catalog-service`  
Group: `com.edtech`  
Package: `com.edtech.catalog`

Descarga el zip y mételo dentro de `micro-edtech/catalog-service`.

### 2.3. Dominio sencillo

```java
// src/main/java/com/edtech/catalog/domain/Curso.java
package com.edtech.catalog.domain;

public class Curso {
    private Long id;
    private String nombre;
    private String estado; // ACTIVO / INACTIVO

    public Curso(Long id, String nombre, String estado) {
        this.id = id;
        this.nombre = nombre;
        this.estado = estado;
    }

    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public String getEstado() { return estado; }
}
```

### 2.4. Puerto del dominio

```java
// src/main/java/com/edtech/catalog/domain/CursoRepositoryPort.java
package com.edtech.catalog.domain;

import java.util.List;

public interface CursoRepositoryPort {
    List<Curso> listarCursos();
}
```

### 2.5. Servicio de aplicación (caso de uso)

```java
// src/main/java/com/edtech/catalog/application/ListarCursosService.java
package com.edtech.catalog.application;

import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;

import java.util.List;

public class ListarCursosService {

    private final CursoRepositoryPort repository;

    public ListarCursosService(CursoRepositoryPort repository) {
        this.repository = repository;
    }

    public List<Curso> ejecutar() {
        return repository.listarCursos();
    }
}
```

### 2.6. Adaptador en memoria (para arrancar rápido)

```java
// src/main/java/com/edtech/catalog/infrastructure/InMemoryCursoAdapter.java
package com.edtech.catalog.infrastructure;

import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class InMemoryCursoAdapter implements CursoRepositoryPort {

    @Override
    public List<Curso> listarCursos() {
        return List.of(
                new Curso(1L, "Módulo 5 - Persistencia", "ACTIVO"),
                new Curso(2L, "Módulo 6 - Frameworks", "ACTIVO")
        );
    }
}
```

### 2.7. Controlador REST

```java
// src/main/java/com/edtech/catalog/infrastructure/rest/CursoController.java
package com.edtech.catalog.infrastructure.rest;

import com.edtech.catalog.application.ListarCursosService;
import com.edtech.catalog.domain.Curso;
import com.edtech.catalog.domain.CursoRepositoryPort;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class CursoController {

    private final ListarCursosService listarCursosService;

    public CursoController(CursoRepositoryPort repositoryPort) {
        this.listarCursosService = new ListarCursosService(repositoryPort);
    }

    @GetMapping("/cursos")
    public List<Curso> listar() {
        return listarCursosService.ejecutar();
    }
}
```

### 2.8. Dockerfile para el catalog-service

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 3. Microservicio 2 (Node.js): lms-service

### 3.1. Crear el proyecto

```bash
mkdir lms-service
cd lms-service
npm init -y
npm install express
```

### 3.2. Código del servicio

```javascript
// lms-service/index.js
const express = require('express');
const app = express();
app.use(express.json());

const entregas = [];

app.post('/entregas', (req, res) => {
  const entrega = {
    id: entregas.length + 1,
    coderId: req.body.coderId,
    moduloId: req.body.moduloId,
    descripcion: req.body.descripcion
  };
  entregas.push(entrega);
  res.status(201).json(entrega);
});

app.get('/entregas', (req, res) => {
  res.json(entregas);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`lms-service escuchando en puerto ${PORT}`);
});
```

### 3.3. Dockerfile para Node

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

---

## 4. docker-compose.yml

En la carpeta raíz `micro-edtech/`:

```yaml
version: '3.8'

services:
  catalog-service:
    build: ./catalog-service
    container_name: catalog-service
    ports:
      - "8080:8080"

  lms-service:
    build: ./lms-service
    container_name: lms-service
    ports:
      - "3000:3000"

  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - "3306:3306"
```

Levantar:

```bash
docker-compose up --build
```

Probar:
- `GET http://localhost:8080/cursos`
- `POST http://localhost:3000/entregas` con un JSON

---

## 5. Qué mostrarle a los estudiantes

1. Que son **dos proyectos distintos**.
2. Que Docker los pone en la misma red.
3. Que uno está en Java y otro en Node y no pasa nada.
4. Que luego puedes meter un gateway y seguridad.

---

Fin ✅