# Chapter 2: Spring AOP — Aspect-Oriented Programming
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [What is AOP?](#1-what-is-aop)
2. [Core AOP Concepts & Terminology](#2-core-aop-concepts--terminology)
3. [Spring AOP vs AspectJ](#3-spring-aop-vs-aspectj)
4. [Pointcut Expressions](#4-pointcut-expressions)
5. [Advice Types](#5-advice-types)
6. [@Aspect — Full Example](#6-aspect--full-example)
7. [AOP Proxy Mechanism](#7-aop-proxy-mechanism)
8. [Ordering Aspects](#8-ordering-aspects)
9. [Real-World AOP Use Cases](#9-real-world-aop-use-cases)
10. [Interview Questions & Answers](#10-interview-questions--answers)

---

## 1. What is AOP?

**Aspect-Oriented Programming (AOP)** is a programming paradigm that separates **cross-cutting concerns** from business logic.

### Cross-Cutting Concerns (problems AOP solves):
- Logging
- Security / Authorization
- Transaction management
- Caching
- Exception handling
- Performance monitoring
- Auditing

### Problem without AOP:
```java
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void placeOrder(Order order) {
        // LOGGING — cross-cutting concern scattered everywhere
        log.info("Placing order: {}", order);

        // SECURITY — cross-cutting concern
        if (!SecurityContext.isAuthenticated()) throw new UnauthorizedException();

        // TRANSACTION — cross-cutting concern
        transactionManager.begin();
        try {
            // ACTUAL BUSINESS LOGIC — only this belongs here!
            orderRepository.save(order);
            paymentService.charge(order);
            transactionManager.commit();
        } catch (Exception e) {
            transactionManager.rollback();
            throw e;
        }

        // LOGGING again
        log.info("Order placed successfully: {}", order.getId());
    }
}
```

### Solution with AOP:
```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        // ONLY business logic — clean!
        orderRepository.save(order);
        paymentService.charge(order);
    }
}

// Cross-cutting concerns in Aspects
@Aspect @Component
public class LoggingAspect { /* handles logging for all methods */ }

@Aspect @Component
public class SecurityAspect { /* handles auth for all methods */ }

@Aspect @Component
public class TransactionAspect { /* handles transactions */ }
```

---

## 2. Core AOP Concepts & Terminology

| Term | Definition | Example |
|---|---|---|
| **Aspect** | Module encapsulating a cross-cutting concern | `LoggingAspect` class |
| **Join Point** | Point in execution where aspect can be applied | Method execution, exception |
| **Pointcut** | Expression that selects specific join points | `execution(* com.example.service.*.*(..))` |
| **Advice** | Action taken at a join point | `@Before`, `@After`, `@Around` |
| **Target Object** | Object being advised | `OrderService` instance |
| **Proxy** | Object created by AOP to wrap the target | JDK Proxy or CGLIB proxy |
| **Weaving** | Process of linking aspects with target | Compile-time, load-time, or runtime |

```
         Aspect
           │
    ┌──────▼──────┐
    │   Pointcut  │ ← selects join points
    └──────┬──────┘
           │ matches
    ┌──────▼──────┐
    │  Join Point │ ← e.g., method execution
    └──────┬──────┘
           │ applies
    ┌──────▼──────┐
    │   Advice    │ ← code to execute
    └─────────────┘
```

---

## 3. Spring AOP vs AspectJ

| Feature | Spring AOP | AspectJ |
|---|---|---|
| Weaving | Runtime (proxy-based) | Compile-time, load-time, runtime |
| Join points supported | Method execution only | Method, field, constructor, etc. |
| Target type | Spring beans only | Any Java object |
| Performance | Slight overhead (proxy) | Faster (woven into bytecode) |
| Configuration | Spring annotations | AspectJ syntax or annotations |
| Use case | 90% of enterprise use cases | Complex, non-Spring scenarios |

Spring AOP uses AspectJ **annotations** (`@Aspect`, `@Before`, etc.) but implements them via Spring proxies — not AspectJ's weaver.

---

## 4. Pointcut Expressions

### execution() — most common:
```
execution([modifiers] return-type [declaring-type].method-name(params) [throws])
```

```java
// Any method in any class
execution(* *(..))

// Any public method
execution(public * *(..))

// Any method returning String
execution(String *(..))

// Any method in OrderService
execution(* com.example.OrderService.*(..))

// Any method in any class in service package
execution(* com.example.service.*.*(..))

// Any method in service package or sub-packages
execution(* com.example.service..*.*(..))

// Specific method with specific param type
execution(* com.example.OrderService.placeOrder(com.example.Order))

// Any method with any params
execution(* com.example.service.*.*(..))

// Any method with no params
execution(* com.example.service.*.*())

// Match and combine
execution(* com.example.service.*.*(..)) && !execution(* com.example.service.*.toString(..))
```

### within() — type-level matching:
```java
within(com.example.service.*)           // Any class in service package
within(com.example.service..*)          // Any class in service or sub-packages
within(@org.springframework.stereotype.Service *)  // Any @Service class
```

### @annotation() — method annotation matching:
```java
@annotation(com.example.Audited)        // Methods annotated with @Audited
@annotation(org.springframework.transaction.annotation.Transactional)
```

### bean() — Spring bean name matching:
```java
bean(orderService)    // Specific bean
bean(*Service)        // Beans ending in "Service"
```

### args() — argument type matching:
```java
args(com.example.Order)      // Methods with Order as first argument
args(com.example.Order, ..)  // Methods with Order as first argument + any others
```

### Combining Pointcuts:
```java
@Pointcut("execution(* com.example.service.*.*(..))")
private void serviceLayer() {}

@Pointcut("@annotation(com.example.Loggable)")
private void loggableMethods() {}

@Pointcut("serviceLayer() && loggableMethods()")
private void serviceLayerLoggable() {}
```

---

## 5. Advice Types

| Annotation | When it runs |
|---|---|
| `@Before` | Before method execution |
| `@After` | After method (always, like finally) |
| `@AfterReturning` | After successful return |
| `@AfterThrowing` | After exception thrown |
| `@Around` | Wraps the method (most powerful) |

### @Before:
```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    String method = joinPoint.getSignature().getName();
    Object[] args = joinPoint.getArgs();
    log.info("Calling {} with args: {}", method, Arrays.toString(args));
}
```

### @AfterReturning:
```java
@AfterReturning(
    pointcut = "execution(* com.example.service.*.*(..))",
    returning = "result"
)
public void logAfterReturning(JoinPoint joinPoint, Object result) {
    log.info("Method {} returned: {}", joinPoint.getSignature().getName(), result);
}
```

### @AfterThrowing:
```java
@AfterThrowing(
    pointcut = "execution(* com.example.service.*.*(..))",
    throwing = "ex"
)
public void logException(JoinPoint joinPoint, Exception ex) {
    log.error("Exception in {}: {}", joinPoint.getSignature().getName(), ex.getMessage());
    // Can inspect but NOT suppress the exception here
}
```

### @After (finally):
```java
@After("execution(* com.example.service.*.*(..))")
public void afterFinally(JoinPoint joinPoint) {
    // Runs regardless of outcome (like finally block)
    log.info("Method completed: {}", joinPoint.getSignature().getName());
}
```

### @Around (most powerful):
```java
@Around("execution(* com.example.service.*.*(..))")
public Object measureExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
    long start = System.currentTimeMillis();
    String method = joinPoint.getSignature().toShortString();

    try {
        // Proceed with actual method execution
        Object result = joinPoint.proceed();
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} completed in {}ms", method, elapsed);
        return result; // Return the actual result

    } catch (Exception ex) {
        log.error("{} threw exception: {}", method, ex.getMessage());
        throw ex; // Re-throw (or swallow if needed)
    }
}
```

> ⚠️ **Critical**: In `@Around`, you MUST call `joinPoint.proceed()` to actually invoke the target method, and return its result. Forgetting to return the result is a common bug.

---

## 6. @Aspect — Full Example

### Custom @Audited annotation:
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String action() default "";
}
```

### Complete Aspect:
```java
@Aspect
@Component
@Order(1)  // Lower number = higher priority
public class AuditingAspect {

    @Autowired
    private AuditLogRepository auditLogRepo;

    // Reusable pointcut
    @Pointcut("@annotation(audited)")
    private void auditedMethod(Audited audited) {}

    @Around("auditedMethod(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        String user = SecurityContextHolder.getContext()
                .getAuthentication().getName();
        String method = pjp.getSignature().getName();
        String action = audited.action().isEmpty() ? method : audited.action();

        AuditLog log = new AuditLog();
        log.setUser(user);
        log.setAction(action);
        log.setTimestamp(LocalDateTime.now());
        log.setArgs(Arrays.toString(pjp.getArgs()));

        try {
            Object result = pjp.proceed();
            log.setStatus("SUCCESS");
            auditLogRepo.save(log);
            return result;
        } catch (Throwable ex) {
            log.setStatus("FAILED: " + ex.getMessage());
            auditLogRepo.save(log);
            throw ex;
        }
    }
}
```

### Usage:
```java
@Service
public class UserService {

    @Audited(action = "DELETE_USER")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
}
```

---

## 7. AOP Proxy Mechanism

Spring AOP creates **proxies** around target beans. Two types:

### JDK Dynamic Proxy:
- Used when the target implements **at least one interface**
- Creates a Proxy that implements the same interface(s)
- Method calls go through `InvocationHandler`

```
Client → JDK Proxy (implements MyService interface) → Target Object
```

### CGLIB Proxy:
- Used when the target does **NOT** implement any interface
- Subclasses the target class at runtime
- Overrides methods to intercept calls

```
Client → CGLIB Subclass of MyServiceImpl → Target Object
```

```java
// Force CGLIB even for interfaces:
@EnableAspectJAutoProxy(proxyTargetClass = true)
// or in Spring Boot application.properties:
// spring.aop.proxy-target-class=true
```

### Proxy Limitations:
```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // Calling THIS won't go through the AOP proxy!
        // @Transactional on processPayment WON'T work here
        this.processPayment(order);
    }

    @Transactional  // AOP @Around advice
    public void processPayment(Order order) {
        paymentRepository.save(order);
    }
}
```

> ⚠️ **Self-invocation problem**: Calling a method from within the same class bypasses the proxy. Fix: inject the bean itself (`@Autowired OrderService self`) or use AspectJ weaving.

---

## 8. Ordering Aspects

When multiple aspects apply to the same join point, use `@Order`:

```java
@Aspect @Component @Order(1)  // Runs first (outermost)
public class SecurityAspect { }

@Aspect @Component @Order(2)
public class LoggingAspect { }

@Aspect @Component @Order(3)  // Runs last (innermost)
public class TransactionAspect { }
```

Execution for `@Around`:
```
SecurityAspect.before → LoggingAspect.before → TransactionAspect.before
        → TARGET METHOD ←
TransactionAspect.after → LoggingAspect.after → SecurityAspect.after
```

---

## 9. Real-World AOP Use Cases

### 9.1 Rate Limiting:
```java
@Around("@annotation(rateLimit)")
public Object rateLimit(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
    String key = "rl:" + pjp.getSignature().getName();
    long count = redisTemplate.opsForValue().increment(key);
    if (count == 1) {
        redisTemplate.expire(key, rateLimit.window(), TimeUnit.SECONDS);
    }
    if (count > rateLimit.maxRequests()) {
        throw new TooManyRequestsException("Rate limit exceeded");
    }
    return pjp.proceed();
}
```

### 9.2 Caching:
```java
@Around("@annotation(cacheable)")
public Object cache(ProceedingJoinPoint pjp, Cacheable cacheable) throws Throwable {
    String key = cacheable.value() + ":" + Arrays.toString(pjp.getArgs());
    Object cached = cacheManager.get(key);
    if (cached != null) return cached;

    Object result = pjp.proceed();
    cacheManager.put(key, result);
    return result;
}
```

### 9.3 Retry:
```java
@Around("@annotation(retryable)")
public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
    int maxAttempts = retryable.maxAttempts();
    Throwable lastException = null;

    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            return pjp.proceed();
        } catch (Exception ex) {
            lastException = ex;
            log.warn("Attempt {}/{} failed: {}", attempt, maxAttempts, ex.getMessage());
            Thread.sleep(retryable.backoffMs() * attempt);
        }
    }
    throw lastException;
}
```

---

## 10. Interview Questions & Answers

---

### Q1: What is AOP and what problem does it solve?
**A:** AOP separates cross-cutting concerns (logging, security, transactions, caching) from business logic. Without AOP, these concerns scatter across every method, creating code duplication and tight coupling. AOP centralizes them in Aspects applied declaratively.

---

### Q2: What is the difference between @Before, @After, @Around, @AfterReturning, and @AfterThrowing?
**A:**
- `@Before` — runs before method; cannot alter return value
- `@After` — runs always after (like `finally`)
- `@AfterReturning` — runs only on success; can inspect return value
- `@AfterThrowing` — runs only on exception; cannot suppress the exception
- `@Around` — wraps the method; can alter args, return value, suppress/translate exceptions, must call `proceed()`

---

### Q3: What is a JoinPoint vs ProceedingJoinPoint?
**A:** `JoinPoint` provides access to method signature, args, target object — used in `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`. `ProceedingJoinPoint` extends `JoinPoint` and adds `proceed()` to invoke the actual method — only available in `@Around`.

---

### Q4: What is the self-invocation problem in AOP?
**A:** When a Spring bean calls its own method internally using `this`, the call bypasses the AOP proxy. So any advice on the internally-called method is skipped. This is the most common `@Transactional` bug. Fix: inject `self` via `@Autowired`, use `AopContext.currentProxy()`, or switch to AspectJ weaving.

---

### Q5: When does Spring use JDK Proxy vs CGLIB proxy?
**A:** Spring uses JDK Dynamic Proxy when the target implements at least one interface. CGLIB is used when the target has no interface (subclasses the class). Spring Boot defaults to `proxyTargetClass=true` (always CGLIB) since Spring Boot 2.x to avoid issues with injection of concrete classes.

---

### Q6: What is weaving in AOP?
**A:** Weaving is the process of applying aspects to target objects. It can happen at:
- **Compile time** (AspectJ compiler): fastest, requires AspectJ tooling
- **Load time** (AspectJ LTW): JVM agent transforms classes on loading
- **Runtime** (Spring AOP): proxy-based, applied when bean is created; most common

---

### Q7: How does @Transactional work under the hood?
**A:** `@Transactional` is implemented as an AOP `@Around` advice. Spring wraps the target bean in a proxy. When a transactional method is called, the proxy intercepts it, calls `transactionManager.begin()`, invokes the real method via `proceed()`, then commits on success or rolls back on exception.

---

### Q8: What is the difference between Spring AOP and AspectJ?
**A:** Spring AOP is proxy-based, runtime-only, works only on Spring beans, and supports only method execution join points. AspectJ does compile-time/load-time weaving, works on any Java object, supports field, constructor, and method join points. Spring AOP uses AspectJ annotations but not AspectJ's weaver.

---

### Q9: Can you apply AOP to private methods?
**A:** No. Spring AOP works via proxies, and proxies can only intercept public (or protected, in CGLIB) method calls from outside the class. Private methods are never intercepted. For private method interception, use AspectJ compile-time weaving.

---

### Q10: How do you enable AOP in Spring?
**A:** Add `@EnableAspectJAutoProxy` to a `@Configuration` class (or use Spring Boot which enables it automatically). Create a class annotated with `@Aspect` and `@Component`. Define pointcut and advice methods.

---

### Q11: What happens if an @Around advice doesn't call proceed()?
**A:** The target method is never invoked. The aspect effectively blocks the call. This is intentional in security aspects (deny access) but a bug if unintentional.

---

### Q12: How do you get method arguments in an advice?
**A:**
```java
@Before("execution(* com.example.service.*.*(..))")
public void logArgs(JoinPoint jp) {
    Object[] args = jp.getArgs();         // all args
    Signature sig = jp.getSignature();    // method signature
    Object target = jp.getTarget();       // target object
}
```
For `@Around`, use `ProceedingJoinPoint`:
```java
Object result = pjp.proceed(modifiedArgs); // Can modify args!
```

---

*End of Chapter 2 — Spring AOP*
