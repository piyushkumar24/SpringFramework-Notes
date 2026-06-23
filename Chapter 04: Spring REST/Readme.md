# Chapter 4: Spring REST — Building RESTful APIs
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [REST Fundamentals](#1-rest-fundamentals)
2. [Building REST APIs with Spring](#2-building-rest-apis-with-spring)
3. [ResponseEntity & HTTP Status Codes](#3-responseentity--http-status-codes)
4. [JSON Handling with Jackson](#4-json-handling-with-jackson)
5. [HATEOAS & Hypermedia](#5-hateoas--hypermedia)
6. [API Versioning Strategies](#6-api-versioning-strategies)
7. [RestTemplate & WebClient](#7-resttemplate--webclient)
8. [Exception Handling for REST](#8-exception-handling-for-rest)
9. [OpenAPI / Swagger Documentation](#9-openapi--swagger-documentation)
10. [REST API Best Practices](#10-rest-api-best-practices)
11. [Interview Questions & Answers](#11-interview-questions--answers)

---

## 1. REST Fundamentals

**REST (Representational State Transfer)** is an architectural style for distributed hypermedia systems.

### 6 Constraints of REST:
1. **Client-Server** — Separation of UI from data storage
2. **Stateless** — Each request contains all info; server stores no client state
3. **Cacheable** — Responses must define if they're cacheable
4. **Uniform Interface** — Standard HTTP methods, self-descriptive messages
5. **Layered System** — Client doesn't know if it's talking to end server or intermediary
6. **Code on Demand** (optional) — Server can send executable code

### HTTP Methods:
| Method | Usage | Idempotent? | Safe? | Body? |
|---|---|---|---|---|
| GET | Read resource | ✅ | ✅ | No |
| POST | Create resource | ❌ | ❌ | Yes |
| PUT | Replace resource | ✅ | ❌ | Yes |
| PATCH | Partial update | ❌ | ❌ | Yes |
| DELETE | Delete resource | ✅ | ❌ | No |
| HEAD | Like GET, no body | ✅ | ✅ | No |
| OPTIONS | Get allowed methods | ✅ | ✅ | No |

> **Idempotent**: Multiple identical requests have same effect as single request.
> **Safe**: Request doesn't modify state (read-only).

### HTTP Status Codes:
| Range | Category | Common Codes |
|---|---|---|
| 1xx | Informational | 100 Continue |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable |
| 5xx | Server Error | 500 Internal Server Error, 503 Service Unavailable |

### Resource Naming Conventions:
```
GET    /api/users           → list all users
POST   /api/users           → create user
GET    /api/users/{id}      → get user by id
PUT    /api/users/{id}      → replace user
PATCH  /api/users/{id}      → partial update
DELETE /api/users/{id}      → delete user

GET    /api/users/{id}/orders       → user's orders
POST   /api/users/{id}/orders       → create order for user
GET    /api/users/{id}/orders/{oid} → specific order

// Filters/search → query params
GET    /api/users?status=active&role=admin
GET    /api/products?category=electronics&minPrice=100&sort=price,asc&page=0&size=20
```

---

## 2. Building REST APIs with Spring

### Complete CRUD REST Controller:
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Slf4j
public class UserController {

    private final UserService userService;
    private final UserMapper userMapper;

    // GET all with pagination
    @GetMapping
    public ResponseEntity<Page<UserResponse>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "createdAt,desc") String[] sort) {

        Sort sorting = Sort.by(Arrays.stream(sort)
            .map(s -> s.split(","))
            .map(arr -> new Sort.Order(
                arr.length > 1 && arr[1].equalsIgnoreCase("desc")
                    ? Sort.Direction.DESC : Sort.Direction.ASC, arr[0]))
            .collect(Collectors.toList()));

        Pageable pageable = PageRequest.of(page, size, sorting);
        Page<UserResponse> users = userService.findAll(pageable)
            .map(userMapper::toResponse);

        return ResponseEntity.ok(users);
    }

    // GET single
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        return userService.findById(id)
            .map(userMapper::toResponse)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    // POST create
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @RequestBody @Valid CreateUserRequest request,
            UriComponentsBuilder ucb) {

        User user = userService.create(request);
        URI location = ucb.path("/api/v1/users/{id}")
            .buildAndExpand(user.getId()).toUri();
        return ResponseEntity.created(location).body(userMapper.toResponse(user));
    }

    // PUT replace
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> replaceUser(
            @PathVariable Long id,
            @RequestBody @Valid UpdateUserRequest request) {

        User updated = userService.replace(id, request);
        return ResponseEntity.ok(userMapper.toResponse(updated));
    }

    // PATCH partial update
    @PatchMapping("/{id}")
    public ResponseEntity<UserResponse> patchUser(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {

        User updated = userService.patch(id, updates);
        return ResponseEntity.ok(userMapper.toResponse(updated));
    }

    // DELETE
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }

    // Search
    @GetMapping("/search")
    public ResponseEntity<List<UserResponse>> searchUsers(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String email,
            @RequestParam(required = false) String role) {

        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .name(name).email(email).role(role).build();
        List<UserResponse> users = userService.search(criteria)
            .stream().map(userMapper::toResponse).toList();
        return ResponseEntity.ok(users);
    }
}
```

---

## 3. ResponseEntity & HTTP Status Codes

`ResponseEntity<T>` gives full control over HTTP response: status, headers, body.

```java
// 200 OK with body
return ResponseEntity.ok(user);

// 201 Created with Location header
URI location = URI.create("/api/users/" + user.getId());
return ResponseEntity.created(location).body(user);

// 204 No Content
return ResponseEntity.noContent().build();

// 400 Bad Request
return ResponseEntity.badRequest().body(errorResponse);

// 404 Not Found
return ResponseEntity.notFound().build();

// Custom status
return ResponseEntity.status(HttpStatus.CONFLICT)
    .header("X-Custom-Header", "value")
    .body(conflictResponse);

// Conditional response (ETag / Last-Modified)
return ResponseEntity.ok()
    .eTag(etag)
    .lastModified(Instant.now())
    .body(resource);
```

### Custom Error Response:
```java
@Data
@Builder
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private List<ValidationError> validationErrors;

    @Data
    @AllArgsConstructor
    public static class ValidationError {
        private String field;
        private String message;
    }
}
```

---

## 4. JSON Handling with Jackson

### Jackson Annotations:
```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)  // Ignore unknown JSON fields
public class UserResponse {

    @JsonProperty("user_id")           // Map to different JSON key
    private Long id;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;

    @JsonIgnore                         // Never serialize this field
    private String password;

    @JsonInclude(Include.NON_NULL)      // Skip null fields
    private String middleName;

    @JsonSerialize(using = CustomSerializer.class)
    private Status status;

    @JsonDeserialize(using = CustomDeserializer.class)
    private SomeComplexType complexField;
}
```

### ObjectMapper Configuration:
```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return Jackson2ObjectMapperBuilder.json()
            .featuresToDisable(
                SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,
                DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES
            )
            .featuresToEnable(
                SerializationFeature.INDENT_OUTPUT,
                MapperFeature.DEFAULT_VIEW_INCLUSION
            )
            .serializationInclusion(JsonInclude.Include.NON_NULL)
            .modules(new JavaTimeModule(), new Jdk8Module())
            .build();
    }
}
```

### JSON Views (for different serialization contexts):
```java
public class Views {
    public interface Public {}
    public interface Internal extends Public {}
}

public class User {
    @JsonView(Views.Public.class)
    private Long id;

    @JsonView(Views.Public.class)
    private String name;

    @JsonView(Views.Internal.class)  // Only shown in internal view
    private String ssn;
}

@GetMapping("/users/{id}/public")
@JsonView(Views.Public.class)
public User getPublicUser(@PathVariable Long id) { ... }

@GetMapping("/users/{id}/internal")
@JsonView(Views.Internal.class)
public User getInternalUser(@PathVariable Long id) { ... }
```

---

## 5. HATEOAS & Hypermedia

HATEOAS (Hypermedia As The Engine Of Application State) — responses include links to related actions.

```java
// Add spring-hateoas dependency
@RestController
public class UserController {

    @GetMapping("/api/users/{id}")
    public EntityModel<UserResponse> getUser(@PathVariable Long id) {
        UserResponse user = userService.findById(id);

        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserController.class).getUserOrders(id)).withRel("orders"),
            linkTo(methodOn(UserController.class).deleteUser(id)).withRel("delete")
        );
    }
}
```

Response:
```json
{
  "id": 1,
  "name": "John Doe",
  "_links": {
    "self": { "href": "http://localhost:8080/api/users/1" },
    "orders": { "href": "http://localhost:8080/api/users/1/orders" },
    "delete": { "href": "http://localhost:8080/api/users/1" }
  }
}
```

---

## 6. API Versioning Strategies

### Strategy 1: URL Path Versioning (most common, clearest)
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { }
```

### Strategy 2: Request Parameter Versioning
```java
@GetMapping(value = "/users", params = "version=1")
public List<UserV1> getUsersV1() { }

@GetMapping(value = "/users", params = "version=2")
public List<UserV2> getUsersV2() { }
// GET /api/users?version=2
```

### Strategy 3: Header Versioning
```java
@GetMapping(value = "/users", headers = "X-API-Version=1")
public List<UserV1> getUsersV1() { }

@GetMapping(value = "/users", headers = "X-API-Version=2")
public List<UserV2> getUsersV2() { }
```

### Strategy 4: Accept Header (Media Type) Versioning
```java
@GetMapping(value = "/users",
            produces = "application/vnd.company.app-v1+json")
public List<UserV1> getUsersV1() { }

@GetMapping(value = "/users",
            produces = "application/vnd.company.app-v2+json")
public List<UserV2> getUsersV2() { }
// Accept: application/vnd.company.app-v2+json
```

| Strategy | Pros | Cons |
|---|---|---|
| URL Path | Visible, easy to test | Breaks REST purist view |
| Request Param | Simple | Not RESTful |
| Header | Clean URL | Hard to test in browser |
| Media Type | Most RESTful | Complex |

---

## 7. RestTemplate & WebClient

### RestTemplate (Synchronous — legacy):
```java
@Service
public class ExternalApiService {

    private final RestTemplate restTemplate;
    private final String apiUrl = "https://api.example.com";

    public ExternalApiService(RestTemplateBuilder builder) {
        this.restTemplate = builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }

    // GET
    public User getUser(Long id) {
        return restTemplate.getForObject(apiUrl + "/users/{id}", User.class, id);
    }

    // GET with response entity
    public ResponseEntity<User> getUserEntity(Long id) {
        return restTemplate.getForEntity(apiUrl + "/users/{id}", User.class, id);
    }

    // POST
    public User createUser(CreateUserRequest request) {
        return restTemplate.postForObject(apiUrl + "/users", request, User.class);
    }

    // PUT
    public void updateUser(Long id, UpdateUserRequest request) {
        restTemplate.put(apiUrl + "/users/{id}", request, id);
    }

    // DELETE
    public void deleteUser(Long id) {
        restTemplate.delete(apiUrl + "/users/{id}", id);
    }

    // Exchange (full control)
    public ResponseEntity<List<User>> getUsersList() {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + getToken());
        headers.setAccept(List.of(MediaType.APPLICATION_JSON));

        HttpEntity<?> entity = new HttpEntity<>(headers);
        ParameterizedTypeReference<List<User>> responseType =
            new ParameterizedTypeReference<>() {};

        return restTemplate.exchange(
            apiUrl + "/users", HttpMethod.GET, entity, responseType);
    }
}
```

### WebClient (Reactive — preferred in modern Spring):
```java
@Service
public class ReactiveApiService {

    private final WebClient webClient;

    public ReactiveApiService(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions.basicAuthentication("user", "pass"))
            .build();
    }

    // Synchronous (blocking)
    public User getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .block();  // Block for synchronous result
    }

    // Reactive (non-blocking)
    public Mono<User> getUserReactive(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatus.NOT_FOUND::equals,
                res -> Mono.error(new UserNotFoundException(id)))
            .bodyToMono(User.class);
    }

    // POST
    public Mono<User> createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(User.class);
    }

    // List (Flux)
    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(User.class);
    }
}
```

---

## 8. Exception Handling for REST

### Problem Details (RFC 7807) — Spring 6+:
```java
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setProperty("resourceId", ex.getResourceId());
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

Response:
```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with id 42 not found",
  "resourceId": 42,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 9. OpenAPI / Swagger Documentation

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```java
@RestController
@Tag(name = "Users", description = "User management API")
@RequestMapping("/api/v1/users")
public class UserController {

    @Operation(summary = "Get user by ID",
               description = "Returns a single user by their ID")
    @ApiResponse(responseCode = "200", description = "User found",
        content = @Content(schema = @Schema(implementation = UserResponse.class)))
    @ApiResponse(responseCode = "404", description = "User not found")
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(
            @Parameter(description = "User ID", required = true)
            @PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

```properties
# Swagger UI at http://localhost:8080/swagger-ui.html
# API docs at http://localhost:8080/v3/api-docs
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.enabled=true
```

---

## 10. REST API Best Practices

1. **Use nouns, not verbs** in URLs (`/users` not `/getUsers`)
2. **Use plural nouns** for collections (`/users` not `/user`)
3. **Return appropriate status codes** (201 for create, 204 for delete, etc.)
4. **Version your API** from day one
5. **Use HTTPS** always
6. **Implement pagination** for list endpoints
7. **Use consistent error format** across all endpoints
8. **Include request IDs** for traceability (`X-Request-ID`)
9. **Use ETags** for caching support
10. **Document with OpenAPI** (Swagger)
11. **Implement rate limiting**
12. **Use JSON:API or HATEOAS** for discoverability
13. **Keep backward compatibility** — never remove fields, add new optional ones

---

## 11. Interview Questions & Answers

---

### Q1: What is the difference between PUT and PATCH?
**A:** `PUT` replaces the entire resource with the request body (full update). If you omit a field, it becomes null/default. `PATCH` performs a partial update — only the fields in the request are changed. PUT is idempotent; PATCH technically isn't (though usually is in practice).

---

### Q2: What is the difference between @RequestBody and @ModelAttribute?
**A:** `@RequestBody` reads the HTTP request body and deserializes it from JSON/XML to a Java object (uses `HttpMessageConverter`). `@ModelAttribute` binds form fields (key-value pairs) from request parameters or form data to a Java object (uses data binding).

---

### Q3: What is RestTemplate and why is WebClient preferred?
**A:** `RestTemplate` is Spring's synchronous HTTP client — blocking, one thread per request. `WebClient` (Spring WebFlux) is non-blocking and reactive — handles concurrency with fewer threads. For new projects, `WebClient` is preferred even in non-reactive apps. `RestTemplate` is deprecated since Spring 5.

---

### Q4: How do you handle errors in REST API?
**A:** Use `@RestControllerAdvice` with `@ExceptionHandler` methods for global error handling. Return consistent error response format (RFC 7807 ProblemDetail in Spring 6). Always include: status code, error message, timestamp, request path. Use domain-specific exception hierarchy.

---

### Q5: How do you implement pagination in Spring REST?
**A:** Use `Pageable` (Spring Data) as a controller method parameter. Spring MVC automatically binds `page`, `size`, `sort` query params:
```java
@GetMapping("/users")
public Page<User> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable);
}
// GET /api/users?page=0&size=10&sort=name,asc
```

---

### Q6: What are the differences between idempotent and safe HTTP methods?
**A:** **Safe** methods don't modify state (GET, HEAD, OPTIONS). **Idempotent** means making N identical requests has same effect as 1 request (GET, PUT, DELETE, HEAD, OPTIONS). POST is neither safe nor idempotent. PATCH is not idempotent (applying the same patch twice may differ).

---

### Q7: How do you implement CORS in Spring?
**A:** Three ways:
1. `@CrossOrigin` annotation on controller/method
2. `WebMvcConfigurer.addCorsMappings()` for global config
3. `CorsFilter` bean for filter-level CORS
Spring Boot also supports `spring.mvc.cors.*` properties.

---

### Q8: What is HATEOAS and when to use it?
**A:** HATEOAS includes hyperlinks in responses that guide clients to available actions. Useful for complex APIs where clients should discover operations dynamically rather than hardcoding URLs. Spring provides the `spring-hateoas` library with `EntityModel`, `CollectionModel`, and `linkTo(methodOn(...))` helpers.

---

### Q9: How do you secure a REST API?
**A:** Common approaches:
- **JWT tokens**: Client sends `Authorization: Bearer <token>` header; server validates
- **OAuth2 / OpenID Connect**: Delegated authorization with access tokens
- **API Keys**: Simple key in header or query param
- **Basic Auth**: Base64(username:password) — only over HTTPS
Spring Security integrates with all of these.

---

### Q10: What is the difference between 401 and 403?
**A:** `401 Unauthorized` — the client is not authenticated (not logged in, missing/invalid token). `403 Forbidden` — the client IS authenticated but doesn't have permission to access the resource. Despite its name, 401 is about authentication; 403 is about authorization.

---

*End of Chapter 4 — Spring REST*
