# 02 — Spring Boot (IoC/DI, Auto-configuration, Security, Testing, Actuator)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: IoC & Dependency Injection

### 🔥 Q1. Explain IoC and Dependency Injection. What are the types of DI in Spring?

**Answer:**  
**Inversion of Control (IoC)**: The framework controls object creation and lifecycle, not the developer. The Spring IoC container (ApplicationContext) manages beans.

**Dependency Injection (DI)**: A specific form of IoC where dependencies are provided (injected) to objects rather than objects creating them.

**Types of DI:**

| Type | Mechanism | Recommended? |
|------|-----------|-------------|
| **Constructor Injection** | Dependencies via constructor | ✅ Yes (immutable, testable) |
| **Setter Injection** | Dependencies via setter methods | For optional dependencies |
| **Field Injection** | `@Autowired` on fields | ❌ No (hard to test, hides dependencies) |

```java
// ✅ Constructor Injection (Best Practice)
@Service
public class PaymentService {
    private final PaymentGateway gateway;
    private final TransactionRepository repository;
    private final NotificationService notificationService;
    
    // @Autowired is optional with single constructor (Spring 4.3+)
    public PaymentService(PaymentGateway gateway, 
                          TransactionRepository repository,
                          NotificationService notificationService) {
        this.gateway = gateway;
        this.repository = repository;
        this.notificationService = notificationService;
    }
}

// ❌ Field Injection (Avoid)
@Service
public class PaymentService {
    @Autowired private PaymentGateway gateway;  // Can't make final, hard to test
}
```

**Why Constructor Injection is preferred:**
1. Fields can be `final` → immutable, thread-safe
2. Easy to unit test (just pass mocks via constructor)
3. Makes dependencies explicit
4. Fails fast at startup if dependency is missing

---

### 🔥 Q2. What is the difference between @Component, @Service, @Repository, and @Controller?

**Answer:**  
All are **stereotype annotations** that mark a class as a Spring-managed bean. They are all specializations of `@Component`.

| Annotation | Layer | Special Behavior |
|-----------|-------|-----------------|
| `@Component` | Generic | Base annotation, no special behavior |
| `@Service` | Business Logic | Semantic only (no extra behavior) |
| `@Repository` | Data Access | Enables **exception translation** (converts DB exceptions to Spring's `DataAccessException`) |
| `@Controller` | Web/MVC | Enables request mapping, view resolution |
| `@RestController` | REST API | `@Controller` + `@ResponseBody` on every method |

```java
@Repository  // Enables PersistenceExceptionTranslationPostProcessor
public class PaymentRepositoryImpl implements PaymentRepository {
    @PersistenceContext
    private EntityManager em;
    
    public Payment findByTxnId(String txnId) {
        // If a JPA/JDBC exception occurs, it's translated to DataAccessException
        return em.createQuery("SELECT p FROM Payment p WHERE p.txnId = :txnId", Payment.class)
            .setParameter("txnId", txnId)
            .getSingleResult();
    }
}
```

---

### 💎 Q3. Explain Bean Scopes in Spring. What happens with a prototype bean inside a singleton?

**Answer:**

| Scope | Lifecycle | Use Case |
|-------|-----------|----------|
| `singleton` (default) | One instance per ApplicationContext | Stateless services |
| `prototype` | New instance every time requested | Stateful beans |
| `request` | One per HTTP request | Request-scoped data |
| `session` | One per HTTP session | User session data |
| `application` | One per ServletContext | App-wide config |

**The Prototype-in-Singleton Problem:**

```java
@Service // Singleton scope (default)
public class PaymentProcessor {
    @Autowired
    private PaymentContext context; // Prototype scope
    
    // ❌ PROBLEM: context is injected ONCE at startup
    // Every call uses the SAME instance — prototype behavior is lost!
    public void process(Payment payment) {
        context.setPayment(payment); // Shared state! Race condition!
    }
}
```

**Solutions:**

```java
// Solution 1: ObjectFactory / ObjectProvider
@Service
public class PaymentProcessor {
    @Autowired
    private ObjectProvider<PaymentContext> contextProvider;
    
    public void process(Payment payment) {
        PaymentContext context = contextProvider.getObject(); // New instance each time
        context.setPayment(payment);
    }
}

// Solution 2: @Lookup method injection
@Service
public abstract class PaymentProcessor {
    @Lookup
    protected abstract PaymentContext createContext(); // Spring overrides this
    
    public void process(Payment payment) {
        PaymentContext context = createContext(); // New instance each time
        context.setPayment(payment);
    }
}

// Solution 3: Scoped proxy
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class PaymentContext { ... }
```

---

## Section 2: Auto-configuration & Starters

### 🔥 Q4. How does Spring Boot Auto-configuration work internally?

**Answer:**

Auto-configuration automatically configures beans based on classpath dependencies, existing beans, and properties.

**Flow:**

```
@SpringBootApplication
    ├── @EnableAutoConfiguration
    │       ├── @Import(AutoConfigurationImportSelector.class)
    │       │       └── Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    │       │           (or META-INF/spring.factories in older versions)
    │       │       └── Returns list of auto-configuration classes
    │       │
    │       └── Each auto-config class uses:
    │           ├── @ConditionalOnClass — Only if class is on classpath
    │           ├── @ConditionalOnMissingBean — Only if user hasn't defined their own
    │           ├── @ConditionalOnProperty — Only if property is set
    │           └── @ConditionalOnWebApplication — Only in web context
    │
    ├── @ComponentScan — Scans current package + sub-packages
    └── @SpringBootConfiguration — Marks as configuration class
```

**Example: How DataSource auto-config works:**
1. Spring Boot sees `HikariCP` on classpath → `@ConditionalOnClass(HikariDataSource.class)` matches
2. Checks no user-defined `DataSource` bean → `@ConditionalOnMissingBean(DataSource.class)` matches
3. Reads `spring.datasource.*` properties
4. Creates and configures `HikariDataSource` bean automatically

```java
// Simplified version of what Spring Boot does internally
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    }
}
```

**To see what's auto-configured:**
```bash
# Debug auto-configuration report
java -jar app.jar --debug
# Or in application.yml
debug: true
# Check: CONDITIONS EVALUATION REPORT in logs
```

---

### 💎 Q5. How do you create a custom Spring Boot Starter?

**Answer:**

A starter is a module that bundles auto-configuration + dependencies for a specific feature.

**Structure:**
```
my-payment-spring-boot-starter/
├── pom.xml (dependencies)
└── src/main/java/
    └── com/company/payment/autoconfigure/
        ├── PaymentAutoConfiguration.java
        ├── PaymentProperties.java
        └── PaymentClient.java
└── src/main/resources/
    └── META-INF/
        └── spring/
            └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```java
// PaymentProperties.java
@ConfigurationProperties(prefix = "payment.gateway")
public class PaymentProperties {
    private String apiUrl = "https://api.gateway.com";
    private String apiKey;
    private Duration timeout = Duration.ofSeconds(30);
    private int maxRetries = 3;
    // getters, setters
}

// PaymentAutoConfiguration.java
@AutoConfiguration
@ConditionalOnClass(PaymentClient.class)
@EnableConfigurationProperties(PaymentProperties.class)
public class PaymentAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public PaymentClient paymentClient(PaymentProperties props) {
        return new PaymentClient(props.getApiUrl(), props.getApiKey(), 
                                  props.getTimeout(), props.getMaxRetries());
    }
}

// Register in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.company.payment.autoconfigure.PaymentAutoConfiguration
```

**Usage by consumers:**
```yaml
# application.yml
payment:
  gateway:
    api-url: https://api.razorpay.com
    api-key: ${PAYMENT_API_KEY}
    timeout: 15s
    max-retries: 2
```

---

## Section 3: Spring Security

### 🔥 Q6. Explain Spring Security filter chain. How does JWT authentication work?

**Answer:**

Spring Security is implemented as a **servlet filter chain** that intercepts every HTTP request.

**Filter Chain Order:**
```
HTTP Request
    │
    ▼
SecurityFilterChain
    ├── 1. CorsFilter
    ├── 2. CsrfFilter
    ├── 3. UsernamePasswordAuthenticationFilter (form login)
    ├── 4. BearerTokenAuthenticationFilter (OAuth2/JWT)
    ├── 5. BasicAuthenticationFilter
    ├── 6. ExceptionTranslationFilter
    └── 7. AuthorizationFilter (formerly FilterSecurityInterceptor)
    │
    ▼
DispatcherServlet → Controller
```

**JWT Authentication Implementation:**

```java
// SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Disable for stateless API
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/payments/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, authEx) -> {
                    res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    res.getWriter().write("{\"error\": \"Unauthorized\"}");
                })
            )
            .build();
    }
}

// JwtAuthenticationFilter.java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain chain) throws ServletException, IOException {
        String token = extractToken(request);
        
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            
            UsernamePasswordAuthenticationToken auth = 
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        chain.doFilter(request, response);
    }
    
    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}

// JwtTokenProvider.java
@Component
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration:3600000}") // 1 hour
    private long expiration;
    
    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(Keys.hmacShaKeyFor(secret.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder().setSigningKey(secret.getBytes()).build().parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    public String getUsername(String token) {
        return Jwts.parserBuilder().setSigningKey(secret.getBytes()).build()
            .parseClaimsJws(token).getBody().getSubject();
    }
}
```

---

### 💎 Q7. Explain OAuth2 flows. Which one is used for microservices?

**Answer:**

| Flow | Use Case | Client Type |
|------|----------|-------------|
| **Authorization Code** | Web apps with backend | Confidential |
| **Authorization Code + PKCE** | SPAs, mobile apps | Public |
| **Client Credentials** | Service-to-service (M2M) | Confidential |
| **Resource Owner Password** | Legacy (deprecated) | Trusted |
| **Device Code** | Smart TVs, CLI tools | Public |

**For Microservices → Client Credentials Flow:**

```
┌──────────────┐     1. POST /oauth/token          ┌──────────────┐
│ Payment      │     (client_id + client_secret)    │ Auth Server  │
│ Service      │ ──────────────────────────────────→│ (Keycloak/   │
│              │     2. Access Token (JWT)           │  Okta)       │
│              │ ←──────────────────────────────────│              │
└──────┬───────┘                                    └──────────────┘
       │
       │ 3. API call with Bearer token
       ▼
┌──────────────┐
│ Order        │  4. Validates JWT (signature + claims + expiry)
│ Service      │
└──────────────┘
```

```java
// Service-to-service call with Client Credentials
@Configuration
public class OAuth2ClientConfig {
    
    @Bean
    public WebClient orderServiceClient(
            ReactiveClientRegistrationRepository clientRegistrations,
            ServerOAuth2AuthorizedClientRepository authorizedClients) {
        
        ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServerOAuth2AuthorizedClientExchangeFilterFunction(
                clientRegistrations, authorizedClients);
        oauth2.setDefaultClientRegistrationId("order-service");
        
        return WebClient.builder()
            .filter(oauth2)
            .baseUrl("http://order-service:8080")
            .build();
    }
}

// application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          order-service:
            client-id: payment-service
            client-secret: ${ORDER_SERVICE_SECRET}
            authorization-grant-type: client_credentials
            scope: orders.read,orders.write
        provider:
          order-service:
            token-uri: http://keycloak:8080/realms/shopease/protocol/openid-connect/token
```

---

## Section 4: Exception Handling & Validation

### 🔥 Q8. How do you implement global exception handling in Spring Boot?

**Answer:**

Use `@RestControllerAdvice` (combines `@ControllerAdvice` + `@ResponseBody`).

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // Standard error response
    public record ErrorResponse(
        String errorCode,
        String message,
        String path,
        Instant timestamp,
        Map<String, String> fieldErrors
    ) {}
    
    @ExceptionHandler(PaymentNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(PaymentNotFoundException ex, HttpServletRequest request) {
        log.warn("Payment not found: {}", ex.getMessage());
        return new ErrorResponse("PAYMENT_NOT_FOUND", ex.getMessage(), 
            request.getRequestURI(), Instant.now(), null);
    }
    
    @ExceptionHandler(DuplicatePaymentException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDuplicate(DuplicatePaymentException ex, HttpServletRequest request) {
        log.warn("Duplicate payment: {}", ex.getMessage());
        return new ErrorResponse("DUPLICATE_PAYMENT", ex.getMessage(),
            request.getRequestURI(), Instant.now(), null);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid value",
                (a, b) -> a // merge function for duplicate keys
            ));
        return new ErrorResponse("VALIDATION_FAILED", "Request validation failed",
            request.getRequestURI(), Instant.now(), fieldErrors);
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}", request.getRequestURI(), ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred",
            request.getRequestURI(), Instant.now(), null);
    }
}
```

**Validation with Bean Validation:**

```java
public record CreatePaymentRequest(
    @NotBlank(message = "Merchant ID is required")
    String merchantId,
    
    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be greater than 0")
    @Digits(integer = 10, fraction = 2, message = "Amount format invalid")
    BigDecimal amount,
    
    @NotNull(message = "Currency is required")
    Currency currency,
    
    @NotBlank(message = "Idempotency key is required")
    @Size(min = 16, max = 64, message = "Idempotency key must be 16-64 characters")
    String idempotencyKey
) {}

@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {
    
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @Valid @RequestBody CreatePaymentRequest request) {
        // If validation fails, MethodArgumentNotValidException is thrown
        // and caught by GlobalExceptionHandler
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(paymentService.create(request));
    }
}
```

---

## Section 5: Actuator & Profiles

### 🔥 Q9. What is Spring Boot Actuator? How do you use it in production?

**Answer:**

Actuator provides **production-ready features**: health checks, metrics, info, environment details.

**Key Endpoints:**

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Application health (DB, disk, custom) |
| `/actuator/metrics` | Micrometer metrics (JVM, HTTP, custom) |
| `/actuator/info` | Build info, git info |
| `/actuator/env` | Environment properties |
| `/actuator/loggers` | View/change log levels at runtime |
| `/actuator/prometheus` | Prometheus-format metrics |

```yaml
# application.yml — Production configuration
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, info, prometheus, loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
  health:
    db:
      enabled: true
    diskspace:
      enabled: true
  metrics:
    tags:
      application: payment-service
      environment: ${SPRING_PROFILES_ACTIVE:default}
    export:
      prometheus:
        enabled: true
```

**Custom Health Indicator:**

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    
    private final PaymentGatewayClient gatewayClient;
    
    @Override
    public Health health() {
        try {
            boolean isUp = gatewayClient.ping();
            if (isUp) {
                return Health.up()
                    .withDetail("gateway", "Razorpay")
                    .withDetail("latency", gatewayClient.getLastPingLatency() + "ms")
                    .build();
            }
            return Health.down()
                .withDetail("gateway", "Razorpay")
                .withDetail("error", "Ping failed")
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

// Custom Metrics
@Service
public class PaymentService {
    private final MeterRegistry meterRegistry;
    private final Counter paymentSuccessCounter;
    private final Counter paymentFailureCounter;
    private final Timer paymentProcessingTimer;
    
    public PaymentService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.paymentSuccessCounter = Counter.builder("payments.processed")
            .tag("status", "success").register(meterRegistry);
        this.paymentFailureCounter = Counter.builder("payments.processed")
            .tag("status", "failure").register(meterRegistry);
        this.paymentProcessingTimer = Timer.builder("payments.processing.time")
            .register(meterRegistry);
    }
    
    public PaymentResult process(PaymentRequest request) {
        return paymentProcessingTimer.record(() -> {
            try {
                PaymentResult result = gateway.charge(request);
                paymentSuccessCounter.increment();
                return result;
            } catch (Exception e) {
                paymentFailureCounter.increment();
                throw e;
            }
        });
    }
}
```

---

### 🔥 Q10. How do Spring Profiles work? How do you manage environment-specific configs?

**Answer:**

Profiles allow different configurations for different environments (dev, staging, prod).

**Activation:**
```bash
# Command line
java -jar app.jar --spring.profiles.active=prod

# Environment variable
SPRING_PROFILES_ACTIVE=prod

# application.yml
spring:
  profiles:
    active: dev
```

**Profile-specific files:**
```
application.yml          # Common/default config
application-dev.yml      # Dev overrides
application-staging.yml  # Staging overrides
application-prod.yml     # Production overrides
```

```yaml
# application.yml (common)
spring:
  application:
    name: payment-service
server:
  port: 8080

---
# application-dev.yml
spring:
  datasource:
    url: jdbc:oracle:thin:@localhost:1521:XEPDB1
    username: dev_user
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
logging:
  level:
    com.shopease: DEBUG

---
# application-prod.yml
spring:
  datasource:
    url: jdbc:oracle:thin:@prod-db-cluster:1521/PAYMENTDB
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: none
logging:
  level:
    com.shopease: INFO
    root: WARN
```

**Conditional Beans:**
```java
@Configuration
public class GatewayConfig {
    
    @Bean
    @Profile("dev")
    public PaymentGateway mockGateway() {
        return new MockPaymentGateway(); // Fake gateway for dev
    }
    
    @Bean
    @Profile("prod")
    public PaymentGateway razorpayGateway(PaymentProperties props) {
        return new RazorpayGateway(props.getApiKey(), props.getSecret());
    }
}
```

---

## Section 6: Filters, Interceptors & AOP

### 🔥 Q11. What is the difference between Filter, Interceptor, and AOP?

**Answer:**

```
HTTP Request
    │
    ▼
┌─────────────────┐
│  Servlet Filter  │  ← Lowest level, works on request/response
│  (javax.servlet) │     Before DispatcherServlet
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DispatcherServlet│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  HandlerInter-   │  ← Spring MVC level
│  ceptor          │     Before/After controller method
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  AOP Aspect      │  ← Method level
│  (@Around, etc.) │     Any Spring bean method
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Controller      │
└─────────────────┘
```

| Feature | Filter | Interceptor | AOP |
|---------|--------|-------------|-----|
| Level | Servlet | Spring MVC | Method |
| Access to | Request/Response | Handler, ModelAndView | JoinPoint, method args |
| Spring beans | Limited | Full access | Full access |
| Use case | CORS, logging, auth | Request logging, locale | Transactions, caching, auditing |

```java
// Filter: Request/Response logging
@Component
@Order(1)
public class RequestLoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId);
        
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("method={} uri={} status={} duration={}ms correlationId={}", 
                request.getMethod(), request.getRequestURI(), 
                response.getStatus(), duration, correlationId);
            MDC.clear();
        }
    }
}

// AOP: Audit logging for payment operations
@Aspect
@Component
@Slf4j
public class PaymentAuditAspect {
    
    @Around("@annotation(auditable)")
    public Object audit(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        String method = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        log.info("AUDIT: Starting {} with args: {}", method, Arrays.toString(args));
        
        try {
            Object result = joinPoint.proceed();
            log.info("AUDIT: Completed {} successfully", method);
            return result;
        } catch (Exception e) {
            log.error("AUDIT: Failed {} with error: {}", method, e.getMessage());
            throw e;
        }
    }
}
```

---

## Section 7: Testing

### 🔥 Q12. How do you test a Spring Boot application? Explain the testing pyramid.

**Answer:**

**Testing Pyramid:**
```
        /\
       /  \        E2E Tests (few)
      /    \       - @SpringBootTest + TestRestTemplate
     /──────\
    /        \     Integration Tests (moderate)
   /          \    - @WebMvcTest, @DataJpaTest, @SpringBootTest
  /────────────\
 /              \  Unit Tests (many)
/                \ - JUnit 5 + Mockito, no Spring context
──────────────────
```

| Test Type | Annotation | What it loads | Speed |
|-----------|-----------|---------------|-------|
| Unit | None (JUnit + Mockito) | Nothing | ⚡ Fast |
| Controller | `@WebMvcTest` | Web layer only | 🔵 Medium |
| Repository | `@DataJpaTest` | JPA + embedded DB | 🔵 Medium |
| Full Integration | `@SpringBootTest` | Full context | 🔴 Slow |

```java
// Unit Test — No Spring context
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    
    @Mock private PaymentGateway gateway;
    @Mock private TransactionRepository repository;
    @Mock private NotificationService notificationService;
    @InjectMocks private PaymentService paymentService;
    
    @Test
    void shouldProcessPaymentSuccessfully() {
        // Given
        PaymentRequest request = new PaymentRequest("M-001", new BigDecimal("100.00"), Currency.INR);
        GatewayResponse gatewayResponse = new GatewayResponse("TXN-001", "SUCCESS");
        
        when(gateway.charge(any())).thenReturn(gatewayResponse);
        when(repository.save(any())).thenAnswer(inv -> inv.getArgument(0));
        
        // When
        PaymentResult result = paymentService.process(request);
        
        // Then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
        assertThat(result.getTransactionId()).isEqualTo("TXN-001");
        verify(repository).save(any(Transaction.class));
        verify(notificationService).sendConfirmation(any());
    }
    
    @Test
    void shouldHandleGatewayTimeout() {
        when(gateway.charge(any())).thenThrow(new GatewayTimeoutException("Timeout"));
        
        assertThatThrownBy(() -> paymentService.process(request))
            .isInstanceOf(PaymentProcessingException.class)
            .hasMessageContaining("Gateway timeout");
        
        verify(repository).save(argThat(txn -> txn.getStatus() == PaymentStatus.FAILED));
    }
}

// Controller Test — @WebMvcTest
@WebMvcTest(PaymentController.class)
class PaymentControllerTest {
    
    @Autowired private MockMvc mockMvc;
    @MockBean private PaymentService paymentService;
    @Autowired private ObjectMapper objectMapper;
    
    @Test
    void shouldCreatePayment() throws Exception {
        CreatePaymentRequest request = new CreatePaymentRequest(
            "M-001", new BigDecimal("100.00"), Currency.INR, "idem-key-123456789");
        PaymentResponse response = new PaymentResponse("TXN-001", PaymentStatus.COMPLETED);
        
        when(paymentService.create(any())).thenReturn(response);
        
        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.transactionId").value("TXN-001"))
            .andExpect(jsonPath("$.status").value("COMPLETED"));
    }
    
    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        CreatePaymentRequest request = new CreatePaymentRequest(
            "", null, null, "short"); // All invalid
        
        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
            .andExpect(jsonPath("$.fieldErrors.merchantId").exists());
    }
}

// Repository Test — @DataJpaTest
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class PaymentRepositoryTest {
    
    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", oracle::getJdbcUrl);
        registry.add("spring.datasource.username", oracle::getUsername);
        registry.add("spring.datasource.password", oracle::getPassword);
    }
    
    @Autowired private PaymentRepository repository;
    
    @Test
    void shouldFindByMerchantIdAndDateRange() {
        // Given
        Payment payment = Payment.builder()
            .txnId("TXN-001").merchantId("M-001")
            .amount(new BigDecimal("100.00")).status(PaymentStatus.COMPLETED)
            .createdAt(Instant.now()).build();
        repository.save(payment);
        
        // When
        List<Payment> results = repository.findByMerchantIdAndCreatedAtBetween(
            "M-001", Instant.now().minus(1, ChronoUnit.HOURS), Instant.now());
        
        // Then
        assertThat(results).hasSize(1);
        assertThat(results.get(0).getTxnId()).isEqualTo("TXN-001");
    }
}
```

---

### 💎 Q13. How do you test with Testcontainers?

**Answer:**

Testcontainers provides **real Docker containers** for integration tests — no more H2 pretending to be Oracle.

```java
@SpringBootTest
@Testcontainers
class PaymentIntegrationTest {
    
    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim")
        .withInitScript("init-test-data.sql");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", oracle::getJdbcUrl);
        registry.add("spring.datasource.username", oracle::getUsername);
        registry.add("spring.datasource.password", oracle::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
    
    @Autowired private PaymentService paymentService;
    @Autowired private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    @Test
    void shouldProcessPaymentEndToEnd() {
        // Full integration: API → Service → DB → Kafka → Redis cache
        PaymentRequest request = new PaymentRequest("M-001", new BigDecimal("500.00"), Currency.INR);
        
        PaymentResult result = paymentService.process(request);
        
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
        // Verify Kafka event was published
        // Verify DB record was created
        // Verify cache was updated
    }
}
```

---

## Quick Revision Checklist

- [ ] IoC/DI: Constructor injection preferred, field injection bad
- [ ] Stereotypes: @Component, @Service, @Repository (exception translation), @Controller
- [ ] Bean Scopes: Singleton (default), Prototype-in-Singleton problem → ObjectProvider
- [ ] Auto-config: @Conditional annotations, META-INF/spring imports
- [ ] Custom Starter: Auto-config class + @ConfigurationProperties + imports file
- [ ] Security: Filter chain → JWT filter → SecurityContextHolder
- [ ] OAuth2: Client Credentials for service-to-service
- [ ] Exception Handling: @RestControllerAdvice + @ExceptionHandler
- [ ] Validation: @Valid + Bean Validation annotations
- [ ] Actuator: /health, /metrics, /prometheus, custom HealthIndicator
- [ ] Profiles: application-{profile}.yml, @Profile beans
- [ ] Filter vs Interceptor vs AOP: Servlet → MVC → Method level
- [ ] Testing: Unit (Mockito) → @WebMvcTest → @DataJpaTest → @SpringBootTest
- [ ] Testcontainers: Real Docker containers for integration tests
