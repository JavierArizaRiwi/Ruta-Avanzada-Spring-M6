#  Guía: Integración de Thymeleaf con Controladores REST (Spring Boot)

> **Objetivo:** Aprender a consumir y renderizar datos de controladores REST (`@RestController`) desde vistas Thymeleaf, utilizando `RestTemplate` o `WebClient` y plantillas HTML dinámicas.

---

##  Dependencias necesarias

Agrega las dependencias de **Thymeleaf** y del **starter web** en tu `pom.xml`:

```xml
<dependencies>
    <!-- Web y MVC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Thymeleaf -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>

    <!-- Validaciones y Documentación -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- (Opcional) Cliente HTTP reactivo -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
</dependencies>
```

---

##  Estructura del proyecto

```
src/main/java/com/riwitienda/pagos/
 ├─ web/controller/UsuarioController.java        # Controlador REST
 ├─ web/view/UsuarioViewController.java          # Controlador MVC para Thymeleaf
 ├─ services/impl/UsuarioServiceImpl.java
 ├─ web/dto/UsuarioRequest.java
 ├─ web/dto/UsuarioResponse.java
src/main/resources/
 ├─ templates/
 │   ├─ usuarios.html
 │   ├─ usuario-detalle.html
 └─ application.yml
```

---

##  Controlador MVC para Thymeleaf

Creamos un nuevo controlador que actuará como **cliente del controlador REST**, inyectando un `RestTemplate` o `WebClient` para consumir los endpoints del `UsuarioController`.

```java
package com.riwitienda.pagos.web.view;

import com.riwitienda.pagos.web.dto.UsuarioResponse;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.client.RestTemplate;
import java.util.List;

@Controller
public class UsuarioViewController {

    private final RestTemplate restTemplate;

    public UsuarioViewController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/usuarios")
    public String listarUsuarios(Model model) {
        String url = "http://localhost:8080/api/v1/usuario";
        List<UsuarioResponse> usuarios = List.of(
                restTemplate.getForObject(url, UsuarioResponse[].class)
        );
        model.addAttribute("usuarios", usuarios);
        return "usuarios"; // thymeleaf -> templates/usuarios.html
    }
}
```

###  Bean de RestTemplate
```java
@Configuration
public class WebConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

##  Plantilla Thymeleaf para listar usuarios

`src/main/resources/templates/usuarios.html`
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Lista de Usuarios</title>
    <link rel="stylesheet" th:href="@{/css/styles.css}">
</head>
<body>
<h1>Usuarios registrados</h1>

<table border="1">
    <thead>
        <tr>
            <th>ID</th>
            <th>Nombre</th>
            <th>Email</th>
        </tr>
    </thead>
    <tbody>
        <tr th:each="usuario : ${usuarios}">
            <td th:text="${usuario.id}"></td>
            <td th:text="${usuario.nombre}"></td>
            <td th:text="${usuario.email}"></td>
        </tr>
    </tbody>
</table>

<a th:href="@{/usuarios/nuevo}">Crear nuevo usuario</a>
</body>
</html>
```

---

##  Crear un formulario para registrar usuarios

`src/main/resources/templates/usuario-nuevo.html`
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Nuevo Usuario</title>
</head>
<body>
<h2>Registrar nuevo usuario</h2>

<form th:action="@{/usuarios}" th:object="${usuario}" method="post">
  <label>Nombre:</label>
  <input type="text" th:field="*{nombre}" required><br>
  
  <label>Email:</label>
  <input type="email" th:field="*{email}" required><br>
  
  <button type="submit">Guardar</button>
</form>

<a th:href="@{/usuarios}">Volver</a>
</body>
</html>
```

---

##  Enviar datos desde Thymeleaf hacia el endpoint REST

Agregamos un método `@PostMapping` en el `UsuarioViewController` para enviar los datos al `UsuarioController` original (`/api/v1/usuario`):

```java
@PostMapping("/usuarios")
public String crearUsuario(@ModelAttribute UsuarioRequest usuario) {
    String url = "http://localhost:8080/api/v1/usuario";
    restTemplate.postForObject(url, usuario, UsuarioResponse.class);
    return "redirect:/usuarios";
}
```

---

##  Configuración de Thymeleaf en `application.yml`

```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    cache: false
    mode: HTML
  mvc:
    view:
      prefix: /templates/
      suffix: .html
server:
  port: 8080
```

---

##  Ejecución y flujo de prueba

1. Inicia tu aplicación (`SpringApplication.run`).
2. Accede a `http://localhost:8080/usuarios`.
3. Verás la lista de usuarios cargada desde el endpoint REST.
4. Clic en “Crear nuevo usuario” → completa el formulario → guarda.
5. Se envía un `POST` al endpoint `/api/v1/usuario` y se redirige al listado actualizado.

---

##  Resultado final

✅ **Controlador REST** (API JSON)  
✅ **Controlador MVC** (renderiza vistas HTML)  
✅ **Consumo de endpoints vía `RestTemplate`**  
✅ **Plantillas Thymeleaf interactivas**

---

##  Extensiones recomendadas

- `Thymeleaf Layout Dialect` → para layouts base (`header`, `footer`, etc.).
- `Spring Boot DevTools` → recarga automática.
- `Bootstrap` → para estilizar tablas y formularios fácilmente.