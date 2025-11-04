# Guía Profesional de Thymeleaf en Spring Boot

## Introducción

Thymeleaf es un motor de plantillas moderno para Java, ampliamente utilizado en aplicaciones web con Spring Boot. Permite crear vistas HTML dinámicas, integrando datos del backend de forma segura y eficiente. Su sintaxis es intuitiva y se integra perfectamente con el modelo MVC de Spring.

---

## Integración con Spring Boot

### Dependencias necesarias

Incluye en tu `pom.xml`:

```xml
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## Controladores y Modelo de Datos

### Uso de `@ModelAttribute`

`@ModelAttribute` se utiliza para vincular datos entre el backend y la vista. Puede aplicarse a métodos o parámetros:

#### A nivel de método

```java
@ModelAttribute("usuario")
public Usuario crearUsuario() {
        return new Usuario();
}
```
Este método se ejecuta antes de cada petición y añade el objeto `usuario` al modelo.

#### A nivel de parámetro

```java
@PostMapping("/usuarios")
public String guardarUsuario(@ModelAttribute Usuario usuario) {
        // Procesa el usuario recibido del formulario
        return "usuarios";
}
```
Permite recibir datos de formularios directamente en el objeto.

---

## Directivas Principales de Thymeleaf

- **th:text**: Inserta texto escapado.
    ```html
    <span th:text="${usuario.nombre}"></span>
    ```
- **th:each**: Itera sobre colecciones.
    ```html
    <tr th:each="usuario : ${usuarios}">
            <td th:text="${usuario.nombre}"></td>
    </tr>
    ```
- **th:if / th:unless**: Condicionales.
    ```html
    <div th:if="${usuario.activo}">Activo</div>
    ```
- **th:href / th:src**: Enlaces y recursos.
    ```html
    <a th:href="@{/usuarios/{id}(id=${usuario.id})}">Ver</a>
    <img th:src="@{/images/logo.png}" />
    ```
- **th:object / th:field**: Formularios vinculados a objetos.
    ```html
    <form th:object="${usuario}" th:action="@{/usuarios}" method="post">
            <input th:field="*{nombre}" />
    </form>
    ```
- **th:value**: Valor de campos.
    ```html
    <input th:value="${usuario.email}" />
    ```
- **th:attr**: Atributos personalizados.
    ```html
    <input th:attr="placeholder=${usuario.nombre}" />
    ```
- **th:replace / th:include**: Fragmentos y layouts.
    ```html
    <div th:replace="fragments/header :: header"></div>
    ```
- **th:switch / th:case**: Estructuras de control.
    ```html
    <div th:switch="${usuario.rol}">
            <span th:case="'ADMIN'">Administrador</span>
            <span th:case="'USER'">Usuario</span>
    </div>
    ```

---

## Ejemplo de Formulario y Validación

```html
<form th:object="${usuario}" th:action="@{/usuarios}" method="post">
        <input th:field="*{nombre}" placeholder="Nombre" />
        <span th:if="${#fields.hasErrors('nombre')}" th:errors="*{nombre}"></span>
        <input th:field="*{email}" placeholder="Email" />
        <span th:if="${#fields.hasErrors('email')}" th:errors="*{email}"></span>
        <button type="submit">Guardar</button>
</form>
```

En el controlador:

```java
@PostMapping("/usuarios")
public String guardarUsuario(@Valid @ModelAttribute Usuario usuario, BindingResult result) {
        if (result.hasErrors()) {
                return "formulario-usuario";
        }
        // Guardar usuario
        return "redirect:/usuarios";
}
```

---

## Fragmentos y Layouts

Thymeleaf permite reutilizar partes de la vista mediante fragmentos:

```html
<!-- fragments/header.html -->
<div th:fragment="header">
        <h1>Mi Aplicación</h1>
</div>
```

En la plantilla principal:

```html
<div th:replace="fragments/header :: header"></div>
```

---

## Internacionalización (i18n)

Configura archivos de mensajes en `src/main/resources/messages.properties`:

```properties
label.nombre=Nombre
label.email=Correo electrónico
```

En la vista:

```html
<label th:text="#{label.nombre}"></label>
```

---

## Buenas Prácticas

- Utiliza fragmentos para layouts reutilizables.
- Escapa siempre el texto con `th:text` para evitar XSS.
- Valida los datos en el backend y muestra errores en la vista.
- Mantén la lógica de negocio fuera de las plantillas.
- Usa internacionalización para soportar múltiples idiomas.

---

¿Dudas o quieres ejemplos específicos? ¡Solicítalos!
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