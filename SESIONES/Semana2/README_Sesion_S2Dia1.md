# Día 1 — Spring Boot: Autoconfiguración profunda, starters y perfiles (con ejemplos)

> En este material vamos **más allá de la teoría**: cada concepto trae **qué es**, **cómo funciona por dentro** y **un ejemplo mínimo** para que lo veas en acción. Todo preparado para integrarlo a tu ecosistema de **microservicios + Gateway** sin dependencias pagas.

---

## 0) Mapa mental de lo que ocurre al arrancar
1. Tu `main` corre `SpringApplication.run(...)`.  
2. Spring Boot crea el **ApplicationContext**, escanea **componentes** y carga **autoconfiguraciones**.  
3. Las **autoconfiguraciones** se **activan condicionalmente** (si hay clases en el **classpath**, si existen beans previos, o si hay propiedades).  
4. Se ensamblan **beans** finales y se levanta el **servidor embebido** (Tomcat por defecto).

---

## 1) ¿Qué hace Spring Boot realmente? (bajo la lupa)

### 1.1 Escaneo del classpath y autoconfiguraciones
- **Classpath**: es la lista de directorios y JARs que la JVM ve al ejecutar tu app (lo definen tu build y dependencias).  
- **Idea clave**: *“si está en el classpath, puedo activarlo”*. Por ejemplo, si está `jakarta.sql.DataSource`, Boot puede activar `DataSourceAutoConfiguration`.

**Condiciones típicas de autoconfiguración** (se evalúan antes de registrar beans):
- `@ConditionalOnClass(Foo.class)`: solo si la clase **existe** en el classpath.  
- `@ConditionalOnMissingBean(Bar.class)`: solo si **no** existe ya un bean `Bar`.  
- `@ConditionalOnProperty(prefix="x.y", name="enabled", havingValue="true")`: solo si una **propiedad** dice que “sí”.

**Ejemplo real** (fragmento conceptual):
```java
@Configuration
@ConditionalOnClass(javax.sql.DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
  @Bean @ConditionalOnMissingBean
  DataSource dataSource(DataSourceProperties props) { ... }
}
```

### 1.2 Starters: dependencias coherentes por “tema”
- Un **starter** es un *pom* que **agrupa dependencias compatibles** (no trae código Java).  
- Ej: `spring-boot-starter-web` trae Spring MVC, Jackson y validación.  
- Ej: `spring-boot-starter-data-jpa` trae Spring Data JPA + Hibernate + transacciones.

### 1.3 Servidor embebido
- Boot integra **Tomcat** (por defecto), opcional **Jetty** o **Undertow**.  
- Al detectar `spring-boot-starter-web`, **auto-registra** un `ServletWebServerFactory` y arranca en el puerto `8080`.

---

## 2) Anatomía de un starter (con ejemplo completo)

> Un starter “real” suele dividirse en 2 módulos: **autoconfigure** (código Java + condiciones) y **starter** (pom con dependencias). En Boot 3, las autoconfiguraciones se registran en el archivo `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### 2.1 Estructura mínima (multi-módulo)
```
my-hello-spring/
 ├─ my-hello-spring-autoconfigure/    # @Configuration + condiciones + @ConfigurationProperties
 │   └─ src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
 └─ my-hello-spring-starter/          # pom que depende del autoconfigure
```

### 2.2 Clase de propiedades tipadas
```java
// com.example.hello.HelloProperties
@ConfigurationProperties(prefix = "hello")
public class HelloProperties {
  /**
   * Mensaje que mostrará el bean HelloService.
   */
  private String message = "Hola por defecto";
  public String getMessage(){ return message; }
  public void setMessage(String m){ this.message = m; }
}
```

### 2.3 Auto-config (condicional)
```java
// com.example.hello.HelloAutoConfiguration
@Configuration
@EnableConfigurationProperties(HelloProperties.class)
@ConditionalOnClass(name = "org.springframework.context.ApplicationContext")
public class HelloAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public HelloService helloService(HelloProperties props) {
    return new HelloService(props.getMessage());
  }
}

// Servicio simple
public class HelloService {
  private final String msg;
  public HelloService(String msg){ this.msg = msg; }
  public String say(){ return msg; }
}
```

### 2.4 Registro de la autoconfiguración (Boot 3.x)
```
# my-hello-spring-autoconfigure/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.hello.HelloAutoConfiguration
```

### 2.5 El POM del starter
```xml
<!-- my-hello-spring-starter/pom.xml -->
<dependencies>
  <dependency>
    <groupId>com.example</groupId>
    <artifactId>my-hello-spring-autoconfigure</artifactId>
    <version>${project.version}</version>
  </dependency>
</dependencies>
```

**Uso en una app**:
```xml
<!-- app/pom.xml -->
<dependency>
  <groupId>com.example</groupId>
  <artifactId>my-hello-spring-starter</artifactId>
  <version>1.0.0</version>
</dependency>
```
```yaml
# application.yml
hello:
  message: "Hola desde starter!"
```
```java
// En cualquier @Component de tu app
@Autowired HelloService hello;
...
System.out.println(hello.say()); // → "Hola desde starter!"
```

---

## 3) Orden de resolución de propiedades (prioridad práctica)

De **mayor** a **menor** prioridad (resumen útil para el día a día):
1. **Argumentos de línea de comandos** (`--server.port=9090`).  
2. **Propiedades del sistema** (VM options: `-Dserver.port=9090`).  
3. **Variables de entorno** (`SERVER_PORT=9090`).  
4. **`application-{profile}.yml`** del perfil **activo** (por ej. `application-dev.yml`).  
5. **`application.yml`** (por defecto).  
6. **Valores por defecto** en `@ConfigurationProperties` o configuración interna.

> Cuando hay conflicto, **gana** el que esté más arriba. Esto permite que **prod** overridee **dev** sin tocar código.

---

## 4) Perfiles y configuración jerárquica (dev/test/prod)

### 4.1 Archivos por perfil
```
src/main/resources/
 ├─ application.yml
 ├─ application-dev.yml
 ├─ application-test.yml
 └─ application-prod.yml
```

### 4.2 Activar un perfil
- **VM Option**: `-Dspring.profiles.active=dev`  
- **Var de entorno**: `SPRING_PROFILES_ACTIVE=dev`

### 4.3 Ejemplo de jerarquía (DB + logs)
```yaml
# application.yml
spring:
  datasource:
    url: jdbc:h2:mem:demo
    username: sa
  jpa:
    hibernate:
      ddl-auto: none
logging:
  level:
    root: info

# application-dev.yml
spring:
  jpa:
    hibernate:
      ddl-auto: update
logging:
  level:
    org.springframework.web: debug

# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app
    username: ${DB_USER}
    password: ${DB_PASS} # ← **nunca** en texto plano; pásalo por env/secret manager
logging:
  level:
    root: warn
server:
  port: 8080
```

> **Regla de oro**: secretos **fuera** del repo; inyecta por variables de entorno o un gestor de secretos.

---

## 5) Autoconfiguración bajo la lupa (inspección y diagnóstico)

### 5.1 Actuator para ver beans y condiciones
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "beans,conditions,env,health,configprops"
```
- `GET /actuator/beans`: lista de beans finales (quién los define, tipo, dependencias).  
- `GET /actuator/conditions`: **por qué** una autoconfig **se aplicó o no** (tu “rayos X”).  
- `GET /actuator/configprops`: muestra `@ConfigurationProperties` enlazadas.

### 5.2 Log de condiciones (modo diagnóstico rápido)
```properties
# application.properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```
Al arrancar verás un reporte con autoconfiguraciones **positivas/negativas** y la **razón**.

### 5.3 Excluir/forzar autoconfiguraciones
```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
o
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class App { ... }
```

---

## 6) Buenas prácticas (con explicación)

- **No mezcles autoconfig con dominio**: el dominio debe ser puro; los beans “mágicos” viven en adaptadores/configuración.  
- **Explicita casos de uso con `@Bean` cuando tiene sentido**: si necesitas un `ObjectMapper` o un `RestClient` con customizaciones, declara el bean y deja que `@ConditionalOnMissingBean` de Boot lo respete.  
- **Mide tiempos de arranque**: habilita `info` o `debug` y observa fases. Si el classpath es enorme, revisa dependencias.  
- **Restringe `@ComponentScan`**: por defecto, arranca desde el paquete de tu clase `@SpringBootApplication`. Si tu monorepo es grande, **delimita** con `scanBasePackages` para evitar registrar cosas que no tocan.

Ejemplo de `@ComponentScan` acotado:
```java
@SpringBootApplication(scanBasePackages = {
  "com.riwi.academico.dto",
  "com.riwi.academico.infrastructure",
  "com.riwi.academico.application"
})
public class AcademicoApplication { ... }
```

---

## 7) Mini-lab: creando **tu propia** autoconfiguración

1. Crea un módulo `my-hello-spring-autoconfigure` con `HelloAutoConfiguration` y `HelloProperties`.  
2. Regístralo en `AutoConfiguration.imports`.  
3. Crea el **starter** (POM) que depende del autoconfigure.  
4. Úsalo desde una app demo y cambia `hello.message` en `application.yml`.  
5. Observa en `/actuator/beans` cómo aparece tu `HelloService`.  
6. Desactívalo con `spring.autoconfigure.exclude` y verifica el cambio en `/actuator/beans`.

---

## 8) Configuración completa en IntelliJ IDEA (paso a paso)

1. **Crear/abrir el proyecto**  
   - `File → New → Project…` (Spring Initializr para Boot).  
   - Selecciona **JDK 17** (Project Structure → Project SDK).

2. **Maven/Gradle (Auto-Import)**  
   - *Settings/Preferences* → **Build Tools** → Maven/Gradle.  
   - Activa **Auto-Import** y usa wrapper (Gradle/Maven).

3. **Plugins**  
   - `Settings → Plugins` → instala/activa **Lombok**, **SonarLint** y **Spring**.  
   - Reinicia si es necesario.

4. **Annotation Processing**  
   - `Build → Compiler → Annotation Processors` → **Enable annotation processing**.

5. **Run/Debug Configurations**  
   - `Run → Edit Configurations…` → *Spring Boot* o *Application*.  
   - **VM Options**: `-Dspring.profiles.active=dev`.  
   - **Environment variables**: `DB_URL`, `DB_USER`, `DB_PASS` (si aplican).  
   - **Working directory**: raíz del proyecto.

6. **Code Style / Inspections**  
   - `Editor → Code Style → Java`: organiza imports (`Ctrl+Alt+O` / `⌘⌥O`).  
   - `Editor → Inspections`: habilita inspecciones Spring/Java.

7. **Atajos clave**  
   - Buscar clase: `Ctrl+N` / `⌘O`  
   - Buscar símbolo: `Ctrl+Alt+Shift+N` / `⌘⌥O`  
   - Buscar en todo: `Ctrl+Shift+F` / `⌘⇧F`  
   - Ir a declaración/uso: `Ctrl+B` / `⌘B`, `Alt+F7` / `⌥F7`  
   - Estructura del archivo: `Ctrl+F12` / `⌘F12`

8. **Perfiles y Properties**  
   - Crea `application-dev.yml`, `application-test.yml`, `application-prod.yml`.  
   - Activa el perfil con `SPRING_PROFILES_ACTIVE`.  
   - **Pro-tip**: usa `@ConfigurationProperties` para config **tipada** y segura.

---

## 9) Cheatsheet de autoconfiguración (rápido y útil)

- **Ver por qué algo no se auto-configura**: `/actuator/conditions` o `logging.level.org.springframework.boot.autoconfigure=DEBUG`  
- **Sobrescribir un bean autoconfigurado**: declara tu propio `@Bean` del **mismo tipo** (Boot lo respeta por `@ConditionalOnMissingBean`).  
- **Apagar una autoconfig puntual**: `spring.autoconfigure.exclude=...`  
- **Cambiar el server**: incluye `spring-boot-starter-jetty` y **excluye** Tomcat en el POM del *starter web*.  
- **Puerto rápido**: `--server.port=9090` al ejecutar.

---

## 10) Verificación final (5 minutos)

1. Añade Actuator y expón `beans,conditions,configprops`.  
2. Arranca con `--debug` o con el logger `autoconfigure=DEBUG`.  
3. Cambia una propiedad en `application-dev.yml` y confirma en `/actuator/configprops`.  
4. Crea un `@Bean` que reemplace una autoconfig (ej. `ObjectMapper`) y verifica en `/actuator/beans` que ganó el tuyo.

---

## 11) Resultado esperado

- Entiendes **cómo** Boot decide qué configurar y **por qué**.  
- Sabes **crear** (y diagnosticar) una autoconfiguración propia.  
- Manejas **perfiles** y **prioridad** de propiedades con seguridad.  
- Tienes herramientas para **inspeccionar** y **depurar** el arranque de tu app.