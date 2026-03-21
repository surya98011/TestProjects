# 🔐 Module 9: Spring Security — JWT & OAuth 2.0

> All examples use the **ShopEase** e-commerce microservice project (user-service).

---

## 📑 Table of Contents

- [1. What is Spring Security?](#1-what-is-spring-security)
- [2. Basic Authentication](#2-basic-authentication)
- [3. Securing Specific URLs](#3-securing-specific-urls)
- [4. Database Authentication](#4-database-authentication)
- [5. JWT (JSON Web Tokens)](#5-jwt-json-web-tokens)
- [6. OAuth 2.0](#6-oauth-20)
- [7. Microservices Security with JWT](#7-microservices-security-with-jwt)

---

## 1. What is Spring Security?

| Concept | Purpose |
|---|---|
| **Authentication** | Verifying WHO the user is (credentials check) |
| **Authorization** | Verifying WHAT the user can access (permissions) |

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

> Adding this starter **automatically secures ALL endpoints** with a random password (printed on console).

---

## 2. Basic Authentication

### Override Default Credentials

```yaml
# application.yml
spring:
  security:
    user:
      name: shopease-admin
      password: admin@123
```

---

## 3. Securing Specific URLs

### ShopEase: Some APIs public, some secured

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                // ✅ Public endpoints — no auth needed
                .requestMatchers("/api/users/register", "/api/users/login").permitAll()
                .requestMatchers("/api/products/**").permitAll()
                .requestMatchers("/swagger-ui/**").permitAll()
                // ✅ Admin only
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                // ✅ Everything else requires authentication
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

---

## 4. Database Authentication

### ShopEase User Entity

```java
@Entity
@Table(name = "users")
@Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;    // BCrypt encoded
    private String email;
    private String role;        // ADMIN / USER
}
```

### UserRepository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

### Custom UserDetailsService

```java
@Service
public class ShopEaseUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepo.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User
                .withUsername(user.getUsername())
                .password(user.getPassword())
                .authorities(user.getRole())
                .build();
    }
}
```

### Security Config with DB Auth

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private ShopEaseUserDetailsService userDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http.csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/users/register", "/api/users/login").permitAll()
                    .anyRequest().authenticated()
                )
                .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authenticationProvider(authenticationProvider())
                .build();
    }
}
```

### Registration & Login Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired private UserRepository userRepo;
    @Autowired private PasswordEncoder passwordEncoder;
    @Autowired private AuthenticationManager authManager;
    @Autowired private JwtService jwtService;

    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody User user) {
        // ✅ Encode password before saving
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        user.setRole("USER");
        userRepo.save(user);
        return ResponseEntity.ok("User registered successfully");
    }

    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {
        UsernamePasswordAuthenticationToken token =
                new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword());
        try {
            Authentication auth = authManager.authenticate(token);
            if (auth.isAuthenticated()) {
                // ✅ Generate JWT token
                String jwt = jwtService.generateToken(request.getUsername());
                return ResponseEntity.ok(jwt);
            }
        } catch (Exception e) {
            // log error
        }
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid Credentials");
    }
}
```

---

## 5. JWT (JSON Web Tokens)

### JWT Structure

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJzdXJ5YSIsImlhdCI6MTcxfQ.abc123signature
|_____HEADER_____|    |________PAYLOAD_________|   |__SIGNATURE__|
```

| Part | Contains |
|---|---|
| **Header** | Algorithm (HS256), Type (JWT) |
| **Payload** | Subject (username), Issued At, Expiry |
| **Signature** | HMAC(header + payload, SECRET_KEY) |

### JwtService.java

```java
@Service
public class JwtService {

    private static final String SECRET = "ShopEaseSecretKeyForJWT2024VeryLongSecure";
    private static final long EXPIRATION = 1000 * 60 * 60; // 1 hour

    public String generateToken(String username) {
        return Jwts.builder()
                .setSubject(username)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION))
                .signWith(getSignKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public boolean validateToken(String token, String username) {
        String tokenUsername = extractUsername(token);
        return tokenUsername.equals(username) && !isTokenExpired(token);
    }

    public String extractUsername(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSignKey()).build()
                .parseClaimsJws(token).getBody().getSubject();
    }

    private boolean isTokenExpired(String token) {
        Date expiry = Jwts.parserBuilder()
                .setSigningKey(getSignKey()).build()
                .parseClaimsJws(token).getBody().getExpiration();
        return expiry.before(new Date());
    }

    private Key getSignKey() {
        return Keys.hmacShaKeyFor(SECRET.getBytes());
    }
}
```

### JWT Filter (Validates token on every request)

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private ShopEaseUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            String username = jwtService.extractUsername(token);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.validateToken(token, userDetails.getUsername())) {
                    UsernamePasswordAuthenticationToken authToken =
                            new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

### How to Use JWT with Postman

```
1. POST /api/users/register → { "username": "surya", "password": "pass123", "email": "surya@test.com" }
2. POST /api/users/login    → { "username": "surya", "password": "pass123" }
   ← Response: "eyJhbGciOiJIUzI1Ni..."

3. GET /api/orders (Protected endpoint)
   Header:  Authorization = Bearer eyJhbGciOiJIUzI1Ni...
```

---

## 6. OAuth 2.0

### Login with GitHub

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            clientId: your-github-client-id
            clientSecret: your-github-client-secret
```

> Create OAuth App: GitHub → Settings → Developer Settings → OAuth Apps → New

---

## 7. Microservices Security with JWT

```
    Client                API Gateway              Microservices
      │                       │                         │
      │── POST /login ──────▶│── route to ──▶ User-Service
      │                       │                    │ (generate JWT)
      │◀── JWT Token ────────│◀─── token ────────│
      │                       │                         │
      │── GET /api/products ─▶│                         │
      │   + Bearer Token      │── Validate JWT ──▶      │
      │                       │── Route to ────▶ Product-Service
      │◀── Product data ─────│◀── response ────────────│
```

> **Key Insight:** User-service generates JWT. API Gateway validates JWT in its filter. Backend microservices trust the gateway and focus only on business logic.

---

*← [08 — Microservices](./08-microservices-architecture.md) | [10 — Apache Kafka →](./10-apache-kafka.md)*
