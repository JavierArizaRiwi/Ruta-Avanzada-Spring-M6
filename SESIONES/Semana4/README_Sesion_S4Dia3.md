
# Día 3 – Spring Security con JWT: Autenticación, Autorización por Roles y Arquitectura Limpia

En esta sesión aprenderás a proteger tus endpoints REST con **Spring Security** usando **JWT (JSON Web Tokens)**, manteniendo el diseño en **arquitectura limpia/hexagonal**. Verás cómo separar responsabilidades: autenticación, emisión y validación del token, y cómo aplicar **roles** y **permisos** con anotaciones.

---

## 1) Objetivos del día

- Configurar **Spring Security** sin `WebSecurityConfigurerAdapter` (enfoque moderno con `SecurityFilterChain`).  
- Implementar **autenticación** con **JWT** (login, emisión de token, validación).  
- Añadir **autorización** basada en **roles** y **anotaciones** (`@PreAuthorize`).  
- Separar capas: adaptadores de entrada (REST), dominio (usuarios/roles), adaptadores de salida (persistencia), utilidades de seguridad.  
- Configurar **CORS/CSRF** y políticas seguras.  
- Probar seguridad con **MockMvc**.

---

## 2) Dependencias necesarias (`pom.xml`)

```xml
<dependencies>
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
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## 3) Propiedades recomendadas (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:secdb;DB_CLOSE_DELAY=-1
    username: sa
    password: 
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true

app:
  security:
    jwt:
      secret: "cambia-esta-clave-secreta-mas-larga-y-aleatoria"
      expiration-minutes: 60

logging:
  level:
    org.springframework.security: info
```

---

## 4) Estructura de proyecto (enfoque limpio/hexagonal)

```
com.riwi.academico
 ├─ domain/
 │   ├─ model/               # Usuario, Rol, Permisos (sin dependencias de Spring)
 │   ├─ ports/               # UserRepositoryPort, AuthServicePort
 │   └─ service/             # Casos de uso (autenticación, registro, verificación)
 ├─ infrastructure/
 │   ├─ adapters/
 │   │   ├─ in/
 │   │   │   ├─ AuthController.java   # /auth/login, /auth/register
 │   │   │   └─ EstudianteController.java  # Endpoints protegidos
 │   │   └─ out/
 │   │       ├─ jpa/                   # Entidades JPA y repositorios Spring Data
 │   │       └─ security/              # JwtTokenProvider, filtros, SecurityConfig
 │   └─ config/                        # Beans y configuración
 └─ dto/                               # LoginRequest, TokenResponse, etc.
```

---

## 5) Modelo de dominio (simplificado)

```java
// domain/model/Role.java
package com.riwi.academico.domain.model;

public enum Role {
    ADMIN, PROFESOR, ESTUDIANTE
}
```

```java
// domain/model/Usuario.java
package com.riwi.academico.domain.model;

public class Usuario {
    private Long id;
    private String username;
    private String password; // encriptado
    private Role role;

    public Usuario(Long id, String username, String password, Role role){
        this.id = id; this.username = username; this.password = password; this.role = role;
    }
    public Long getId(){ return id; }
    public String getUsername(){ return username; }
    public String getPassword(){ return password; }
    public Role getRole(){ return role; }
}
```

Puertos del dominio:

```java
// domain/ports/UserRepositoryPort.java
package com.riwi.academico.domain.ports;

import com.riwi.academico.domain.model.Usuario;
import java.util.Optional;

public interface UserRepositoryPort {
    Optional<Usuario> findByUsername(String username);
    Usuario save(Usuario user);
}
```

```java
// domain/ports/AuthServicePort.java
package com.riwi.academico.domain.ports;

public interface AuthServicePort {
    String login(String username, String rawPassword);
    void register(String username, String rawPassword, String roleName);
}
```

---

## 6) Adaptadores JPA (salida)

```java
// infrastructure/adapters/out/jpa/UserEntity.java
package com.riwi.academico.infrastructure.adapters.out.jpa;

import jakarta.persistence.*;

@Entity @Table(name="usuarios")
public class UserEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(unique = true, nullable = false) private String username;
    @Column(nullable = false) private String password;
    @Column(nullable = false) private String role;
    public Long getId(){ return id; }
    public String getUsername(){ return username; }
    public String getPassword(){ return password; }
    public String getRole(){ return role; }
    public void setUsername(String u){ this.username = u; }
    public void setPassword(String p){ this.password = p; }
    public void setRole(String r){ this.role = r; }
}
```

```java
// infrastructure/adapters/out/jpa/SpringUserRepository.java
package com.riwi.academico.infrastructure.adapters.out.jpa;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface SpringUserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByUsername(String username);
}
```

```java
// infrastructure/adapters/out/jpa/UserRepositoryAdapter.java
package com.riwi.academico.infrastructure.adapters.out.jpa;

import com.riwi.academico.domain.model.Role;
import com.riwi.academico.domain.model.Usuario;
import com.riwi.academico.domain.ports.UserRepositoryPort;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public class UserRepositoryAdapter implements UserRepositoryPort {

    private final SpringUserRepository repo;

    public UserRepositoryAdapter(SpringUserRepository repo) { this.repo = repo; }

    @Override
    public Optional<Usuario> findByUsername(String username) {
        return repo.findByUsername(username)
                .map(e -> new Usuario(e.getId(), e.getUsername(), e.getPassword(), Role.valueOf(e.getRole())));
    }

    @Override
    public Usuario save(Usuario u) {
        UserEntity e = new UserEntity();
        e.setUsername(u.getUsername());
        e.setPassword(u.getPassword());
        e.setRole(u.getRole().name());
        UserEntity saved = repo.save(e);
        return new Usuario(saved.getId(), saved.getUsername(), saved.getPassword(), Role.valueOf(saved.getRole()));
    }
}
```

---

## 7) Utilidades JWT y configuración de seguridad

### 7.1 JwtTokenProvider

```java
// infrastructure/adapters/out/security/JwtTokenProvider.java
package com.riwi.academico.infrastructure.adapters.out.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.security.Key;
import java.util.Date;

@Component
public class JwtTokenProvider {

    private final Key key;
    private final long expirationMs;

    public JwtTokenProvider(
            @Value("${app.security.jwt.secret}") String secret,
            @Value("${app.security.jwt.expiration-minutes}") long expirationMinutes) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes());
        this.expirationMs = expirationMinutes * 60_000;
    }

    public String generate(String username, String role) {
        Date now = new Date();
        Date exp = new Date(now.getTime() + expirationMs);
        return Jwts.builder()
                .setSubject(username)
                .claim("role", role)
                .setIssuedAt(now)
                .setExpiration(exp)
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }

    public Jws<Claims> validate(String token) {
        return Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
    }

    public String getUsername(String token) { return validate(token).getBody().getSubject(); }
    public String getRole(String token) { return (String) validate(token).getBody().get("role"); }
}
```

### 7.2 Filtro JWT

```java
// infrastructure/adapters/out/security/JwtAuthFilter.java
package com.riwi.academico.infrastructure.adapters.out.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtTokenProvider provider;

    public JwtAuthFilter(JwtTokenProvider provider) {
        this.provider = provider;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                String username = provider.getUsername(token);
                String role = provider.getRole(token);
                var auth = new UsernamePasswordAuthenticationToken(
                        username, null, List.of(new SimpleGrantedAuthority("ROLE_" + role)));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (Exception ex) {
                SecurityContextHolder.clearContext();
            }
        }
        chain.doFilter(request, response);
    }
}
```

### 7.3 SecurityFilterChain y PasswordEncoder

```java
// infrastructure/adapters/out/security/SecurityConfig.java
package com.riwi.academico.infrastructure.adapters.out.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableMethodSecurity // habilita @PreAuthorize
public class SecurityConfig {

    private final JwtTokenProvider provider;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtTokenProvider provider, UserDetailsService uds) {
        this.provider = provider;
        this.userDetailsService = uds;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
           .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
           .authorizeHttpRequests(auth -> auth
               .requestMatchers("/auth/**").permitAll()
               .requestMatchers(HttpMethod.GET, "/api/estudiantes/**").hasAnyRole("ADMIN","PROFESOR","ESTUDIANTE")
               .requestMatchers("/api/**").hasAnyRole("ADMIN","PROFESOR")
               .anyRequest().authenticated()
           )
           .addFilterBefore(new JwtAuthFilter(provider), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return new ProviderManager(provider);
    }
}
```

---

## 8) UserDetailsService (adaptador de lectura de usuarios)

```java
// infrastructure/adapters/out/security/CustomUserDetailsService.java
package com.riwi.academico.infrastructure.adapters.out.security;

import com.riwi.academico.domain.ports.UserRepositoryPort;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepositoryPort users;

    public CustomUserDetailsService(UserRepositoryPort users){ this.users = users; }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        var u = users.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Usuario no encontrado"));
        return User.withUsername(u.getUsername())
                .password(u.getPassword())
                .roles(u.getRole().name())
                .build();
    }
}
```

---

## 9) Casos de uso y controlador de autenticación

```java
// domain/service/AuthService.java
package com.riwi.academico.domain.service;

import com.riwi.academico.domain.model.Role;
import com.riwi.academico.domain.model.Usuario;
import com.riwi.academico.domain.ports.AuthServicePort;
import com.riwi.academico.domain.ports.UserRepositoryPort;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class AuthService implements AuthServicePort {

    private final UserRepositoryPort users;
    private final PasswordEncoder encoder;
    private final com.riwi.academico.infrastructure.adapters.out.security.JwtTokenProvider jwt;

    public AuthService(UserRepositoryPort users, PasswordEncoder encoder,
                       com.riwi.academico.infrastructure.adapters.out.security.JwtTokenProvider jwt) {
        this.users = users; this.encoder = encoder; this.jwt = jwt;
    }

    @Override
    public String login(String username, String rawPassword) {
        var u = users.findByUsername(username).orElseThrow(() -> new RuntimeException("Credenciales inválidas"));
        if (!encoder.matches(rawPassword, u.getPassword())) throw new RuntimeException("Credenciales inválidas");
        return jwt.generate(u.getUsername(), u.getRole().name());
    }

    @Override
    public void register(String username, String rawPassword, String roleName) {
        String enc = encoder.encode(rawPassword);
        users.save(new Usuario(null, username, enc, Role.valueOf(roleName)));
    }
}
```

DTOs y controlador:

```java
// dto/LoginRequest.java
package com.riwi.academico.dto;

import jakarta.validation.constraints.NotBlank;
public class LoginRequest {
    @NotBlank private String username;
    @NotBlank private String password;
    public String getUsername(){ return username; }
    public void setUsername(String u){ this.username = u; }
    public String getPassword(){ return password; }
    public void setPassword(String p){ this.password = p; }
}
```

```java
// dto/TokenResponse.java
package com.riwi.academico.dto;

public class TokenResponse {
    private String token;
    public TokenResponse(String token){ this.token = token; }
    public String getToken(){ return token; }
}
```

```java
// infrastructure/adapters/in/AuthController.java
package com.riwi.academico.infrastructure.adapters.in;

import com.riwi.academico.domain.ports.AuthServicePort;
import com.riwi.academico.dto.LoginRequest;
import com.riwi.academico.dto.TokenResponse;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthServicePort auth;

    public AuthController(AuthServicePort auth){ this.auth = auth; }

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest req){
        String token = auth.login(req.getUsername(), req.getPassword());
        return ResponseEntity.ok(new TokenResponse(token));
    }

    @PostMapping("/register")
    public ResponseEntity<Void> register(@RequestParam String username,
                                         @RequestParam String password,
                                         @RequestParam String role){
        auth.register(username, password, role);
        return ResponseEntity.ok().build();
    }
}
```

---

## 10) Autorización con anotaciones

En el controlador o en el caso de uso puedes usar `@PreAuthorize`:

```java
// infrastructure/adapters/in/EstudianteController.java
@PreAuthorize("hasRole('ADMIN') or hasRole('PROFESOR')")
@PostMapping
public ResponseEntity<EstudianteResponse> crear(@Valid @RequestBody EstudianteRequest req) { ... }

@PreAuthorize("hasAnyRole('ADMIN','PROFESOR','ESTUDIANTE')")
@GetMapping
public ResponseEntity<List<EstudianteResponse>> listar() { ... }
```

Asegúrate de tener `@EnableMethodSecurity` en `SecurityConfig`.

---

## 11) CORS y CSRF

- Para APIs REST con JWT, **CSRF suele deshabilitarse** porque no hay cookies de sesión.  
- Habilita **CORS** solo para orígenes confiables si hay un frontend externo.

```java
http.csrf(csrf -> csrf.disable())
    .cors(cors -> {});
```

En `application.yml` puedes definir orígenes permitidos y registrarlos con un `CorsConfigurationSource` si lo necesitas.

---

## 12) Pruebas de seguridad con MockMvc

```java
@WebMvcTest(AuthController.class)
class AuthControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private AuthServicePort auth;

    @Test
    void debeRetornarTokenAlLoguear() throws Exception {
        Mockito.when(auth.login("admin", "123")).thenReturn("jwt-token");

        mvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"admin\",\"password\":\"123\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.token").value("jwt-token"));
    }
}
```

Para endpoints protegidos, agrega el header `Authorization: Bearer <token>` en la solicitud.

---

## 13) Configuración en IntelliJ IDEA

1. Plugins: Spring Boot, Lombok, SonarLint.  
2. Configurar perfiles: `-Dspring.profiles.active=dev`.  
3. Variables sensibles: no hardcodear secretos; usar variables de entorno (`APP_SECURITY_JWT_SECRET`) y mapear con `@Value`.  
4. Atajos útiles:  
   - Buscar clases: `Ctrl+N` / `⌘O`  
   - Ejecutar pruebas: `Ctrl+Shift+F10` / `⌘⇧R`  
   - Depurar filtros: breakpoints en `JwtAuthFilter`.  

---

## 14) Buenas prácticas

| Práctica | Beneficio |
|----------|-----------|
| Encriptar contraseñas con BCrypt | Seguridad de credenciales |
| Tokens con expiración corta | Reduce superficie de ataque |
| No guardar JWT en BD | Stateless, escalable |
| Rotación de secretos | Mitiga fugas |
| `@PreAuthorize` en casos de uso sensibles | Defensa por capas |
| Centralizar excepciones de auth | Respuestas consistentes |

---

## 15) Resultado esperado

- API protegida con **Spring Security + JWT**.  
- Endpoints públicos `/auth/**` y privados `/api/**`.  
- Autorización por roles con `@PreAuthorize`.  
- Diseño en **arquitectura limpia**: dominio independiente, adaptadores y utilidades de seguridad separadas.  
- Pruebas de autenticación con MockMvc exitosas.