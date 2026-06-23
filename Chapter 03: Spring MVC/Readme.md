# Chapter 3: Spring MVC — Web Layer Architecture
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [What is Spring MVC?](#1-what-is-spring-mvc)
2. [DispatcherServlet & Request Flow](#2-dispatcherservlet--request-flow)
3. [Controller Layer](#3-controller-layer)
4. [Request Mapping](#4-request-mapping)
5. [Handler Method Arguments](#5-handler-method-arguments)
6. [Model, View & ViewResolver](#6-model-view--viewresolver)
7. [Data Binding & Validation](#7-data-binding--validation)
8. [Exception Handling](#8-exception-handling)
9. [Interceptors](#9-interceptors)
10. [Filters vs Interceptors](#10-filters-vs-interceptors)
11. [Form Handling & Thymeleaf](#11-form-handling--thymeleaf)
12. [Multipart File Upload](#12-multipart-file-upload)
13. [Spring MVC Configuration](#13-spring-mvc-configuration)
14. [Interview Questions & Answers](#14-interview-questions--answers)

---

## 1. What is Spring MVC?

Spring MVC is Spring's **web framework** implementing the **Model-View-Controller** pattern. It provides a clean separation of web layer concerns:

- **Model**: Application data (Java objects, domain model)
- **View**: Rendering layer (HTML, JSON, XML, PDF, etc.)
- **Controller**: Handles requests, processes input, returns response

### Architecture Overview:
```
Browser
  │
  ▼
DispatcherServlet (Front Controller)
  │
  ├─── HandlerMapping → finds Controller
  │
  ├─── HandlerAdapter → invokes Controller method
  │
  ├─── Controller → business logic → returns ModelAndView
  │
  ├─── ViewResolver → resolves view name → View
  │
  └─── View → renders response
  │
  ▼
Browser (HTML / JSON response)
```

---

## 2. DispatcherServlet & Request Flow

`DispatcherServlet` is the **Front Controller** — the single entry point for all HTTP requests.

### Complete Request Lifecycle:
```
1. Client sends HTTP request
        ↓
2. DispatcherServlet receives it
        ↓
3. HandlerMapping consulted → returns HandlerExecutionChain
   (Handler + List<HandlerInterceptor>)
        ↓
4. HandlerInterceptor.preHandle() called
        ↓
5. HandlerAdapter found for handler type
        ↓
6. HandlerAdapter invokes controller method
   (argument resolving, return value handling)
        ↓
7. Controller returns ModelAndView (or @ResponseBody)
        ↓
8. HandlerInterceptor.postHandle() called
        ↓
9. ViewResolver resolves view name → View object
        ↓
10. View renders response (HTML, JSON, etc.)
        ↓
11. HandlerInterceptor.afterCompletion() called
        ↓
12. Response sent to client
```

### DispatcherServlet Registration:
```java
// Modern: Spring Boot auto-registers it
// Manual: via AbstractAnnotationConfigDispatcherServletInitializer

public class WebAppInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };  // Service, Repo beans
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };   // MVC beans
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };  // Map all URLs
    }
}
```

---

## 3. Controller Layer

### @Controller (template rendering):
```java
@Controller
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public String listUsers(Model model) {
        model.addAttribute("users", userService.findAll());
        return "users/list";  // View name resolved by ViewResolver
    }

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.findById(id));
        return "users/detail";
    }
}
```

### @RestController (API responses):
```java
@RestController  // = @Controller + @ResponseBody on all methods
@RequestMapping("/api/users")
public class UserRestController {

    @GetMapping
    public List<UserDTO> listUsers() {
        return userService.findAll();  // Automatically serialized to JSON
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## 4. Request Mapping

### HTTP Method Annotations:
```java
@GetMapping("/users")           // GET
@PostMapping("/users")          // POST
@PutMapping("/users/{id}")      // PUT
@PatchMapping("/users/{id}")    // PATCH
@DeleteMapping("/users/{id}")   // DELETE

// Equivalent verbose form:
@RequestMapping(value = "/users", method = RequestMethod.GET)
```

### Path Variables:
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { ... }

// Multiple path variables
@GetMapping("/users/{userId}/orders/{orderId}")
public Order getOrder(@PathVariable Long userId,
                      @PathVariable Long orderId) { ... }

// Optional path variable
@GetMapping({"/users", "/users/{id}"})
public User getUser(@PathVariable(required = false) Long id) { ... }
```

### Request Parameters:
```java
@GetMapping("/users")
public List<User> search(
    @RequestParam String name,                    // required by default
    @RequestParam(required = false) String email, // optional
    @RequestParam(defaultValue = "0") int page,  // with default
    @RequestParam(defaultValue = "10") int size
) { ... }
```

### Headers & Content Negotiation:
```java
@GetMapping(value = "/users",
            produces = MediaType.APPLICATION_JSON_VALUE)
public List<User> getJsonUsers() { ... }

@PostMapping(value = "/users",
             consumes = MediaType.APPLICATION_JSON_VALUE,
             produces = MediaType.APPLICATION_JSON_VALUE)
public User createUser(@RequestBody User user) { ... }

// Header-based routing
@GetMapping(value = "/users", headers = "X-API-VERSION=2")
public List<UserV2> getUsersV2() { ... }
```

---

## 5. Handler Method Arguments

Spring MVC auto-resolves these parameter types:

| Parameter Type | Description |
|---|---|
| `@PathVariable` | URL template variable |
| `@RequestParam` | Query or form parameter |
| `@RequestBody` | HTTP body (JSON→Object via Jackson) |
| `@RequestHeader` | HTTP header value |
| `@CookieValue` | HTTP cookie value |
| `@ModelAttribute` | Model attribute / form data |
| `Model` / `ModelMap` | Add attributes to model |
| `HttpServletRequest/Response` | Raw servlet objects |
| `HttpSession` | Current session |
| `Principal` | Authenticated principal |
| `Locale` | Current locale |
| `InputStream/OutputStream` | Request/response streams |
| `@SessionAttribute` | Session attribute |
| `@RequestAttribute` | Request attribute |
| `Errors` / `BindingResult` | Validation results |
| `UriComponentsBuilder` | Build URIs |

### Examples:
```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
    @RequestBody @Valid OrderRequest request,   // JSON body + validation
    @RequestHeader("Authorization") String auth, // header
    @CookieValue("sessionId") String sessionId,  // cookie
    @RequestAttribute("requestId") String reqId, // request attr
    HttpServletRequest servletRequest,           // raw request
    Principal principal                          // authenticated user
) {
    String userId = principal.getName();
    Order order = orderService.create(request, userId);
    URI location = URI.create("/api/orders/" + order.getId());
    return ResponseEntity.created(location).body(order);
}
```

---

## 6. Model, View & ViewResolver

### ModelAndView:
```java
@GetMapping("/dashboard")
public ModelAndView dashboard() {
    ModelAndView mav = new ModelAndView("dashboard");  // View name
    mav.addObject("stats", analyticsService.getStats());
    mav.addObject("user", getCurrentUser());
    return mav;
}
```

### ViewResolver Configuration:
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public InternalResourceViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }

    // Thymeleaf (Spring Boot auto-configures this)
    @Bean
    public SpringTemplateEngine templateEngine(ITemplateResolver templateResolver) {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setTemplateResolver(templateResolver);
        engine.setEnableSpringELCompiler(true);
        return engine;
    }
}
```

### ContentNegotiatingViewResolver:
Picks the best view based on `Accept` header:
```java
// Accept: text/html → JSP/Thymeleaf view
// Accept: application/json → Jackson JSON
// Accept: application/xml → JAXB XML
```

---

## 7. Data Binding & Validation

### @Valid + Bean Validation (JSR-380):
```java
public class UserDTO {
    @NotNull(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be 2-50 chars")
    private String name;

    @NotNull
    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Must be 18+")
    @Max(value = 120)
    private int age;

    @NotNull
    @Pattern(regexp = "^[0-9]{10}$", message = "Phone must be 10 digits")
    private String phone;
}
```

```java
@PostMapping("/users")
public ResponseEntity<?> createUser(
    @RequestBody @Valid UserDTO userDTO,
    BindingResult bindingResult  // Must immediately follow @Valid param
) {
    if (bindingResult.hasErrors()) {
        Map<String, String> errors = new HashMap<>();
        bindingResult.getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage())
        );
        return ResponseEntity.badRequest().body(errors);
    }
    User user = userService.create(userDTO);
    return ResponseEntity.ok(user);
}
```

### Custom Validator:
```java
@Component
public class UniqueEmailValidator
        implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        if (email == null) return true;
        return !userRepository.existsByEmail(email);
    }
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### @ModelAttribute:
```java
// Adds to model before every request in this controller
@ModelAttribute("categories")
public List<Category> loadCategories() {
    return categoryService.findAll();
}

// Form binding
@PostMapping("/users/new")
public String saveUser(@ModelAttribute UserForm form, BindingResult result) {
    if (result.hasErrors()) return "users/new";
    userService.save(form);
    return "redirect:/users";
}
```

---

## 8. Exception Handling

### @ExceptionHandler (Controller level):
```java
@Controller
public class UserController {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("USER_NOT_FOUND", ex.getMessage()));
    }
}
```

### @ControllerAdvice (Global):
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        return new ErrorResponse("VALIDATION_FAILED", errors.toString());
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex) {
        return new ErrorResponse("ACCESS_DENIED", "Insufficient permissions");
    }

    @ExceptionHandler(Exception.class)  // Catch-all
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

### ResponseEntityExceptionHandler (Spring MVC base class):
```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // Override specific Spring MVC exceptions
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {

        Map<String, Object> body = new LinkedHashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", status.value());
        body.put("errors", ex.getBindingResult().getFieldErrors()
            .stream().map(FieldError::getDefaultMessage).toList());

        return new ResponseEntity<>(body, headers, status);
    }
}
```

---

## 9. Interceptors

Interceptors apply to Spring MVC handler execution (after `DispatcherServlet`, before controller).

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || !jwtService.isValid(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;  // Stop processing — controller NOT called
        }
        return true;  // Continue processing
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) {
        // After controller, before view rendering
        // Can add model attributes here
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        // After view rendered (or exception occurred)
        // Used for resource cleanup
    }
}
```

### Registering Interceptors:
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private AuthenticationInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**", "/api/auth/**");
    }
}
```

---

## 10. Filters vs Interceptors

| Aspect | Filter | Interceptor |
|---|---|---|
| Layer | Servlet container | Spring MVC |
| Interface | `javax.servlet.Filter` | `HandlerInterceptor` |
| Access to Spring context | ❌ (usually) | ✅ |
| Handler access | ❌ | ✅ (`handler` parameter) |
| Apply to | All requests | Only Spring MVC mapped requests |
| Use case | Auth tokens, encoding, CORS, compression | Logging, auth (MVC-level), model augmentation |

```java
// Servlet Filter
@Component
@Order(1)
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        long start = System.currentTimeMillis();

        chain.doFilter(request, response); // Continue chain

        long elapsed = System.currentTimeMillis() - start;
        log.info("{} {} completed in {}ms",
            req.getMethod(), req.getRequestURI(), elapsed);
    }
}
```

---

## 11. Form Handling & Thymeleaf

```html
<!-- Thymeleaf form -->
<form th:action="@{/users}" th:object="${userForm}" method="post">
    <input type="text" th:field="*{name}"
           th:errorclass="field-error"/>
    <span th:if="${#fields.hasErrors('name')}"
          th:errors="*{name}" class="error-msg"></span>

    <input type="email" th:field="*{email}"/>
    <button type="submit">Save</button>
</form>
```

```java
@GetMapping("/users/new")
public String newUserForm(Model model) {
    model.addAttribute("userForm", new UserForm());
    return "users/new";
}

@PostMapping("/users")
public String saveUser(@ModelAttribute @Valid UserForm form,
                       BindingResult result, Model model) {
    if (result.hasErrors()) {
        return "users/new";  // Return to form with errors
    }
    userService.save(form);
    return "redirect:/users";  // POST-Redirect-GET pattern
}
```

---

## 12. Multipart File Upload

```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
    @RequestParam("file") MultipartFile file,
    @RequestParam("description") String description
) throws IOException {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("File is empty");
    }

    String filename = StringUtils.cleanPath(file.getOriginalFilename());
    Path uploadDir = Paths.get("uploads");
    Files.createDirectories(uploadDir);
    Files.copy(file.getInputStream(), uploadDir.resolve(filename),
               StandardCopyOption.REPLACE_EXISTING);

    return ResponseEntity.ok("Uploaded: " + filename);
}
```

```properties
# application.properties
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

---

## 13. Spring MVC Configuration

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Static resources
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCachePeriod(3600);
    }

    // CORS
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }

    // Message converters (Jackson)
    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json()
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .modules(new JavaTimeModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
    }

    // Default servlet
    @Override
    public void configureDefaultServletHandling(
            DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

---

## 14. Interview Questions & Answers

---

### Q1: What is DispatcherServlet and its role in Spring MVC?
**A:** `DispatcherServlet` is the **Front Controller** that intercepts all HTTP requests. It coordinates:
1. `HandlerMapping` to find which controller handles the request
2. `HandlerAdapter` to invoke the controller
3. `ViewResolver` to render the response
It's registered in `web.xml` or via `WebApplicationInitializer`.

---

### Q2: What is the difference between @Controller and @RestController?
**A:** `@Controller` is for MVC — methods return view names (String) or `ModelAndView`. `@RestController` = `@Controller` + `@ResponseBody` — every method's return value is serialized directly to the HTTP response body (JSON/XML). No view resolution.

---

### Q3: What is @ResponseBody and how does it work?
**A:** `@ResponseBody` tells Spring to serialize the return value using `HttpMessageConverter` (Jackson for JSON, JAXB for XML) and write it directly to the response body, bypassing view resolution.

---

### Q4: How does Spring MVC handle JSON serialization/deserialization?
**A:** Via `HttpMessageConverter`, specifically `MappingJackson2HttpMessageConverter`. When `@RequestBody` is on a parameter, Jackson deserializes the JSON request body to the Java object. When `@ResponseBody` is present, Jackson serializes the Java object to JSON response body. Spring checks the `Content-Type` and `Accept` headers.

---

### Q5: What is the difference between @PathVariable and @RequestParam?
**A:** `@PathVariable` extracts values from the URL path: `/users/{id}`. `@RequestParam` extracts values from query string or form data: `/users?page=1&size=10`. Both can be required or optional.

---

### Q6: What is @ControllerAdvice? How does it differ from @ExceptionHandler in a controller?
**A:** `@ControllerAdvice` is a global controller enhancement class. `@ExceptionHandler` inside a controller handles only that controller's exceptions. `@ControllerAdvice` applies to all controllers globally. It can also contain `@ModelAttribute` and `@InitBinder` methods that apply globally.

---

### Q7: Explain the HandlerMapping and HandlerAdapter.
**A:** `HandlerMapping` maps incoming requests to handler objects (controller methods). Default is `RequestMappingHandlerMapping` which processes `@RequestMapping`. `HandlerAdapter` invokes the handler in a uniform way — `RequestMappingHandlerAdapter` handles `@Controller` methods, resolving arguments, calling the method, and handling return values.

---

### Q8: What is content negotiation in Spring MVC?
**A:** Content negotiation determines the response format based on the client's `Accept` header, URL extension (`.json`, `.xml`), or request parameter. `ContentNegotiatingViewResolver` picks the best view. `@RequestMapping(produces=...)` restricts methods to specific output formats.

---

### Q9: What is the difference between Filter and Interceptor?
**A:** Filters are Servlet-level — run before `DispatcherServlet`, applied to all requests, no Spring context access. Interceptors are Spring MVC level — run after `DispatcherServlet`, have access to handler info and Spring context, only apply to requests mapped by Spring MVC.

---

### Q10: What is the POST-Redirect-GET pattern and why is it important?
**A:** After a POST form submission, redirect to a GET request (`return "redirect:/users"`). This prevents duplicate form submissions on browser refresh. Without it, refreshing the page re-submits the POST request.

---

### Q11: What is @InitBinder?
**A:** `@InitBinder` customizes the `WebDataBinder` for a specific controller or globally (in `@ControllerAdvice`). Used to register custom editors/converters:
```java
@InitBinder
public void initBinder(WebDataBinder binder) {
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
    binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
}
```

---

### Q12: How does Spring MVC support async request processing?
**A:** Spring MVC supports `Callable<T>` (runs in separate thread from thread pool), `DeferredResult<T>` (result set from any thread, like reactive), `CompletableFuture<T>`, and `ResponseBodyEmitter`/`SseEmitter` for streaming. This frees the request-handling thread while processing.

---

*End of Chapter 3 — Spring MVC*
