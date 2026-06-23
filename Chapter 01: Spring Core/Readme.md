# Chapter 1: Spring Core — IoC, DI, Beans & Container
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [What is Spring Framework?](#1-what-is-spring-framework)
2. [The Spring Container (ApplicationContext)](#2-the-spring-container-applicationcontext)
3. [Inversion of Control (IoC)](#3-inversion-of-control-ioc)
4. [Dependency Injection (DI)](#4-dependency-injection-di)
5. [Spring Beans — Lifecycle & Scopes](#5-spring-beans--lifecycle--scopes)
6. [Bean Configuration Approaches](#6-bean-configuration-approaches)
7. [Auto-wiring](#7-auto-wiring)
8. [Spring Annotations](#8-spring-annotations)
9. [Bean Post Processors & BeanFactory](#9-bean-post-processors--beanfactory)
10. [Spring Expression Language (SpEL)](#10-spring-expression-language-spel)
11. [Profiles & Conditional Beans](#11-profiles--conditional-beans)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. What is Spring Framework?

Spring is an open-source **lightweight**, **loosely-coupled** application framework for Java. It provides comprehensive infrastructure support for Java applications.

### Core Principles:
- **IoC (Inversion of Control)** — Objects do not create their dependencies; the container does.
- **DI (Dependency Injection)** — Dependencies are injected at runtime.
- **AOP (Aspect-Oriented Programming)** — Cross-cutting concerns separated from business logic.
- **Convention over Configuration** — Reduces boilerplate.

### Spring Modules (High Level):
```
Spring Framework
├── Core Container (IoC, Beans, Context, SpEL)
├── AOP & Instrumentation
├── Data Access (JDBC, ORM, Transactions)
├── Web (MVC, WebFlux, Websocket)
├── Integration (Messaging, JMS)
└── Test
```

### Why Spring over plain Java EE?
| Feature | Spring | Java EE |
|---|---|---|
| Configuration | Annotations / XML / Java | XML heavy |
| Testing | Easy (POJO based) | Hard (needs container) |
| Coupling | Loose (IoC) | Tight |
| Boilerplate | Minimal | High |
| Community | Large, active | Shrinking |

---

## 2. The Spring Container (ApplicationContext)

The **Spring IoC Container** is responsible for:
1. Creating objects (beans)
2. Wiring their dependencies
3. Managing their lifecycle

### BeanFactory vs ApplicationContext

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation | Lazy (on demand) | Eager (at startup) |
| Internationalization | ❌ | ✅ |
| Event publication | ❌ | ✅ |
| AOP integration | Limited | Full |
| Usage | Lightweight environments | Standard (preferred) |

### ApplicationContext Implementations:
```java
// XML-based
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

// Annotation-based
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// Web context
ApplicationContext ctx = new WebApplicationContext(...);
```

### Getting Beans:
```java
// By type
MyService service = ctx.getBean(MyService.class);

// By name
MyService service = (MyService) ctx.getBean("myService");

// By name and type (safer)
MyService service = ctx.getBean("myService", MyService.class);
```

---

## 3. Inversion of Control (IoC)

**IoC** is a design principle where the control of object creation is transferred from the application code to the framework (container).

### Without IoC (tight coupling):
```java
public class OrderService {
    // OrderService creates its own dependency — TIGHT COUPLING
    private PaymentService paymentService = new PaymentService();

    public void placeOrder() {
        paymentService.process();
    }
}
```

### With IoC (loose coupling):
```java
public class OrderService {
    private PaymentService paymentService; // dependency is INJECTED

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        paymentService.process();
    }
}
```
The **Spring Container** creates both `OrderService` and `PaymentService` and wires them together.

---

## 4. Dependency Injection (DI)

DI is the mechanism through which IoC is implemented. There are 3 types:

### 4.1 Constructor Injection (RECOMMENDED)
```java
@Component
public class OrderService {
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    // @Autowired is OPTIONAL since Spring 4.3 for single constructor
    @Autowired
    public OrderService(PaymentService paymentService,
                        NotificationService notificationService) {
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
}
```
✅ **Best Practice**: Dependencies are **immutable** (final), easier to test, detects circular dependencies early.

### 4.2 Setter Injection
```java
@Component
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
✅ Use for **optional dependencies** or when reconfiguration is needed after creation.

### 4.3 Field Injection (NOT RECOMMENDED for production)
```java
@Component
public class OrderService {
    @Autowired
    private PaymentService paymentService; // injected directly into field
}
```
❌ **Problems**: Cannot make field `final`, hard to unit test without Spring context, hides dependencies.

### DI via XML Configuration:
```xml
<bean id="paymentService" class="com.example.PaymentService"/>

<bean id="orderService" class="com.example.OrderService">
    <!-- Constructor injection -->
    <constructor-arg ref="paymentService"/>
    
    <!-- Setter injection -->
    <property name="paymentService" ref="paymentService"/>
</bean>
```

### DI via Java Config:
```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }

    @Bean
    public OrderService orderService() {
        // Spring calls paymentService() — but it's proxied, so singleton is maintained
        return new OrderService(paymentService());
    }
}
```

---

## 5. Spring Beans — Lifecycle & Scopes

### 5.1 Bean Scopes

| Scope | Description | Default? |
|---|---|---|
| `singleton` | One instance per Spring container | ✅ YES |
| `prototype` | New instance on every `getBean()` call | ❌ |
| `request` | One instance per HTTP request (web) | ❌ |
| `session` | One instance per HTTP session (web) | ❌ |
| `application` | One instance per ServletContext | ❌ |
| `websocket` | One instance per WebSocket session | ❌ |

```java
@Component
@Scope("singleton")   // Default
public class DatabaseConnection { }

@Component
@Scope("prototype")
public class ShoppingCart { }  // New cart object per request

// Web scopes
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { }
```

> ⚠️ **Important**: Injecting a `prototype` bean into a `singleton` bean is a classic problem. Use `@Lookup` or `ApplicationContext.getBean()` to get fresh instances.

```java
@Component
public class SingletonService {
    // Every call returns the same prototype instance — BUG!
    @Autowired
    private PrototypeBean protoBean; // This is resolved ONCE at injection time

    // CORRECT approach: use @Lookup
    @Lookup
    public PrototypeBean getPrototypeBean() {
        return null; // Spring overrides this at runtime
    }
}
```

### 5.2 Bean Lifecycle

```
Container Started
      ↓
Bean Definition Loaded
      ↓
Bean Instantiation (constructor)
      ↓
Dependency Injection (setters/fields)
      ↓
setBeanName() [BeanNameAware]
      ↓
setBeanFactory() [BeanFactoryAware]
      ↓
setApplicationContext() [ApplicationContextAware]
      ↓
postProcessBeforeInitialization() [BeanPostProcessor]
      ↓
@PostConstruct / afterPropertiesSet() [InitializingBean] / init-method
      ↓
postProcessAfterInitialization() [BeanPostProcessor]
      ↓
Bean is READY for use
      ↓
... Application Running ...
      ↓
@PreDestroy / destroy() [DisposableBean] / destroy-method
      ↓
Container Shutdown
```

### 5.3 Lifecycle Hooks in Code:
```java
@Component
public class DatabasePool implements InitializingBean, DisposableBean {

    @PostConstruct
    public void init() {
        System.out.println("1. @PostConstruct — pool initialized");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("2. InitializingBean.afterPropertiesSet()");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("3. @PreDestroy — pool closed");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("4. DisposableBean.destroy()");
    }
}
```

Order: `@PostConstruct` → `afterPropertiesSet()` → custom `init-method`
Destroy: `@PreDestroy` → `destroy()` → custom `destroy-method`

---

## 6. Bean Configuration Approaches

### 6.1 XML Configuration (Legacy)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="...">

    <bean id="userRepository" class="com.example.UserRepository"
          scope="singleton"
          init-method="init"
          destroy-method="cleanup"/>

    <bean id="userService" class="com.example.UserService">
        <constructor-arg ref="userRepository"/>
        <property name="timeout" value="30"/>
    </bean>
</beans>
```

### 6.2 Java-based Configuration (Modern)
```java
@Configuration          // Marks this as a Spring config class
@ComponentScan("com.example")  // Scan for @Component, @Service, etc.
@PropertySource("classpath:app.properties")
public class AppConfig {

    @Value("${db.url}")
    private String dbUrl;

    @Bean(initMethod = "init", destroyMethod = "cleanup")
    @Scope("singleton")
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setUrl(dbUrl);
        return ds;
    }
}
```

### 6.3 Annotation-based (Component Scanning)
```java
@Component       // Generic Spring-managed component
@Service         // Business logic layer (same as @Component, semantic)
@Repository      // Data access layer (adds exception translation)
@Controller      // Web layer (same as @Component + request mapping)
@RestController  // @Controller + @ResponseBody
```

---

## 7. Auto-wiring

### @Autowired Resolution Order:
1. Match by **type** (primary mechanism)
2. If multiple matches → match by **name** (field name / parameter name)
3. Use `@Qualifier` to explicitly specify
4. Use `@Primary` to mark preferred bean

```java
// Multiple implementations
@Component
public class EmailNotificationService implements NotificationService { }

@Component
@Primary  // This will be preferred when injecting NotificationService
public class SmsNotificationService implements NotificationService { }

// Usage
@Autowired
private NotificationService notificationService; // Gets SmsNotificationService

// Explicit choice with @Qualifier
@Autowired
@Qualifier("emailNotificationService")
private NotificationService notificationService; // Gets EmailNotificationService
```

### @Autowired(required = false):
```java
@Autowired(required = false)
private OptionalFeature optionalFeature; // Won't throw if bean not found

// Better approach with Optional
@Autowired
private Optional<OptionalFeature> optionalFeature;
```

---

## 8. Spring Annotations

### Stereotype Annotations:
```java
@Component   // Base stereotype
@Service     // Service layer
@Repository  // DAO layer — enables PersistenceExceptionTranslation
@Controller  // MVC controller
@RestController // REST controller
```

### Configuration Annotations:
```java
@Configuration     // Java config class
@Bean              // Method produces a Spring bean
@ComponentScan     // Enable component scanning
@Import            // Import other @Configuration classes
@PropertySource    // Load .properties file
@Profile           // Conditional bean based on active profile
@Conditional       // Fine-grained condition
@Lazy              // Delay initialization
@DependsOn         // Ensure another bean is created first
@Scope             // Define bean scope
```

### Injection Annotations:
```java
@Autowired    // Spring's injection annotation
@Qualifier    // Disambiguate multiple beans
@Value        // Inject property values
@Primary      // Preferred bean when multiple candidates
@Resource     // JSR-250 — inject by name
@Inject       // JSR-330 — like @Autowired
```

### Value Injection:
```java
@Component
public class AppSettings {

    @Value("${server.port:8080}")     // With default value
    private int serverPort;

    @Value("${app.name}")
    private String appName;

    @Value("#{T(java.lang.Math).PI}") // SpEL expression
    private double pi;

    @Value("#{systemProperties['user.home']}")
    private String userHome;
}
```

---

## 9. Bean Post Processors & BeanFactory

### BeanPostProcessor:
Intercepts bean creation for all beans. Used internally by Spring for AOP proxy creation, `@Autowired` processing, etc.

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("Before init: " + beanName);
        return bean; // Return the bean (or a wrapper/proxy)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("After init: " + beanName);
        return bean;
    }
}
```

### BeanFactoryPostProcessor:
Modifies **bean definitions** before any beans are instantiated. `PropertySourcesPlaceholderConfigurer` is an example.

```java
@Component
public class CustomBFPP implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf)
            throws BeansException {
        BeanDefinition bd = bf.getBeanDefinition("myBean");
        bd.setScope("prototype"); // Change scope before instantiation
    }
}
```

---

## 10. Spring Expression Language (SpEL)

SpEL is a powerful expression language for querying and manipulating objects at runtime.

```java
// In @Value
@Value("#{1 + 2}")                         // 3
@Value("#{'Hello ' + 'World'}")            // Hello World
@Value("#{T(Math).random() * 100}")        // random number
@Value("#{userService.getDefaultUser()}")  // Call bean method
@Value("#{someBean.property > 0 ? 'pos' : 'neg'}")  // Ternary

// Collection operations
@Value("#{list.?[value > 10]}")            // Filter
@Value("#{list.^[value > 10]}")            // First match
@Value("#{list.$[value > 10]}")            // Last match
@Value("#{list.![value * 2]}")             // Project/transform

// In @Query (Spring Data JPA)
@Query("SELECT u FROM User u WHERE u.name = :#{#name}")
List<User> findByName(@Param("name") String name);
```

---

## 11. Profiles & Conditional Beans

### @Profile:
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Real database
        return new HikariDataSource(productionConfig());
    }
}
```

Activation:
```properties
# application.properties
spring.profiles.active=dev
```
```java
// Programmatic
ctx.getEnvironment().setActiveProfiles("prod");
```

### @Conditional:
```java
// Custom condition
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return System.getProperty("os.name").contains("Linux");
    }
}

@Bean
@Conditional(OnLinuxCondition.class)
public FileWatcher linuxFileWatcher() {
    return new LinuxFileWatcher();
}

// Spring Boot conditionals (preview)
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
@ConditionalOnClass(name = "com.example.SomeClass")
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnWebApplication
```

---

## 12. Interview Questions & Answers

---

### Q1: What is the difference between BeanFactory and ApplicationContext?
**A:** `BeanFactory` is the basic container that lazily instantiates beans only when requested. `ApplicationContext` extends `BeanFactory` and adds enterprise features: eager initialization, event publication, i18n support, AOP integration, and auto-detection of `BeanPostProcessor`s. Always prefer `ApplicationContext` in production.

---

### Q2: What are the different types of DI in Spring? Which is recommended and why?
**A:** Three types — Constructor, Setter, and Field injection.
- **Constructor injection** is recommended: dependencies are immutable (`final`), all required deps are visible, easier unit testing, circular dependency detection at startup.
- **Setter injection** for optional dependencies.
- **Field injection** is discouraged: hides dependencies, can't make fields final, requires Spring context for testing.

---

### Q3: What is the default scope of a Spring bean?
**A:** `singleton` — one instance per Spring IoC container. Not per JVM; if you have multiple ApplicationContexts, each has its own singleton.

---

### Q4: What happens when you inject a prototype bean into a singleton bean?
**A:** The prototype bean is injected once at creation time, so the singleton always uses the same instance — defeating the purpose of prototype scope. Fix: use `@Lookup` method injection, `ObjectFactory<T>`, or `ApplicationContext.getBean()`.

---

### Q5: Explain the complete Spring Bean lifecycle.
**A:** 
1. Instantiation (constructor)
2. Populate properties (DI)
3. `BeanNameAware.setBeanName()`
4. `BeanFactoryAware.setBeanFactory()`
5. `ApplicationContextAware.setApplicationContext()`
6. `BeanPostProcessor.postProcessBeforeInitialization()`
7. `@PostConstruct` / `InitializingBean.afterPropertiesSet()` / `init-method`
8. `BeanPostProcessor.postProcessAfterInitialization()`
9. Bean in use
10. `@PreDestroy` / `DisposableBean.destroy()` / `destroy-method`

---

### Q6: What is the difference between @Component, @Service, @Repository, and @Controller?
**A:** All are specializations of `@Component` — functionally equivalent for component scanning. The difference is **semantic** and **behavioral**:
- `@Repository`: Adds **persistence exception translation** (wraps JPA/JDBC exceptions into Spring's DataAccessException hierarchy).
- `@Service`: Marks business logic — no additional behavior currently.
- `@Controller`: Marks web MVC controller — enables `@RequestMapping`.
- `@Component`: Generic component.

---

### Q7: What is @Autowired and how does Spring resolve ambiguity?
**A:** `@Autowired` tells Spring to inject a dependency. Resolution order:
1. By type — finds all beans of that type.
2. If single match → inject it.
3. If multiple matches → try by name (field/parameter name matches bean name).
4. If still ambiguous → throw `NoUniqueBeanDefinitionException`.
5. Fix: use `@Qualifier("beanName")` or mark one bean `@Primary`.

---

### Q8: What is a circular dependency and how does Spring handle it?
**A:** Circular dependency = A depends on B, B depends on A.
- With **constructor injection**: Spring throws `BeanCurrentlyInCreationException` at startup.
- With **setter/field injection**: Spring resolves it using a three-level cache (early reference exposure) — but this relies on AOP proxies and can be fragile.
- **Best fix**: Redesign to eliminate the circular dependency. Use `@Lazy` on one dependency as a workaround.

---

### Q9: What is the difference between @Bean and @Component?
**A:** 
- `@Component` is class-level — Spring auto-detects via component scanning.
- `@Bean` is method-level inside `@Configuration` — gives explicit control over creation, can configure third-party classes you don't own.

---

### Q10: What is BeanPostProcessor? Give a real use case.
**A:** `BeanPostProcessor` intercepts every bean before/after initialization. Spring itself uses it for:
- `AutowiredAnnotationBeanPostProcessor` — processes `@Autowired`
- `RequiredAnnotationBeanPostProcessor` — checks `@Required`
- `AbstractAdvisorAutoProxyCreator` — creates AOP proxies
Custom use case: logging, security checks, modifying bean properties uniformly.

---

### Q11: What is @Primary and @Qualifier? When to use which?
**A:**
- `@Primary` — marks a bean as the default candidate when multiple beans of the same type exist. Good for "mostly use this one."
- `@Qualifier` — injection-point level, explicitly names which bean to inject. Good for "use specifically this one here."
Use `@Primary` for global default; `@Qualifier` for exception cases.

---

### Q12: Can Spring inject a null value? What is @Nullable?
**A:** Yes. `@Autowired(required=false)` + `@Nullable` (from `org.springframework.lang`) signals that null is acceptable:
```java
@Autowired
@Nullable
private OptionalService optionalService;
```
If the bean doesn't exist, Spring injects `null` without throwing an exception.

---

### Q13: What is the difference between @Configuration and @Component for declaring beans?
**A:** `@Configuration` uses **CGLIB proxying** — calls to `@Bean` methods are intercepted, ensuring singleton semantics (same instance returned on repeated calls). `@Component` with `@Bean` methods uses **lite mode** — no proxying, so each call creates a new instance. This matters when one `@Bean` calls another.

---

### Q14: How does Spring handle the Environment and properties?
**A:** The `Environment` abstraction unifies property sources: system properties, environment variables, `application.properties`, JNDI, etc. `@Value("${key}")` + `@PropertySource` loads `.properties` files. Spring Boot auto-loads `application.properties`/`application.yml` and supports externalized config.

---

### Q15: What is the difference between @Import and @ComponentScan?
**A:** 
- `@ComponentScan` scans a package and registers all `@Component`-annotated classes.
- `@Import` explicitly imports specific `@Configuration` classes, `ImportSelector`, or `ImportBeanDefinitionRegistrar` — used for fine-grained, conditional configuration (like Spring Boot auto-configuration).

---

*End of Chapter 1 — Spring Core*
