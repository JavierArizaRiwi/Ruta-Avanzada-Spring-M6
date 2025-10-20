# Día 4 — Introducción a los Servicios Web y Servidores de Aplicaciones

En este módulo entenderás **qué son los servicios web**, por qué surgen en el mundo Java y **por qué necesitamos un servidor de aplicaciones** para desplegarlos.  
Este conocimiento marca la transición natural desde tus programas en **Java SE (de escritorio o consola)** hacia **Java empresarial (Spring, Jakarta EE, microservicios, etc.)**.

---

## 1) Contexto: ¿Qué hacíamos en Java SE?

Hasta ahora has construido aplicaciones que:
- Se ejecutan en una sola máquina (local).
- Usan consola o Swing como interfaz.
- Guardan datos en archivos o bases de datos locales con JDBC.

Estas aplicaciones son **autónomas**: el flujo inicia con un `main()` y termina cuando el usuario sale.

```java
public class App {
    public static void main(String[] args) {
        System.out.println("Sistema Académico iniciado...");
    }
}
```

Pero… ¿qué pasa cuando varios usuarios necesitan acceder **al mismo sistema** desde distintas ubicaciones o dispositivos?

Necesitamos **comunicación remota**, **distribución de servicios** y **procesamiento concurrente**.  
Ahí nacen los **servicios web**.

---

## 2) ¿Qué es un servicio web?

Un **servicio web** es una aplicación que **expone funcionalidades a través de la red (HTTP)** para ser consumidas por otros programas (no solo personas).

**Ejemplo de idea:**
Tu sistema académico tiene un módulo para registrar estudiantes.  
En lugar de usar consola, quieres que otras aplicaciones (un front en Angular o una app móvil) puedan consumirlo vía HTTP:

```
POST /api/estudiantes
{
  "id": "1",
  "nombre": "Ana Pérez"
}
```

Tu backend responde con JSON:
```
201 Created
{
  "mensaje": "Estudiante creado correctamente"
}
```

**Conclusión:** un servicio web convierte tu lógica Java en **endpoints HTTP** que otros pueden usar.

---

## 3) Tipos principales de servicios web

| Tipo | Descripción | Ejemplo |
|------|--------------|----------|
| **SOAP (XML)** | Basado en XML y contratos WSDL; usado en sistemas corporativos legados. | WS en Java EE con JAX-WS |
| **REST (JSON)** | Ligero, usa HTTP estándar, trabaja con JSON o XML. | Spring Boot, JAX-RS |
| **GraphQL** | Lenguaje de consultas moderno; el cliente define qué datos quiere. | Integraciones modernas con microservicios |

Hoy en día, **REST** es el enfoque más usado en arquitecturas modernas.

---

## 4) ¿Por qué necesitamos un servidor de aplicaciones?

Un programa Java SE se ejecuta con un `main()`.  
Pero un servicio web **debe permanecer escuchando peticiones HTTP**.  
Esto requiere un **servidor** que:

- Abra un **puerto TCP/IP** (por ejemplo, `localhost:8080`).
- Reciba y despache solicitudes (requests) a tus controladores.
- Administre **hilos, seguridad, sesiones y contexto web**.

### Ejemplo visual

```
Cliente (Navegador / App Móvil)
        ↓  HTTP
Servidor de Aplicaciones (Tomcat, Jetty, WildFly)
        ↓
Tu código Java (controladores REST, servicios, repositorios)
        ↓
Base de datos / otros servicios
```

El servidor actúa como **intérprete entre el mundo HTTP y tus clases Java**.

---

## 5) Tipos de servidores en el ecosistema Java

| Tipo | Ejemplos | Características |
|------|-----------|----------------|
| **Servidor web** | Apache HTTP Server, Nginx | Sirven contenido estático (HTML, imágenes, JS). |
| **Servidor de aplicaciones Java EE/Jakarta EE** | WildFly, Payara, GlassFish, JBoss | Ejecutan aplicaciones empresariales completas (EJB, JPA, JAX-RS, JMS). |
| **Servlet Containers (ligeros)** | Tomcat, Jetty, Undertow | Ejecutan aplicaciones web basadas en Servlets o Spring MVC. |

**Spring Boot** integra un servidor embebido (normalmente **Tomcat**) para evitar configuraciones externas.  
Por eso puedes ejecutar tu aplicación web simplemente con:
```bash
mvn spring-boot:run
```
y acceder a `http://localhost:8080`.

---

## 6) Cómo funciona una aplicación web en un servidor Java

### Ciclo básico de una solicitud HTTP

1. El cliente hace una petición (`GET /api/estudiantes`).
2. El servidor de aplicaciones la recibe en el **Servlet Dispatcher**.
3. Spring redirige la petición al controlador correcto.
4. El método del controlador ejecuta la lógica (servicios/repositorios).
5. Se construye una respuesta (`ResponseEntity`) con el resultado.
6. El servidor devuelve la respuesta HTTP al cliente.

```java
@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {

    private final RegistrarEstudianteUseCase registrar;

    public EstudianteController(RegistrarEstudianteUseCase registrar) {
        this.registrar = registrar;
    }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@RequestBody Estudiante e) {
        Estudiante creado = registrar.ejecutar(e.getId(), e.getNombre());
        return ResponseEntity.ok(creado);
    }
}
```

Al ejecutar el proyecto, Spring Boot levanta **Tomcat embebido**, escucha en el puerto 8080 y procesa las solicitudes.

---

## 7) Diferencias clave: Aplicación Java SE vs Web

| Característica | Java SE | Aplicación Web con Spring |
|----------------|----------|----------------------------|
| Punto de entrada | `main()` | Servidor de aplicaciones (Servlet Container) |
| Ejecución | Local | Escucha peticiones HTTP |
| Interacción | Usuario en consola | Cliente HTTP (navegador, app) |
| Ciclo de vida | Termina al ejecutar main | Persistente mientras el servidor esté activo |
| Concurrencia | Hilos manejados manualmente | Pool de hilos manejado por el servidor |

---

## 8) Cómo desplegar una aplicación web

1. **Empaquetar la aplicación:**  
   Maven o Gradle generan un archivo `.war` (Web Application Archive) o `.jar` ejecutable.

   ```bash
   mvn clean package
   ```

2. **Elegir un servidor:**
   - `.war` → despliegas en Tomcat, WildFly, Payara, etc.
   - `.jar` → ejecutas directamente con Spring Boot.

3. **Ejecutar:**
   ```bash
   java -jar academico-0.0.1-SNAPSHOT.jar
   ```

4. **Acceder en navegador:**
   ```
   http://localhost:8080/api/estudiantes
   ```

---

## 9) Ventajas de usar un servidor de aplicaciones

- Permite **desarrollo modular** (web, lógica, datos).  
- Facilita el **despliegue centralizado** (un servidor, múltiples clientes).  
- Soporta **concurrencia y seguridad** de forma estándar.  
- Integra componentes empresariales (transacciones, colas, mensajería).  
- Escala horizontalmente con balanceadores y contenedores (Docker/Kubernetes).

---

## 10) Conclusión

Pasar de Java SE a un entorno web implica un cambio de paradigma:  
- El código deja de ser “local” y pasa a vivir en un servidor.  
- Se responde a peticiones HTTP, no a entradas de usuario por consola.  
- Aparecen nuevos conceptos (endpoints, JSON, API, contexto de aplicación).  

Este paso es la base para trabajar con **Spring MVC**, **Spring Boot** y luego **microservicios**, donde cada módulo puede actuar como un **servicio independiente desplegado en la nube**.

---

**Próximo paso:**  
En el siguiente día aprenderás cómo **crear tu primer servicio REST en Spring Boot**, estructurar endpoints y consumirlos desde Postman o Angular.