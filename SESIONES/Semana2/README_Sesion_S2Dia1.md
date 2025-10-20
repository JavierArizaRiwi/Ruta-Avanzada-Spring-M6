# Día 1 — Spring Boot: Autoconfiguración profunda, starters y perfiles

## 1) Qué hace Spring Boot realmente
- Escanea el classpath y **activa autoconfiguraciones** condicionadas (ConditionalOnClass/Property).  
- Provee **starters** que agrupan dependencias compatibles.  
- Integra servidor embebido (Tomcat/Jetty/Undertow).

## 2) Anatomía de un starter
- `spring-boot-starter-web`: brings Spring MVC + Jackson + validation.  
- `spring-boot-starter-data-jpa`: JPA + Hibernate + transacciones.

## 3) Orden de resolución
1. `application.yml`/`application.properties`  
2. Variables env / VM Options  
3. `@ConfigurationProperties` (tipadas)

## 4) Profiles y configuración jerárquica
- Guidelines para separar dev/test/prod con `application-*.yml`.  
- Secretos fuera del repo; carga por entorno.

## 5) Autoconfiguración bajo la lupa
- `spring-boot-actuator` para inspeccionar beans y endpoints (`/actuator/beans`).  
- `conditions` log para diagnosticar por qué una autoconfig entra o no.

## 6) Buenas prácticas
- No abusar de autoconfig en el **dominio**.  
- Explicitar casos de uso con `@Bean`.  
- Medir tiempos de arranque; restringir `@ComponentScan`.


## 8) Configuración completa en IntelliJ IDEA

1. **Crear/abrir el proyecto**  
   - `File → New → Project…` (para Spring Boot usa Spring Initializr; para Spring Core selecciona Maven/Gradle).  
   - Selecciona **JDK 17** (Project Structure → Project SDK).

2. **Maven/Gradle (Auto-Import)**  
   - Settings/Preferences → *Build, Execution, Deployment* → **Build Tools** → *Maven/Gradle*.  
   - Activa **Auto-Import** y usa la opción recomendada (Gradle wrapper / Maven).

3. **Plugins**  
   - `Settings → Plugins` → instala o activa **Lombok**, **SonarLint** y **Spring**.  
   - Reinicia si lo solicita.

4. **Annotation Processing**  
   - `Build, Execution, Deployment → Compiler → Annotation Processors` → habilitar **Enable annotation processing**.

5. **Run/Debug Configurations**  
   - `Run → Edit Configurations…` → **+** → *Spring Boot* (si aplica) o *Application*.  
   - **VM Options**: `-Dspring.profiles.active=dev` (o el perfil del día).  
   - **Environment variables**: `DB_URL`, `DB_USER`, `DB_PASS` si se requieren.
   - **Working directory**: raíz del proyecto.

6. **Code Style / Inspections**  
   - `Editor → Code Style → Java`: organiza imports (usar `Ctrl+Alt+O` / `⌘⌥O`).  
   - `Editor → Inspections`: habilita inspecciones Spring/Java para detección temprana de problemas.

7. **Atajos clave**  
   - Buscar clase: `Ctrl+N` / `⌘O`  
   - Buscar símbolo (método/campo): `Ctrl+Alt+Shift+N` / `⌘⌥O`  
   - Buscar en todo: `Ctrl+Shift+F` / `⌘⇧F`  
   - Ir a declaración/uso: `Ctrl+B` / `⌘B`, `Alt+F7` / `⌥F7`  
   - Estructura del archivo: `Ctrl+F12` / `⌘F12`

8. **Perfiles y Properties**  
   - Crea `application-dev.yml`, `application-test.yml` y `application-prod.yml`.  
   - Activa el perfil desde VM Options o variable `SPRING_PROFILES_ACTIVE`.