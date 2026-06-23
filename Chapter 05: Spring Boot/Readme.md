# Chapter 5: Spring Boot — Auto-configuration, Starters & Production Features
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [What is Spring Boot?](#1-what-is-spring-boot)
2. [Auto-Configuration Deep Dive](#2-auto-configuration-deep-dive)
3. [Spring Boot Starters](#3-spring-boot-starters)
4. [application.properties / application.yml](#4-applicationproperties--applicationyml)
5. [Externalized Configuration & @ConfigurationProperties](#5-externalized-configuration--configurationproperties)
6. [Spring Boot Actuator](#6-spring-boot-actuator)
7. [Spring Boot DevTools](#7-spring-boot-devtools)
8. [Embedded Server & Deployment](#8-embedded-server--deployment)
9. [Custom Auto-Configuration](#9-custom-auto-configuration)
10. [Testing in Spring Boot](#10-testing-in-spring-boot)
11. [Spring Boot + Docker](#11-spring-boot--docker)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. What is Spring Boot?

Spring Boot makes it easy to create **stand-alone, production-ready** Spring applications with **minimal configuration**.

### Key Features:
- **Auto-configuration** — automatically configures Spring based on JAR dependencies
- **Starters** — curated dependency descriptors
- **Embedded servers** — Tomcat, Jetty, Undertow built in
- **Production ready** — Actuator health checks, metrics, tracing
- **No code generation** — no XML required

### Spring Boot vs Spring Framework:
| Aspect | Spring Framework | Spring Boot |
|---|---|---|
| Configuration | Manual | Auto-configured |
| Server | External (WAR deploy) | Embedded (JAR) |
| Dependencies | Manual management | Starters (curated) |
| Properties | Manual `@Bean` setup | `application.properties` |
| Production features | Separate | Built-in (Actuator) |
| Startup | Slower (manual) | Faster (auto) |

### Main Application Class:
```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

`@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`

### SpringApplication Customization:
```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApplication.class);
        app.setBannerMode(Banner.Mode.OFF);
        app.setDefaultProperties(Map.of("server.port", "8081"));
        app.addListeners(new ApplicationEventListener());
        app.run(args);
    }
}
```

---

## 2. Auto-Configuration Deep Dive

Auto-configuration automatically configures beans based on **what's on the classpath**.

### How it works:
```
@SpringBootApplication
    └── @EnableAutoConfiguration
            └── AutoConfigurationImportSelector
                    └── Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
                            └── Loads all AutoConfiguration classes
                                    └── Each class has @ConditionalOn... conditions
                                            └── If conditions pass → beans registered
```

### Example: DataSource Auto-Configuration:
```java
// Spring Boot's internal class (simplified)
@Configuration
@ConditionalOnClass(DataSource.class)              // DataSource class is on classpath
@ConditionalOnMissingBean(DataSource.class)        // No custom DataSource bean defined
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

### @Conditional Annotations:
```java
@ConditionalOnClass(DataSource.class)      // Class is on classpath
@ConditionalOnMissingClass("...")          // Class is NOT on classpath
@ConditionalOnBean(DataSource.class)       // Bean of type exists
@ConditionalOnMissingBean(DataSource.class) // Bean of type does NOT exist
@ConditionalOnProperty(                    // Property has value
    name = "feature.enabled",
    havingValue = "true",
    matchIfMissing = false
)
@ConditionalOnExpression("${app.flag} == true")  // SpEL condition
@ConditionalOnWebApplication               // Is a web app
@ConditionalOnNotWebApplication            // Is NOT a web app
@ConditionalOnResource("classpath:config.xml")  // Resource exists
@ConditionalOnJava(JavaVersion.TWENTY_ONE)  // Java version
@ConditionalOnSingleCandidate(DataSource.class) // Exactly one candidate
```

### Debugging Auto-Configuration:
```bash
# Run with --debug flag
java -jar myapp.jar --debug

# Or in application.properties
debug=true
```
Output shows:
- **Positive matches** — auto-configurations applied
- **Negative matches** — auto-configurations skipped and why
- **Exclusions** — explicitly excluded

### Excluding Auto-Configuration:
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
```
Or via properties:
```properties
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

---

## 3. Spring Boot Starters

Starters are **all-in-one dependency descriptors** — one dependency pulls in everything needed.

| Starter | What it includes |
|---|---|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson, Validation |
| `spring-boot-starter-data-jpa` | Spring Data JPA, Hibernate, JDBC |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Hamcrest |
| `spring-boot-starter-actuator` | Production monitoring |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-mail` | JavaMail |
| `spring-boot-starter-thymeleaf` | Thymeleaf template engine |
| `spring-boot-starter-webflux` | Spring WebFlux, Reactor, Netty |
| `spring-boot-starter-amqp` | Spring AMQP, RabbitMQ |
| `spring-boot-starter-kafka` | Spring Kafka |
| `spring-boot-starter-redis` | Spring Data Redis, Lettuce |
| `spring-boot-starter-logging` | Logback (default) |
| `spring-boot-starter-aop` | Spring AOP, AspectJ |

---

## 4. application.properties / application.yml

### Profile-specific configuration:
```
application.properties          # always loaded
application-dev.properties      # loaded when profile=dev
application-prod.properties     # loaded when profile=prod
application-dev.yml             # YAML version
```

### Configuration hierarchy (lowest to highest priority):
```
1. Default values (code)
2. application.properties
3. application-{profile}.properties
4. OS environment variables
5. JVM system properties (-D)
6. Command-line arguments
7. Spring Cloud Config Server (highest)
```

### Common Properties:
```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api
  error:
    include-message: always
    include-binding-errors: always

spring:
  application:
    name: my-service

  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: ${DB_PASSWORD}  # from environment variable
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true

  profiles:
    active: dev

  cache:
    type: redis

  data:
    redis:
      host: localhost
      port: 6379
      timeout: 60000ms

logging:
  level:
    root: INFO
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

---

## 5. Externalized Configuration & @ConfigurationProperties

### @ConfigurationProperties:
```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Validated  // Enable Bean Validation on config
@Data
public class AppProperties {

    @NotNull
    private String name;

    @NotNull
    private String version;

    @Valid
    private Database database = new Database();

    @Valid
    private Security security = new Security();

    @Data
    public static class Database {
        @NotNull
        private String url;
        private String username;
        private String password;

        @Min(1) @Max(100)
        private int maxPoolSize = 10;

        @NotNull
        @DurationUnit(ChronoUnit.SECONDS)
        private Duration timeout = Duration.ofSeconds(30);
    }

    @Data
    public static class Security {
        private String jwtSecret;

        @DurationUnit(ChronoUnit.HOURS)
        private Duration jwtExpiration = Duration.ofHours(24);

        private List<String> allowedOrigins = new ArrayList<>();
    }
}
```

```yaml
# application.yml
app:
  name: My Application
  version: 1.0.0
  database:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    max-pool-size: 20
    timeout: 30s
  security:
    jwt-secret: my-secret-key
    jwt-expiration: 24h
    allowed-origins:
      - https://app.example.com
      - https://admin.example.com
```

```java
// Usage
@Service
@RequiredArgsConstructor
public class ConfigDemoService {
    private final AppProperties appProperties;

    public void demo() {
        String url = appProperties.getDatabase().getUrl();
        Duration timeout = appProperties.getDatabase().getTimeout();
    }
}
```

### Enable @ConfigurationProperties:
```java
// On main class or @Configuration class:
@EnableConfigurationProperties(AppProperties.class)

// Or use @ConfigurationPropertiesScan
@ConfigurationPropertiesScan("com.example.config")
```

---

## 6. Spring Boot Actuator

Actuator provides **production-ready endpoints** for monitoring and management.

### Key Endpoints:

| Endpoint | Description |
|---|---|
| `/actuator/health` | Application health status |
| `/actuator/info` | Application info |
| `/actuator/metrics` | Application metrics |
| `/actuator/metrics/{name}` | Specific metric |
| `/actuator/env` | Environment properties |
| `/actuator/beans` | All Spring beans |
| `/actuator/mappings` | All @RequestMapping |
| `/actuator/loggers` | Logger levels |
| `/actuator/loggers/{name}` | Change log level at runtime |
| `/actuator/heapdump` | JVM heap dump |
| `/actuator/threaddump` | JVM thread dump |
| `/actuator/httptrace` | Recent HTTP traces |
| `/actuator/prometheus` | Prometheus metrics |
| `/actuator/shutdown` | Graceful shutdown (POST) |

### Configuration:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
        exclude: shutdown,heapdump
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
    shutdown:
      enabled: true
  info:
    env:
      enabled: true
```

### Custom Health Indicator:
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("version", getDatabaseVersion(conn))
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.unknown().build();
    }
}
```

### Custom Metrics (Micrometer):
```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;
    private final Gauge pendingOrders;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.total")
            .description("Total orders placed")
            .tag("type", "all")
            .register(registry);

        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(registry);

        this.pendingOrders = Gauge.builder("orders.pending",
            orderRepository, OrderRepository::countPending)
            .register(registry);
    }

    public Order placeOrder(OrderRequest req) {
        return orderTimer.record(() -> {
            Order order = processOrder(req);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

## 7. Spring Boot DevTools

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

Features:
- **Automatic restart** on classpath changes (faster than full restart)
- **LiveReload** — triggers browser refresh on resource changes
- **Property defaults** — disables caching (Thymeleaf, etc.) in development
- **Remote debugging** support

```properties
# Disable for specific situations
spring.devtools.restart.enabled=false
spring.devtools.livereload.enabled=false
# Trigger file (restart only when this file changes)
spring.devtools.restart.trigger-file=.trigger
```

---

## 8. Embedded Server & Deployment

### Switch Server (Tomcat → Jetty):
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Server Customization:
```java
@Component
public class ServerConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(8443);
        factory.addConnectorCustomizers(connector -> {
            connector.setURIEncoding("UTF-8");
        });
    }
}
```

### Deployment as JAR (Fat JAR):
```bash
mvn clean package -DskipTests
java -jar target/myapp-1.0.0.jar
java -jar myapp.jar --server.port=8081 --spring.profiles.active=prod
```

### Deployment as WAR (legacy):
```java
@SpringBootApplication
public class MyApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(MyApplication.class);
    }
}
```

### Graceful Shutdown:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

## 9. Custom Auto-Configuration

### Create custom starter:
```java
// Auto-configuration class
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyServiceProperties.class)
@ConditionalOnProperty(prefix = "my.service", name = "enabled", havingValue = "true")
public class MyServiceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyServiceProperties props) {
        return new MyService(props.getUrl(), props.getApiKey());
    }
}
```

```properties
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfigure.MyServiceAutoConfiguration
```

```java
// Properties
@Data
@ConfigurationProperties(prefix = "my.service")
public class MyServiceProperties {
    private String url;
    private String apiKey;
    private boolean enabled = true;
}
```

---

## 10. Testing in Spring Boot

### @SpringBootTest:
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void shouldCreateAndRetrieveUser() {
        // Create
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");
        ResponseEntity<UserResponse> createResponse =
            restTemplate.postForEntity("/api/users", request, UserResponse.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        Long userId = createResponse.getBody().getId();

        // Retrieve
        ResponseEntity<UserResponse> getResponse =
            restTemplate.getForEntity("/api/users/" + userId, UserResponse.class);

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getName()).isEqualTo("John");
    }
}
```

### @WebMvcTest (slice test):
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;  // Mocked

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldReturn200WhenUserExists() throws Exception {
        UserResponse user = new UserResponse(1L, "John", "john@example.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));

        mockMvc.perform(get("/api/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    void shouldReturn400OnInvalidInput() throws Exception {
        CreateUserRequest bad = new CreateUserRequest("", "invalid-email");

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bad)))
            .andExpect(status().isBadRequest());
    }
}
```

### @DataJpaTest (slice test):
```java
@DataJpaTest
@ActiveProfiles("test")
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        User user = new User("John", "john@example.com");
        entityManager.persist(user);
        entityManager.flush();

        Optional<User> found = userRepository.findByEmail("john@example.com");
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

---

## 11. Spring Boot + Docker

### Dockerfile:
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/myapp-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Multi-stage Dockerfile (optimized layers):
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]
```

### Spring Boot Buildpacks (no Dockerfile needed!):
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest
```

---

## 12. Interview Questions & Answers

---

### Q1: What is @SpringBootApplication and what are its components?
**A:** It's a meta-annotation combining:
- `@Configuration` — marks it as a configuration class
- `@EnableAutoConfiguration` — triggers auto-configuration
- `@ComponentScan` — scans for components in the current package and sub-packages

---

### Q2: How does Spring Boot Auto-Configuration work?
**A:** `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, which reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each listed class is a `@Configuration` with `@Conditional` annotations. If conditions are met (e.g., class on classpath, no existing bean), the configuration is applied and beans are registered.

---

### Q3: How do you disable a specific auto-configuration?
**A:** Two ways:
1. `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`
2. `spring.autoconfigure.exclude=fully.qualified.ClassName` in properties

---

### Q4: What is the difference between @ConfigurationProperties and @Value?
**A:** `@Value` injects single values — limited to simple types, hard to refactor. `@ConfigurationProperties` maps an entire group of properties to a POJO — supports nested objects, lists, maps, type conversion, validation with `@Validated`, and IDE auto-completion with metadata. Prefer `@ConfigurationProperties` for groups of related config.

---

### Q5: What is Spring Boot Actuator?
**A:** Actuator provides production-ready endpoints for health checking, metrics, environment inspection, thread dumps, etc. It integrates with monitoring systems like Prometheus and Grafana via Micrometer. Key endpoints: `/actuator/health`, `/actuator/metrics`, `/actuator/env`.

---

### Q6: What is the Spring Boot parent POM?
**A:** `spring-boot-starter-parent` provides: dependency management (curated versions for hundreds of libraries), plugin configurations (maven-compiler-plugin, spring-boot-maven-plugin), resource filtering, property placeholders. You inherit from it in your POM to avoid version conflicts.

---

### Q7: What is the embedded server in Spring Boot? What servers are supported?
**A:** Spring Boot embeds the server inside the JAR. Supported: **Tomcat** (default), **Jetty**, **Undertow**. This enables running as `java -jar myapp.jar` without installing a server. Spring WebFlux uses **Netty** by default.

---

### Q8: How does Spring Boot handle properties across environments?
**A:** Profile-specific properties files: `application-dev.yml`, `application-prod.yml`. Activated via `spring.profiles.active`. Higher-priority sources (env vars, command-line args) override properties file values. Spring Cloud Config externalizes config to a central server.

---

### Q9: What is Spring Boot's @Conditional mechanism?
**A:** `@Conditional` allows beans to be registered only when specific conditions are true. Spring Boot provides rich implementations: `@ConditionalOnClass`, `@ConditionalOnBean`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, etc. This is the foundation of auto-configuration.

---

### Q10: How do you create a custom Spring Boot Starter?
**A:** 
1. Create auto-configuration class with `@Configuration` + `@Conditional` annotations
2. Create `@ConfigurationProperties` class for properties
3. Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. Package with a `spring-boot-autoconfigure` dependency
5. Optionally create a separate `*-starter` module that depends on your autoconfigure module

---

### Q11: What is graceful shutdown in Spring Boot?
**A:** With `server.shutdown=graceful`, Spring Boot waits for in-flight requests to complete before shutting down (up to `spring.lifecycle.timeout-per-shutdown-phase` duration). In-progress requests are handled; new requests are rejected. Important for Kubernetes zero-downtime deployments.

---

### Q12: How does @SpringBootTest WebEnvironment work?
**A:** `MOCK` — loads WebApplicationContext with mock servlet environment (default). `RANDOM_PORT` — starts a real embedded server on a random port. `DEFINED_PORT` — uses `server.port`. `NONE` — no web environment. Use `RANDOM_PORT` for full integration tests with `TestRestTemplate` or `WebTestClient`.

---

*End of Chapter 5 — Spring Boot*
