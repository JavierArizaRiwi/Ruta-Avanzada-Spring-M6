# Guía: Plataforma Educativa con Microservicios + Arquitectura Hexagonal + Docker Compose (sin seguridad por ahora)

> Objetivo: mostrar a los aprendices cómo estructurar un sistema educativo realista usando **microservicios**, aplicando **arquitectura hexagonal** y orquestando todo con **docker-compose**, sin meter autenticación todavía para no distraer del modelo de servicios.

---

## 1. Contexto del escenario

Imagina que tu programa de formación (rutas, módulos, HU, pruebas de desempeño) quiere exponer sus datos para que un frontend (Angular, React o incluso Postman) pueda:

- Listar cursos/módulos activos.
- Registrar entregas de los coders.
- Consultar el estado de las entregas.
- (Opcional) disparar notificaciones cuando alguien entrega.

En lugar de hacer **un monolito**, lo dividimos en **varios microservicios**, cada uno con una responsabilidad clara y con su propio modelo de datos.

---

## 2. Microservicios propuestos

Vamos a usar 4 servicios básicos:

1. **api-gateway**  
   - Punto único de entrada.  
   - Solo enruta peticiones a los demás servicios.  
   - No tiene seguridad todavía.

2. **catalog-service**  
   - Administra **rutas, módulos y cursos**.  
   - Ejemplo: “Módulo 5 – Persistencia”, “Módulo 6 – Frameworks”, “Ruta Java + Spring”.

3. **lms-service**  
   - Administra **entregas, evaluaciones y estados** de los coders.  
   - Ejemplo: registrar que Javier entregó la HU de Semana 2.

4. **notification-service**  
   - Se encarga de procesar mensajes y “notificar”.  
   - Para ambiente educativo basta con que haga `println("notificación enviada")`.

Además, tendremos una **base de datos** compartida o varias, según lo quieras enseñar.

---

## 3. Por qué arquitectura hexagonal aquí

La arquitectura hexagonal (puertos y adaptadores) nos ayuda a que:

- El **dominio** (reglas de negocio) no dependa de Spring, ni de MySQL, ni de HTTP.
- Podamos cambiar el adaptador de persistencia (MySQL → Postgres) sin tocar el dominio.
- Podamos tener varios “drivers”: REST, eventos, línea de comandos.

La forma general de un servicio con hexagonal:

```text
/catalog-service
  /domain
    Curso.java
    Modulo.java
    CursoService.java
    ports/
      CursoRepositoryPort.java
  /application
    ListarCursosActivosUseCase.java
    CrearCursoUseCase.java
  /infrastructure
    /rest
      CursoController.java
    /persistence
      CursoJpaEntity.java
      SpringDataCursoRepository.java
      CursoJpaAdapter.java   <-- IMPLEMENTA el puerto
  Application.java
```

- **domain**: lo que la escuela hace.
- **application**: casos de uso concretos.
- **infrastructure**: todo lo que depende de frameworks.

---

## 4. Diseño funcional

### 4.1. Funcionalidades mínimas por servicio

**catalog-service**
- `GET /cursos` → lista todos los cursos.
- `GET /cursos/activos` → lista los activos.
- `POST /cursos` → crea un curso (por ahora sin seguridad).
- `GET /modulos?cursoId=...` → lista los módulos de un curso.

**lms-service**
- `POST /entregas` → registrar entrega de HU / prueba.
- `GET /entregas?coderId=...` → consultar lo entregado por un coder.
- `GET /estadisticas` → opcional para mostrar al mentor.

**notification-service**
- Podría exponer `POST /notify` para probarlo.
- O escuchar de una cola (RabbitMQ) cuando lms-service publica un evento.

**api-gateway**
- `GET /catalog/**` → redirige a catalog-service.
- `GET /lms/**` → redirige a lms-service.
- Esto puede ser un Spring Cloud Gateway, Traefik o Nginx simple.

---

## 5. docker-compose como orquestador

La idea es que los coders corran **todo** con un solo comando:

```bash
docker-compose up -d
```

y ya tengan:

- BD
- gateway
- services
- cola de mensajería (opcional)

### 5.1. Archivo `docker-compose.yml` de ejemplo

```yaml
version: '3.8'

services:
  api-gateway:
    image: mi-org/api-gateway:latest
    container_name: api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - catalog-service
      - lms-service
    environment:
      - CATALOG_SERVICE_URL=http://catalog-service:8082
      - LMS_SERVICE_URL=http://lms-service:8083

  catalog-service:
    image: mi-org/catalog-service:latest
    container_name: catalog-service
    ports:
      - "8082:8082"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/catalogdb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=root
    depends_on:
      - mysql

  lms-service:
    image: mi-org/lms-service:latest
    container_name: lms-service
    ports:
      - "8083:8083"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/lmsdb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=root
      - RABBITMQ_HOST=rabbitmq
    depends_on:
      - mysql
      - rabbitmq

  notification-service:
    image: mi-org/notification-service:latest
    container_name: notification-service
    depends_on:
      - rabbitmq

  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"

  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - "3306:3306"
```

**Notas para explicar en clase:**

- Todos los servicios están en la **misma red docker** (la crea compose).
- Por eso pueden llamarse por nombre: `http://catalog-service:8082`.
- Puedes usar **una BD por microservicio** (catalogdb, lmsdb) o una sola; tener varias refuerza el concepto de **bounded context**.
- No hay seguridad todavía, así que cualquier Postman puede consumir.

---

## 6. Ejemplo de controlador (catalog-service)

```java
@RestController
@RequestMapping("/cursos")
public class CursoController {

    private final ListarCursosActivosUseCase listarCursosActivosUseCase;
    private final CrearCursoUseCase crearCursoUseCase;

    public CursoController(ListarCursosActivosUseCase listarCursosActivosUseCase,
                           CrearCursoUseCase crearCursoUseCase) {
        this.listarCursosActivosUseCase = listarCursosActivosUseCase;
        this.crearCursoUseCase = crearCursoUseCase;
    }

    @GetMapping("/activos")
    public List<CursoDTO> listarActivos() {
        return listarCursosActivosUseCase.ejecutar();
    }

    @PostMapping
    public CursoDTO crear(@RequestBody CrearCursoCommand command) {
        return crearCursoUseCase.ejecutar(command);
    }
}
```

Esto es el **adaptador de entrada** (driving adapter): traduce HTTP → caso de uso.

---

## 7. Ejemplo de puerto + adaptador de persistencia

**Puerto (en el dominio):**

```java
public interface CursoRepositoryPort {
    Curso guardar(Curso curso);
    List<Curso> listarActivos();
}
```

**Adaptador (en infraestructura):**

```java
@Component
public class CursoJpaAdapter implements CursoRepositoryPort {

    private final SpringDataCursoRepository repository;

    public CursoJpaAdapter(SpringDataCursoRepository repository) {
        this.repository = repository;
    }

    @Override
    public Curso guardar(Curso curso) {
        CursoEntity entity = CursoEntity.fromDomain(curso);
        return repository.save(entity).toDomain();
    }

    @Override
    public List<Curso> listarActivos() {
        return repository.findByEstado("ACTIVO")
                .stream()
                .map(CursoEntity::toDomain)
                .toList();
    }
}
```

**Repositorio Spring Data (infra):**

```java
public interface SpringDataCursoRepository extends JpaRepository<CursoEntity, Long> {
    List<CursoEntity> findByEstado(String estado);
}
```

Con esto les queda clarísimo el concepto: **el dominio habla con un puerto, y la infraestructura implementa ese puerto**.

---

## 8. Flujo de uso (sin seguridad)

1. Levantas el compose:
   ```bash
   docker-compose up -d
   ```

2. Desde Postman llamas:
   ```http
   GET http://localhost:8080/catalog/cursos/activos
   ```
   - El gateway redirige a `catalog-service` en la red docker.
   - `catalog-service` consulta su BD.
   - Devuelve lista de cursos.

3. Registras una entrega:
   ```http
   POST http://localhost:8080/lms/entregas
   Content-Type: application/json

   {
     "coderId": "123",
     "moduloId": "M6",
     "tipo": "HU",
     "descripcion": "Entrega HU semana 2"
   }
   ```
   - El gateway enruta a `lms-service`.
   - `lms-service` guarda en su BD.
   - Opcional: publica evento a RabbitMQ → `notification-service` lo toma.

Todo esto sin tokens ni headers especiales.

---

## 9. Qué enseñar con este ejemplo

- **Desacoplamiento**: cada microservicio tiene su propio dominio.
- **Hexagonal**: puertos en el centro, adaptadores en el borde.
- **Orquestación**: docker-compose levanta todo junto.
- **Networking en Docker**: se consumen por nombre de servicio.
- **Evolución**: después se puede agregar seguridad (JWT) sin reescribir los casos de uso.

---

## 10. Próximos pasos (para otra guía)

- Agregar **Spring Cloud Gateway** real con `routes`.
- Agregar **seguridad** en el gateway solamente.
- Separar las bases en volúmenes.
- Meter observabilidad (Actuator) para que vean los endpoints de salud.

---

Fin de la guía ✅