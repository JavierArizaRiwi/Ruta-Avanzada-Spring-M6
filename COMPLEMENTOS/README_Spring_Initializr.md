Initializr# Guía Detallada sobre Spring Initializr

## Introducción

**Spring Initializr** es una herramienta oficial de Spring que te
permite **generar un proyecto base de Spring Boot** con solo seleccionar
las dependencias, el lenguaje y la versión de Java. Es el punto de
partida más común para crear aplicaciones Spring modernas.

Puedes acceder a ella desde: - 🌐 <https://start.spring.io> - O
directamente desde **IntelliJ IDEA**, **Spring Tools Suite (STS)** o
**VS Code**.

------------------------------------------------------------------------

## 1. ¿Qué es Spring Initializr?

Spring Initializr automatiza la creación del esqueleto de un proyecto
Spring Boot. Te evita configurar manualmente el `pom.xml` o
`build.gradle`, la estructura de carpetas y la configuración base.

Es ideal para crear proyectos con: - Microservicios - APIs REST -
Aplicaciones con bases de datos (H2, MySQL, PostgreSQL, MongoDB) -
Aplicaciones con seguridad (Spring Security) - Aplicaciones web (Spring
MVC o WebFlux)

------------------------------------------------------------------------

## 2. Cómo acceder a Spring Initializr

### Opción 1: Desde el navegador

1.  Abre <https://start.spring.io>
2.  Configura las opciones según tu proyecto.
3.  Haz clic en **"Generate"**.
4.  Se descargará un `.zip` con el proyecto listo para importar.

### Opción 2: Desde IntelliJ IDEA

1.  Abre IntelliJ.
2.  Selecciona **New Project → Spring Initializr**.
3.  IntelliJ conecta automáticamente con `https://start.spring.io`.
4.  Configura el proyecto y haz clic en **Finish**.

------------------------------------------------------------------------

## 3. Parámetros principales del proyecto

  ------------------------------------------------------------------------
  Campo             Descripción                     Ejemplo
  ----------------- ------------------------------- ----------------------
  **Project**       Sistema de construcción: Maven  Maven Project
                    o Gradle                        

  **Language**      Lenguaje de programación        Java

  **Spring Boot     Versión del framework           3.3.3
  Version**                                         

  **Group**         Nombre del grupo base del       com.codeup
                    paquete                         

  **Artifact**      Nombre principal del proyecto   academico-spring
                    (jar generado)                  

  **Name**          Nombre del proyecto             academico-spring

  **Description**   Descripción corta del proyecto  Sistema académico con
                                                    Spring Boot

  **Package Name**  Paquete raíz del código fuente  com.codeup.academico

  **Packaging**     Tipo de empaquetado: Jar o War  Jar

  **Java Version**  Versión del JDK                 17
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 4. Dependencias más comunes

Dependencias que puedes seleccionar en Spring Initializr:

  ------------------------------------------------------------------------
  Categoría              Dependencia              Descripción
  ---------------------- ------------------------ ------------------------
  **Core**               Spring Boot DevTools     Recarga automática
                                                  durante el desarrollo

  **Web**                Spring Web               Permite crear APIs REST
                                                  y aplicaciones MVC

  **Data**               Spring Data JPA          ORM para interactuar con
                                                  bases de datos

  **Database**           H2 Database              Base de datos en memoria
                                                  para pruebas

  **Database**           MySQL Driver             Conector para bases de
                                                  datos MySQL

  **Template Engines**   Thymeleaf                Motor de plantillas HTML

  **Security**           Spring Security          Seguridad y
                                                  autenticación

  **Tools**              Lombok                   Reduce código repetitivo
                                                  (getters, setters, etc.)

  **Cloud**              Spring Cloud Config      Configuración
                                                  distribuida

  **Messaging**          Spring Kafka             Integración con colas
                                                  Kafka
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 5. Estructura generada

Cuando descargas el proyecto, obtendrás una estructura como esta:

    academico-spring/
     ├─ src/main/java/com/codeup/academico/
     │   └─ AcademicoSpringApplication.java
     ├─ src/main/resources/
     │   ├─ application.properties
     │   └─ static/ (para recursos estáticos como CSS/JS)
     │   └─ templates/ (para vistas Thymeleaf)
     ├─ src/test/java/
     ├─ pom.xml

### Explicación

-   **src/main/java:** Contiene tu código fuente principal.
-   **src/main/resources:** Configuración, propiedades y archivos
    estáticos.
-   **src/test/java:** Pruebas unitarias e integradas.
-   **pom.xml:** Configuración del proyecto, dependencias y plugins.

------------------------------------------------------------------------

## 6. Archivo `pom.xml` básico

Ejemplo generado automáticamente por Spring Initializr:

``` xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.codeup</groupId>
    <artifactId>academico-spring</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>academico-spring</name>
    <description>Sistema académico con Spring Boot</description>
    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

------------------------------------------------------------------------

## 7. Ejecución del proyecto

``` bash
# Compilar
mvn clean install

# Ejecutar
mvn spring-boot:run

# O empaquetar y ejecutar el jar
java -jar target/academico-spring-0.0.1-SNAPSHOT.jar
```

El proyecto se ejecutará por defecto en `http://localhost:8080`.

------------------------------------------------------------------------

## 8. Configuración adicional (application.properties)

``` properties
server.port=8080
spring.application.name=academico-spring
spring.h2.console.enabled=true
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.username=sa
spring.jpa.hibernate.ddl-auto=update
logging.level.org.hibernate.SQL=debug
```

------------------------------------------------------------------------

## 9. Consejos para nuevos proyectos Spring

1.  **Usa H2** al inicio para evitar configuraciones complejas de base
    de datos.\
2.  **Activa DevTools** para reinicio automático.\
3.  **Crea paquetes separados:** `controller`, `service`, `repository`,
    `domain`.\
4.  **Usa Lombok** para simplificar POJOs.\
5.  **Agrega pruebas básicas** con `@SpringBootTest`.\
6.  **Usa perfiles** (`application-dev.properties`,
    `application-prod.properties`) para ambientes distintos.\
7.  **Versiona tu proyecto con Git** desde el inicio.

------------------------------------------------------------------------

## 10. Recursos útiles

-   📘 [Documentación oficial de Spring
    Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/)
-   🧩 [Guías de Spring Boot](https://spring.io/guides)
-   ⚙️ [Spring Initializr
    GitHub](https://github.com/spring-io/initializr)
-   🎓 [Curso gratuito de Spring en
    spring.io/guides](https://spring.io/guides)

------------------------------------------------------------------------

**Autor:** Javier Junior Ariza Montenegro

**Versión:** 1.0

**Licencia:** MIT