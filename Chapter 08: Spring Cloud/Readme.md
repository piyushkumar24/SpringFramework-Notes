# Chapter 8: Spring Cloud — Microservices Architecture
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [Microservices Overview](#1-microservices-overview)
2. [Spring Cloud Components](#2-spring-cloud-components)
3. [Service Discovery — Eureka](#3-service-discovery--eureka)
4. [API Gateway — Spring Cloud Gateway](#4-api-gateway--spring-cloud-gateway)
5. [Centralized Configuration — Spring Cloud Config](#5-centralized-configuration--spring-cloud-config)
6. [Load Balancing — Spring Cloud LoadBalancer](#6-load-balancing--spring-cloud-loadbalancer)
7. [Circuit Breaker — Resilience4j](#7-circuit-breaker--resilience4j)
8. [Distributed Tracing — Micrometer + Zipkin](#8-distributed-tracing--micrometer--zipkin)
9. [Message-Driven Microservices](#9-message-driven-microservices)
10. [Feign Client](#10-feign-client)
11. [Saga Pattern & Distributed Transactions](#11-saga-pattern--distributed-transactions)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Microservices Overview

Microservices is an architectural style where an application is decomposed into small, independently deployable services.

### Monolith vs Microservices:
| | Monolith | Microservices |
|---|---|---|
| Deployment | Single unit | Independent per service |
| Scaling | Scale everything | Scale only what's needed |
| Tech stack | Single | Polyglot (any language) |
| Team size | Large team | Small teams per service |
| Data | Single DB | DB per service |
| Failure | Single point of failure | Isolated failures |
| Complexity | Simple | High (distributed) |

### Microservices Challenges:
- **Distributed system complexity** (latency, partial failures)
- **Service discovery** (how services find each other)
- **Configuration management** (across many services)
- **Distributed tracing** (trace requests across services)
- **Data consistency** (no ACID across services)
- **Network reliability** (fallbacks, retries, circuit breakers)

### Spring Cloud Architecture:
```
Client
  │
  ▼
API Gateway (Spring Cloud Gateway)
  │
  ├── Service Discovery (Eureka)
  │
  ├── Config Server (Spring Cloud Config)
  │
  ├── User Service ──┐
  ├── Order Service ─┤── Message Broker (Kafka/RabbitMQ)
  ├── Payment Service─┘
  │
  └── Observability (Zipkin, Prometheus, Grafana)
```

---

## 2. Spring Cloud Components

| Component | Purpose | Technology |
|---|---|---|
| Service Registry | Service discovery | Netflix Eureka, Consul |
| API Gateway | Single entry point, routing, filtering | Spring Cloud Gateway |
| Config Server | Centralized configuration | Spring Cloud Config |
| Load Balancer | Client-side load balancing | Spring Cloud LoadBalancer |
| Circuit Breaker | Fault tolerance | Resilience4j |
| Distributed Tracing | Request tracing across services | Micrometer + Zipkin/Jaeger |
| Messaging | Async communication | Spring Cloud Stream (Kafka/RabbitMQ) |
| Service Mesh | Advanced networking | Istio (not Spring-specific) |

---

## 3. Service Discovery — Eureka

Services register themselves with Eureka and discover other services by name.

### Eureka Server:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# eureka-server/application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false    # Don't register itself
    fetchRegistry: false         # Don't fetch its own registry
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: eureka-server
```

### Eureka Client (any microservice):
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableDiscoveryClient  // Enables service registration
public class UserServiceApplication { ... }
```

```yaml
# user-service/application.yml
spring:
  application:
    name: user-service  # Service name in Eureka

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30    # Heartbeat interval
    lease-expiration-duration-in-seconds: 90 # Removed if no heartbeat
```

---

## 4. API Gateway — Spring Cloud Gateway

Single entry point for all client requests — routing, filtering, security, rate limiting.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### Route Configuration:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service          # lb:// = load balanced
          predicates:
            - Path=/api/users/**          # URL predicate
            - Method=GET,POST
            - Header=X-API-Version, 2     # Header predicate
          filters:
            - StripPrefix=1               # Remove first path segment
            - AddRequestHeader=X-Source, gateway
            - RequestRateLimiter(redis-rate-limiter.replenishRate=10,
                                  redis-rate-limiter.burstCapacity=20)

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/orders

        - id: auth-service
          uri: lb://auth-service
          predicates:
            - Path=/api/auth/**
```

### Custom Gateway Filter:
```java
@Component
public class JwtGatewayFilter implements GlobalFilter, Ordered {

    @Autowired
    private JwtService jwtService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Skip public paths
        if (isPublicPath(request.getPath().toString())) {
            return chain.filter(exchange);
        }

        // Validate JWT
        String token = extractToken(request);
        if (token == null || !jwtService.isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Propagate user info to downstream services
        String userId = jwtService.extractUserId(token);
        ServerHttpRequest modified = request.mutate()
            .header("X-User-Id", userId)
            .header("X-User-Role", jwtService.extractRole(token))
            .build();

        return chain.filter(exchange.mutate().request(modified).build());
    }

    @Override
    public int getOrder() { return -100; }  // High priority
}
```

### Rate Limiting with Redis:
```java
@Configuration
public class RateLimiterConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ).switchIfEmpty(Mono.just("anonymous"));
    }
}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api-route
          uri: lb://api-service
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10   # req/sec
                redis-rate-limiter.burstCapacity: 20    # burst
                key-resolver: "#{@userKeyResolver}"
```

---

## 5. Centralized Configuration — Spring Cloud Config

One place to manage configuration for all microservices.

### Config Server:
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
```

```yaml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'   # Subdirectory per service
          username: ${GIT_USERNAME}
          password: ${GIT_TOKEN}
        # Or file-based:
        # native:
        #   search-locations: classpath:/configs
```

### Config Client (any microservice):
```yaml
# bootstrap.yml (loaded before application.yml)
spring:
  application:
    name: user-service     # Fetches user-service.yml from config repo
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      uri: http://localhost:8888
      profile: dev
      label: main
      fail-fast: true  # Fail startup if config server unavailable
      retry:
        max-attempts: 6
        initial-interval: 1000
```

### Dynamic Refresh:
```java
@RefreshScope  // Bean re-created when /actuator/refresh called
@Component
public class DynamicConfig {
    @Value("${feature.enabled:false}")
    private boolean featureEnabled;
}
```

```bash
# Trigger refresh (after config change in Git)
curl -X POST http://localhost:8080/actuator/refresh

# Or use Spring Cloud Bus to broadcast to all instances:
curl -X POST http://localhost:8888/actuator/busrefresh
```

---

## 6. Load Balancing — Spring Cloud LoadBalancer

Client-side load balancing — service instances discovered from Eureka.

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced  // Enables client-side load balancing
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final WebClient.Builder webClientBuilder;

    public UserDTO getUserForOrder(Long userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://user-service/api/users/{id}", userId)  // Service name, not host
            .retrieve()
            .bodyToMono(UserDTO.class)
            .block();
    }
}
```

### Custom Load Balancer Strategy:
```java
@Configuration
public class CustomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
        // Or: RoundRobinLoadBalancer (default)
    }
}
```

---

## 7. Circuit Breaker — Resilience4j

Prevents cascade failures by "opening the circuit" when a service is down.

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

### Circuit Breaker States:
```
CLOSED → calls pass through (normal operation)
  ↓ failure threshold exceeded
OPEN → calls fail fast (circuit broken) → goes to fallback
  ↓ after wait duration
HALF_OPEN → limited calls to test recovery
  ↓ successful calls
CLOSED (recovered)
```

### Configuration:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        register-health-indicator: true
        failure-rate-threshold: 50           # Open at 50% failures
        slow-call-rate-threshold: 80         # Also count slow calls
        slow-call-duration-threshold: 2s     # Slow = > 2 seconds
        minimum-number-of-calls: 10          # Need 10 calls before evaluating
        wait-duration-in-open-state: 10s     # Wait 10s before HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 3
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20              # Last 20 calls

  retry:
    instances:
      userService:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2   # 500ms, 1s, 2s
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException

  ratelimiter:
    instances:
      userService:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 0s

  bulkhead:
    instances:
      userService:
        max-concurrent-calls: 20
        max-wait-duration: 0ms
```

### Using in Code:
```java
@Service
@RequiredArgsConstructor
public class UserServiceClient {

    private final WebClient.Builder webClientBuilder;

    // All resilience patterns combined
    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @Retry(name = "userService")
    @RateLimiter(name = "userService")
    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public UserDTO getUser(Long userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://user-service/api/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserDTO.class)
            .block();
    }

    // Fallback must have same return type + Throwable param
    public UserDTO getUserFallback(Long userId, Throwable ex) {
        log.error("User service down, using fallback for userId: {}", userId, ex);
        return UserDTO.builder()
            .id(userId)
            .name("Unknown User")
            .build();
    }
}
```

---

## 8. Distributed Tracing — Micrometer + Zipkin

Track a request as it flows through multiple microservices.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0    # 100% sampling (use 0.1 in prod)
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

spring:
  application:
    name: order-service
```

### Trace IDs propagate automatically:
```
Request enters API Gateway
  TraceId: abc123, SpanId: span1
      │
  Order Service receives request
    TraceId: abc123 (same!), SpanId: span2
        │
    User Service called by Order Service
      TraceId: abc123 (same!), SpanId: span3
```

All these spans are linked by TraceId in Zipkin UI.

### Manual Tracing:
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;

    public Order placeOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("place-order").start();
        span.tag("userId", request.getUserId().toString());
        span.tag("orderAmount", request.getTotal().toString());

        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            return processOrder(request);
        } catch (Exception ex) {
            span.error(ex);
            throw ex;
        } finally {
            span.end();
        }
    }
}
```

---

## 9. Message-Driven Microservices

### Spring Cloud Stream (Kafka):
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        orderPlaced-out-0:
          destination: order-events
          content-type: application/json
        orderPlaced-in-0:
          destination: order-events
          group: payment-service-group
          content-type: application/json
```

```java
// Producer
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final StreamBridge streamBridge;

    public void publishOrderPlaced(Order order) {
        OrderPlacedEvent event = OrderPlacedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .total(order.getTotal())
            .timestamp(Instant.now())
            .build();

        streamBridge.send("orderPlaced-out-0", event);
        log.info("Published OrderPlacedEvent: {}", event);
    }
}

// Consumer
@Service
public class PaymentEventHandler {

    @Bean
    public Consumer<OrderPlacedEvent> orderPlaced() {
        return event -> {
            log.info("Processing payment for order: {}", event.getOrderId());
            paymentService.processPayment(event);
        };
    }
}
```

### Spring AMQP (RabbitMQ):
```java
@Configuration
public class RabbitConfig {

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "dead-letter-exchange")
            .withArgument("x-message-ttl", 60000)
            .build();
    }

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }

    @Bean
    public Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with("order.#");
    }
}

@Service
public class OrderMessageSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendOrderEvent(OrderEvent event) {
        rabbitTemplate.convertAndSend("order.exchange", "order.placed", event);
    }
}

@Service
public class OrderMessageReceiver {

    @RabbitListener(queues = "order.queue")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Received order event: {}", event);
        paymentService.process(event);
    }
}
```

---

## 10. Feign Client

Declarative REST client that integrates with Eureka for service discovery.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication { }

// Declare a client interface
@FeignClient(
    name = "user-service",       // Eureka service name
    path = "/api/v1",
    fallback = UserClientFallback.class,
    configuration = FeignConfig.class
)
public interface UserFeignClient {

    @GetMapping("/users/{id}")
    UserDTO getUserById(@PathVariable("id") Long id);

    @GetMapping("/users")
    Page<UserDTO> getAllUsers(@RequestParam int page,
                              @RequestParam int size);

    @PostMapping("/users")
    UserDTO createUser(@RequestBody CreateUserRequest request);
}

// Fallback
@Component
public class UserClientFallback implements UserFeignClient {

    @Override
    public UserDTO getUserById(Long id) {
        return UserDTO.builder().id(id).name("Unknown").build();
    }

    @Override
    public Page<UserDTO> getAllUsers(int page, int size) {
        return Page.empty();
    }

    @Override
    public UserDTO createUser(CreateUserRequest request) {
        throw new ServiceUnavailableException("User service unavailable");
    }
}

// Configuration (request interceptor for auth)
public class FeignConfig {

    @Bean
    public RequestInterceptor authInterceptor() {
        return requestTemplate -> {
            String token = SecurityContextHolder.getContext()
                .getAuthentication().getCredentials().toString();
            requestTemplate.header("Authorization", "Bearer " + token);
        };
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            if (response.status() == 404) {
                return new UserNotFoundException("User not found");
            }
            return new Default().decode(methodKey, response);
        };
    }
}
```

---

## 11. Saga Pattern & Distributed Transactions

In microservices, you can't have ACID across services. Use **Saga** for distributed transactions.

### Choreography-based Saga:
```
Order Service → emit OrderCreated event
  → Payment Service listens → process payment → emit PaymentCompleted
    → Inventory Service listens → reserve stock → emit StockReserved
      → Order Service listens → confirm order

On failure:
  → Payment Service fails → emit PaymentFailed
    → Order Service listens → cancel order
```

### Orchestration-based Saga:
```java
@Component
public class PlaceOrderSaga {

    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final OrderService orderService;

    public void execute(OrderRequest request) {
        Order order = orderService.create(request);

        try {
            paymentService.processPayment(order);
        } catch (PaymentException e) {
            orderService.cancel(order.getId(), "Payment failed");
            throw e;
        }

        try {
            inventoryService.reserveStock(order);
        } catch (InventoryException e) {
            paymentService.refundPayment(order.getId());  // Compensate
            orderService.cancel(order.getId(), "Stock unavailable");
            throw e;
        }

        orderService.confirm(order.getId());
    }
}
```

---

## 12. Interview Questions & Answers

---

### Q1: What is service discovery and why is it needed in microservices?
**A:** Services run on dynamic IPs/ports (especially in Kubernetes). Hardcoding addresses is brittle. Service discovery: services **register** with a registry (Eureka) on startup, providing name + IP + port. **Clients query the registry** by service name to get current instances. Eureka maintains heartbeats to detect failures. This enables auto-scaling and zero-config routing.

---

### Q2: What is an API Gateway and what does it do?
**A:** A single entry point for all clients. Responsibilities: routing (forward to right service), authentication/authorization, rate limiting, request/response transformation, load balancing, SSL termination, CORS handling, logging. Spring Cloud Gateway uses reactive (Netty) stack — non-blocking. Eliminates client-to-service coupling.

---

### Q3: What is a circuit breaker pattern?
**A:** Prevents cascade failures. When calls to a service fail repeatedly, the circuit "opens" — subsequent calls fail fast without hitting the downstream service (which is likely down). After a wait period, it transitions to HALF_OPEN to test recovery. States: CLOSED (normal) → OPEN (failing fast) → HALF_OPEN (testing) → CLOSED. Resilience4j is the standard implementation.

---

### Q4: What is the difference between choreography and orchestration in Saga?
**A:** **Choreography**: Each service reacts to events from other services — no central coordinator. Decoupled but harder to track flow, risk of cyclic dependencies. **Orchestration**: A central orchestrator (saga manager) tells each service what to do and handles compensating transactions on failure. More complex orchestrator but clearer flow and easier debugging. Both need compensating transactions for rollback.

---

### Q5: How do you handle distributed configuration in microservices?
**A:** Spring Cloud Config Server stores configuration in Git. Services fetch config at startup and can refresh dynamically via `/actuator/refresh` or `Spring Cloud Bus` for broadcasting across all instances. Supports profiles (dev/prod), encryption of secrets, and fallback to local config.

---

### Q6: How does Feign differ from RestTemplate/WebClient?
**A:** Feign is declarative — you define an interface with Spring MVC annotations; Feign generates the implementation. Integrates automatically with Eureka (service name routing), load balancing, circuit breaker, and interceptors. RestTemplate/WebClient are imperative — you write the HTTP calls. Feign is cleaner for service-to-service calls; WebClient is better for reactive programming.

---

### Q7: What is distributed tracing and how does it work?
**A:** A request spanning multiple services gets a **TraceId** (unique per request) and each service creates a **Span** (unit of work). Spans are linked by TraceId. Each service propagates TraceId via HTTP headers (`X-B3-TraceId`). A tracing backend (Zipkin, Jaeger) collects and visualizes the full request flow. Micrometer Tracing provides the Spring abstraction.

---

### Q8: What is a bulkhead pattern?
**A:** Isolates failures — limits concurrent calls to a service so a slow/failing service doesn't exhaust all threads. Named after ship bulkheads that prevent water from flooding the whole ship. Resilience4j provides Semaphore-based (count concurrent calls) and Thread-pool-based bulkheads.

---

### Q9: How do you implement inter-service security?
**A:** Options:
1. **JWT propagation** — Gateway validates JWT, adds user info as headers, downstream services trust headers
2. **mTLS (mutual TLS)** — Services authenticate via certificates (service mesh: Istio)
3. **OAuth2 Token Relay** — Gateway relays user's OAuth2 token to downstream
4. **Service-to-service tokens** — Each service gets its own client credentials for machine-to-machine auth

---

### Q10: What is the Strangler Fig pattern?
**A:** Gradual migration from monolith to microservices. A proxy (API Gateway) sits in front of the monolith. New features are built as microservices; existing features are gradually extracted. The gateway routes new features to microservices and old features to the monolith. Over time, the monolith "dies" like a tree strangled by the fig plant.

---

*End of Chapter 8 — Spring Cloud*
