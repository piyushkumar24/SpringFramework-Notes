# Chapter 9: Spring WebFlux — Reactive Programming
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [Reactive Programming Fundamentals](#1-reactive-programming-fundamentals)
2. [Project Reactor — Mono & Flux](#2-project-reactor--mono--flux)
3. [Spring WebFlux Architecture](#3-spring-webflux-architecture)
4. [Reactive Controllers](#4-reactive-controllers)
5. [Functional Endpoints (Router Functions)](#5-functional-endpoints-router-functions)
6. [Reactive Data Access — R2DBC](#6-reactive-data-access--r2dbc)
7. [WebClient (Reactive HTTP Client)](#7-webclient-reactive-http-client)
8. [Server-Sent Events & WebSocket](#8-server-sent-events--websocket)
9. [Error Handling in WebFlux](#9-error-handling-in-webflux)
10. [Testing WebFlux](#10-testing-webflux)
11. [WebFlux vs MVC — When to Use What](#11-webflux-vs-mvc--when-to-use-what)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Reactive Programming Fundamentals

**Reactive programming** is a paradigm for building **asynchronous, non-blocking, event-driven** systems that handle data streams with **backpressure**.

### Traditional Blocking vs Reactive:
```
Blocking (Thread per request):
Thread 1 → HTTP Request → [BLOCKED waiting for DB] → Response
Thread 2 → HTTP Request → [BLOCKED waiting for DB] → Response
...
Thread N → HTTP Request → [BLOCKED waiting for DB] → Response
(Limited by thread pool size)

Reactive (Event loop):
Event Loop → HTTP Request → schedules DB call → FREES UP
Event Loop → HTTP Request → schedules DB call → FREES UP
...
DB result arrives → Event Loop → continues processing → Response
(Handles millions of concurrent requests with few threads)
```

### The 4 Reactive Streams Interfaces:
```java
// Publisher — produces data
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// Subscriber — consumes data
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}

// Subscription — control channel (backpressure)
public interface Subscription {
    void request(long n);  // Request N items (backpressure)
    void cancel();
}

// Processor — both publisher and subscriber
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

### Backpressure:
Subscriber controls the rate of data flow by requesting only what it can handle:
```
Producer → (10,000 items/sec)  ← request(100) ← Consumer (100 items/sec)
Producer → (100 items/sec)     ← Backpressure applied!
```

---

## 2. Project Reactor — Mono & Flux

Project Reactor is Spring's reactive library, implementing Reactive Streams.

| | `Mono<T>` | `Flux<T>` |
|---|---|---|
| Items | 0 or 1 | 0 to N (unbounded) |
| Use case | Single value / empty | Multiple values / stream |
| Example | `findById()` | `findAll()` / stream of events |

### Creating Publishers:
```java
// Mono
Mono<String> mono1 = Mono.just("Hello");
Mono<String> mono2 = Mono.empty();  // No value
Mono<String> mono3 = Mono.error(new RuntimeException("Error"));
Mono<String> mono4 = Mono.fromCallable(() -> expensiveOperation());
Mono<String> mono5 = Mono.fromFuture(CompletableFuture.supplyAsync(() -> "async"));
Mono<Long> mono6 = Mono.delay(Duration.ofSeconds(1));

// Flux
Flux<String> flux1 = Flux.just("a", "b", "c");
Flux<Integer> flux2 = Flux.range(1, 10);  // 1 to 10
Flux<Long> flux3 = Flux.interval(Duration.ofMillis(100));  // Every 100ms
Flux<String> flux4 = Flux.fromIterable(List.of("a", "b", "c"));
Flux<String> flux5 = Flux.fromStream(Stream.of("a", "b", "c"));
Flux<String> flux6 = Flux.create(sink -> {
    sink.next("a");
    sink.next("b");
    sink.complete();
});
```

### Operators — Transforming Streams:
```java
// map — transform each item
Flux<String> names = Flux.just("alice", "bob", "charlie")
    .map(String::toUpperCase);  // ALICE, BOB, CHARLIE

// flatMap — async transform (returns Publisher)
Flux<User> users = Flux.just(1L, 2L, 3L)
    .flatMap(id -> userRepository.findById(id));  // async, unordered

// concatMap — like flatMap but preserves order
Flux<User> users = Flux.just(1L, 2L, 3L)
    .concatMap(id -> userRepository.findById(id));

// filter — keep matching items
Flux<Integer> evens = Flux.range(1, 10)
    .filter(n -> n % 2 == 0);  // 2, 4, 6, 8, 10

// reduce — aggregate
Mono<Integer> sum = Flux.range(1, 5)
    .reduce(0, Integer::sum);  // 15

// collect
Mono<List<String>> list = Flux.just("a", "b", "c")
    .collectList();

Mono<Map<String, User>> map = userFlux
    .collectMap(User::getId);

// zip — combine two publishers
Mono<String> result = Mono.zip(
    Mono.just("Hello"),
    Mono.just("World"),
    (a, b) -> a + " " + b  // Hello World
);

// merge — merge multiple publishers (unordered)
Flux<String> merged = Flux.merge(flux1, flux2, flux3);

// buffer — batch items
Flux<List<String>> batches = Flux.range(1, 20)
    .map(String::valueOf)
    .buffer(5);  // Lists of 5 items

// take/skip
Flux<Integer> first5 = Flux.range(1, 100).take(5);
Flux<Integer> after5 = Flux.range(1, 100).skip(5);

// distinct
Flux<String> distinct = Flux.just("a", "b", "a", "c").distinct();

// timeout
Mono<User> withTimeout = userRepository.findById(id)
    .timeout(Duration.ofSeconds(3));

// retry
Mono<User> withRetry = userRepository.findById(id)
    .retry(3)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));

// switchIfEmpty
Mono<User> withDefault = userRepository.findById(id)
    .switchIfEmpty(Mono.just(defaultUser));

// defaultIfEmpty
Mono<String> withDefault2 = Mono.<String>empty()
    .defaultIfEmpty("default value");
```

### Subscribing (triggering execution):
```java
// Reactive chains are LAZY — nothing executes until subscribed!

// Simple subscribe
Flux.just("a", "b", "c")
    .subscribe(
        item -> System.out.println("Next: " + item),    // onNext
        error -> System.err.println("Error: " + error), // onError
        () -> System.out.println("Complete!")            // onComplete
    );

// Block (for testing/debugging — NOT in reactive context!)
User user = userRepository.findById(1L).block();
List<User> users = userRepository.findAll().collectList().block();
```

### Schedulers (Threading):
```java
// Bounce to different thread pool
Flux.range(1, 100)
    .subscribeOn(Schedulers.boundedElastic())   // Blocking I/O
    .publishOn(Schedulers.parallel())            // CPU-bound work
    .map(this::processItem);

// Schedulers
Schedulers.immediate()        // Current thread
Schedulers.single()           // Single reusable thread
Schedulers.parallel()         // CPU count threads (compute)
Schedulers.boundedElastic()   // Elastic pool for blocking I/O
Schedulers.fromExecutorService(executor)  // Custom executor
```

---

## 3. Spring WebFlux Architecture

WebFlux is built on **Netty** (event loop) instead of Tomcat's thread-per-request model.

```
Request arrives
    │
Netty Event Loop Thread (non-blocking!)
    │
DispatcherHandler (like DispatcherServlet)
    │
HandlerMapping → finds handler (controller)
    │
HandlerAdapter → invokes handler
    │
Controller returns Mono/Flux (NOT blocking!)
    │
ResponseBodyResultHandler writes to response
    │
Event Loop continues (never blocked!)
```

---

## 4. Reactive Controllers

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;
    private final UserService userService;

    // Returns single or empty
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable Long id) {
        return userRepository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Returns stream of items
    @GetMapping
    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }

    // With pagination
    @GetMapping(params = "page")
    public Mono<Page<User>> getUsersPaged(
            @RequestParam int page,
            @RequestParam(defaultValue = "10") int size) {
        return userService.findAllPaged(PageRequest.of(page, size));
    }

    // Create
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> createUser(@RequestBody @Valid Mono<CreateUserRequest> requestMono) {
        return requestMono
            .flatMap(userService::create);
    }

    // Server-Sent Events stream
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<User> streamUsers() {
        return userRepository.findAll()
            .delayElements(Duration.ofMillis(100));  // Stream with delay
    }

    // Reactive file download
    @GetMapping("/export")
    public Mono<ResponseEntity<Resource>> exportUsers() {
        return userService.exportToCsv()
            .map(data -> ResponseEntity.ok()
                .header("Content-Disposition", "attachment; filename=users.csv")
                .contentType(MediaType.TEXT_PLAIN)
                .body((Resource) new ByteArrayResource(data)));
    }
}
```

---

## 5. Functional Endpoints (Router Functions)

Alternative to annotated controllers — more functional, composable:

```java
@Configuration
public class UserRouterConfig {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions
            .route(GET("/api/users").and(accept(APPLICATION_JSON)), handler::getAllUsers)
            .andRoute(GET("/api/users/{id}"), handler::getUserById)
            .andRoute(POST("/api/users").and(contentType(APPLICATION_JSON)), handler::createUser)
            .andRoute(PUT("/api/users/{id}"), handler::updateUser)
            .andRoute(DELETE("/api/users/{id}"), handler::deleteUser);
    }
}

@Component
@RequiredArgsConstructor
public class UserHandler {

    private final UserRepository userRepository;

    public Mono<ServerResponse> getAllUsers(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(userRepository.findAll(), User.class);
    }

    public Mono<ServerResponse> getUserById(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return userRepository.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .flatMap(req -> {
                User user = new User(req.getName(), req.getEmail());
                return userRepository.save(user);
            })
            .flatMap(user -> ServerResponse.created(
                URI.create("/api/users/" + user.getId()))
                .bodyValue(user));
    }
}
```

---

## 6. Reactive Data Access — R2DBC

R2DBC (Reactive Relational Database Connectivity) provides reactive access to relational databases.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
  data:
    r2dbc:
      repositories:
        enabled: true
```

```java
@Table("users")
@Data
public class User {
    @Id
    private Long id;
    private String name;
    private String email;
}

// Reactive repository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Flux<User> findByName(String name);
    Mono<User> findByEmail(String email);
    Mono<Long> countByRole(String role);

    @Query("SELECT * FROM users WHERE role = :role ORDER BY name")
    Flux<User> findByRoleOrdered(@Param("role") String role);
}

// Service usage
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;

    public Flux<User> findAll() {
        return userRepository.findAll();
    }

    public Mono<User> findById(Long id) {
        return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
    }

    public Mono<User> create(CreateUserRequest req) {
        User user = new User();
        user.setName(req.getName());
        user.setEmail(req.getEmail());
        return userRepository.save(user);
    }
}
```

### Reactive Transactions with R2DBC:
```java
@Service
@RequiredArgsConstructor
public class TransactionalService {

    private final ReactiveTransactionManager txManager;
    private final UserRepository userRepository;
    private final OrderRepository orderRepository;

    @Transactional  // Works reactively!
    public Mono<Order> placeOrder(Long userId, OrderRequest request) {
        return userRepository.findById(userId)
            .switchIfEmpty(Mono.error(new UserNotFoundException(userId)))
            .flatMap(user -> {
                Order order = new Order(userId, request.getTotal());
                return orderRepository.save(order);
            });
    }

    // Programmatic reactive transaction
    public Mono<User> createWithTransaction(CreateUserRequest req) {
        TransactionalOperator operator = TransactionalOperator.create(txManager);
        User user = new User(req.getName(), req.getEmail());
        return userRepository.save(user)
            .as(operator::transactional);
    }
}
```

---

## 7. WebClient (Reactive HTTP Client)

```java
@Service
public class PaymentServiceClient {

    private final WebClient webClient;

    public PaymentServiceClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://payment-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunction.ofRequestProcessor(req -> {
                log.info("Request: {} {}", req.method(), req.url());
                return Mono.just(req);
            }))
            .codecs(configurer -> configurer
                .defaultCodecs()
                .maxInMemorySize(16 * 1024 * 1024))  // 16MB
            .build();
    }

    public Mono<PaymentResult> processPayment(PaymentRequest request) {
        return webClient.post()
            .uri("/api/payments")
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatus.BAD_REQUEST::equals,
                res -> res.bodyToMono(ErrorResponse.class)
                    .flatMap(err -> Mono.error(new PaymentValidationException(err))))
            .onStatus(HttpStatus::is5xxServerError,
                res -> Mono.error(new PaymentServiceException("Payment service error")))
            .bodyToMono(PaymentResult.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .filter(ex -> ex instanceof PaymentServiceException));
    }
}
```

---

## 8. Server-Sent Events & WebSocket

### Server-Sent Events (SSE):
```java
@GetMapping(value = "/notifications/stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamNotifications(
        @RequestParam String userId) {
    return notificationService.getNotificationsForUser(userId)
        .map(notification -> ServerSentEvent.<String>builder()
            .id(notification.getId())
            .event("NOTIFICATION")
            .data(objectMapper.writeValueAsString(notification))
            .comment("User notification")
            .build())
        .mergeWith(Flux.interval(Duration.ofSeconds(30))
            .map(i -> ServerSentEvent.<String>builder()
                .event("heartbeat").data("ping").build()));
}
```

### WebSocket:
```java
@Component
public class WebSocketHandler implements WebSocketHandler {

    private final Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // Receive messages from client
        Mono<Void> input = session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .doOnNext(msg -> {
                log.info("Received: {}", msg);
                sink.tryEmitNext("Broadcast: " + msg);
            })
            .then();

        // Send messages to client
        Mono<Void> output = session.send(
            sink.asFlux().map(session::textMessage));

        return Mono.zip(input, output).then();
    }
}

@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketMapping(WebSocketHandler handler) {
        Map<String, WebSocketHandler> map = Map.of("/ws/chat", handler);
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1);
        return mapping;
    }

    @Bean
    public WebSocketHandlerAdapter webSocketHandlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

---

## 9. Error Handling in WebFlux

```java
// In controllers
@GetMapping("/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userRepository.findById(id)
        .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
}

// onErrorReturn — return default value
Mono<User> user = userRepository.findById(id)
    .onErrorReturn(UserNotFoundException.class, defaultUser);

// onErrorResume — fallback to another publisher
Mono<User> user = userRepository.findById(id)
    .onErrorResume(UserNotFoundException.class,
        ex -> cacheService.getCachedUser(id));

// onErrorMap — transform exception
Mono<User> user = userRepository.findById(id)
    .onErrorMap(DataAccessException.class,
        ex -> new ServiceException("Database error", ex));

// doOnError — side effect (logging) without handling
Mono<User> user = userRepository.findById(id)
    .doOnError(ex -> log.error("Error fetching user {}: {}", id, ex.getMessage()));

// Global error handling with @ControllerAdvice
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleNotFound(UserNotFoundException ex) {
        return Mono.just(ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage())));
    }
}

// Or WebExceptionHandler for low-level
@Component
@Order(-2)  // Higher priority than default
public class GlobalWebExceptionHandler implements WebExceptionHandler {

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        if (ex instanceof UserNotFoundException) {
            exchange.getResponse().setStatusCode(HttpStatus.NOT_FOUND);
            byte[] bytes = ("{\"error\": \"" + ex.getMessage() + "\"}").getBytes();
            return exchange.getResponse().writeWith(
                Mono.just(exchange.getResponse().bufferFactory().wrap(bytes)));
        }
        return Mono.error(ex);
    }
}
```

---

## 10. Testing WebFlux

### WebTestClient:
```java
@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private UserRepository userRepository;

    @Test
    void shouldReturnUser() {
        User user = new User(1L, "John", "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Mono.just(user));

        webTestClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .consumeWith(result -> {
                assertThat(result.getResponseBody().getName()).isEqualTo("John");
            });
    }

    @Test
    void shouldReturn404WhenNotFound() {
        when(userRepository.findById(99L)).thenReturn(Mono.empty());

        webTestClient.get()
            .uri("/api/users/99")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void shouldReturnAllUsers() {
        Flux<User> users = Flux.just(
            new User(1L, "Alice", "alice@example.com"),
            new User(2L, "Bob", "bob@example.com")
        );
        when(userRepository.findAll()).thenReturn(users);

        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(User.class)
            .hasSize(2)
            .contains(users.collectList().block().get(0));
    }
}
```

### StepVerifier (for Mono/Flux):
```java
@Test
void testFluxSequence() {
    Flux<Integer> flux = Flux.range(1, 5);

    StepVerifier.create(flux)
        .expectNext(1)
        .expectNext(2)
        .expectNext(3)
        .expectNext(4)
        .expectNext(5)
        .verifyComplete();
}

@Test
void testMonoError() {
    Mono<User> mono = Mono.error(new UserNotFoundException(99L));

    StepVerifier.create(mono)
        .expectErrorMatches(ex ->
            ex instanceof UserNotFoundException &&
            ex.getMessage().contains("99"))
        .verify();
}

@Test
void testWithVirtualTime() {  // Test time-based operators
    StepVerifier.withVirtualTime(
        () -> Flux.interval(Duration.ofHours(1)).take(3))
        .expectSubscription()
        .expectNoEvent(Duration.ofHours(1))
        .expectNext(0L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(1L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(2L)
        .verifyComplete();
}
```

---

## 11. WebFlux vs MVC — When to Use What

| Criteria | Spring MVC | Spring WebFlux |
|---|---|---|
| I/O type | Blocking | Non-blocking |
| Server | Tomcat (thread-per-request) | Netty (event loop) |
| Scalability | Limited by thread pool | Very high (few threads) |
| Programming model | Imperative | Reactive / Functional |
| Learning curve | Low | High |
| Ecosystem | Mature | Growing |
| Debugging | Easy | Hard (async stack traces) |
| Best for | CRUD apps, standard web | High-concurrency, streaming, microservices |

### Use WebFlux when:
- Streaming large amounts of data (SSE, WebSocket)
- High concurrency with many I/O calls
- Reactive database (MongoDB reactive, R2DBC)
- Microservices making many downstream calls

### Stick with MVC when:
- CRUD applications with relational DB (JPA)
- Team unfamiliar with reactive programming
- Complex transactions needed
- Existing MVC codebase

---

## 12. Interview Questions & Answers

---

### Q1: What is reactive programming and what problem does it solve?
**A:** Reactive programming is asynchronous, non-blocking, event-driven programming with backpressure. It solves the **C10K problem** — handling thousands of concurrent connections. Traditional blocking I/O wastes threads (one thread per request). Event-loop based reactive: one thread handles millions of requests since it never blocks — it registers a callback and moves on.

---

### Q2: What is the difference between Mono and Flux?
**A:** `Mono<T>` represents 0 or 1 item (like Optional but reactive). `Flux<T>` represents 0 to N items (like Stream but reactive). Both implement `Publisher`. Key difference: Flux can be an infinite stream (e.g., `Flux.interval()` fires forever). Both support the same operators but Flux has additional stream-specific ones (buffer, window, etc.).

---

### Q3: What is backpressure?
**A:** Backpressure is a mechanism where the subscriber signals to the publisher how many items it can handle (`subscription.request(n)`). This prevents fast producers from overwhelming slow consumers. Example: database produces 1M records but consumer can only process 100/sec — without backpressure, you'd get OOM. Reactive Streams build this into the protocol.

---

### Q4: What is the difference between flatMap and concatMap?
**A:** Both map each item to a Publisher and flatten results. `flatMap` subscribes to all inner publishers simultaneously (concurrent, unordered results). `concatMap` subscribes to one inner publisher at a time, preserving order (sequential). Use `flatMap` for performance when order doesn't matter; `concatMap` when order matters.

---

### Q5: Why shouldn't you call block() in reactive code?
**A:** `block()` blocks the current thread waiting for a result, defeating the purpose of reactive programming. In Netty's event loop, blocking can cause deadlocks (blocking the only thread that can process callbacks). Spring detects this and throws an exception. `block()` is acceptable in tests or at the application boundary (e.g., `main()`).

---

### Q6: What is the difference between subscribeOn and publishOn?
**A:** `subscribeOn` affects the thread where subscription (source) happens — changes the upstream thread. `publishOn` switches the thread for downstream operators from that point. Example: `subscribeOn(Schedulers.boundedElastic())` moves blocking DB call to I/O pool; `publishOn(Schedulers.parallel())` moves CPU-bound processing to compute pool.

---

### Q7: How does WebFlux handle exceptions?
**A:** Via operators: `onErrorReturn` (return default), `onErrorResume` (fallback publisher), `onErrorMap` (transform), `onErrorComplete` (complete without error). Globally via `@RestControllerAdvice` with `@ExceptionHandler` returning `Mono<ResponseEntity>`, or `WebExceptionHandler` for low-level handling. Operators like `switchIfEmpty` handle empty publishers.

---

### Q8: What is StepVerifier?
**A:** `StepVerifier` from `reactor-test` is the primary tool for testing reactive streams. It subscribes to a Publisher and lets you assert each event: `expectNext()`, `expectError()`, `verifyComplete()`. `withVirtualTime()` allows testing time-based operators instantly without actual sleeping.

---

*End of Chapter 9 — Spring WebFlux*
