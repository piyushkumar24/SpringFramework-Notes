# Chapter 6: Spring Security — Authentication, Authorization & JWT
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [Spring Security Architecture](#1-spring-security-architecture)
2. [Authentication vs Authorization](#2-authentication-vs-authorization)
3. [Security Filter Chain](#3-security-filter-chain)
4. [HttpSecurity Configuration](#4-httpsecurity-configuration)
5. [User Details & UserDetailsService](#5-user-details--userdetailsservice)
6. [Password Encoding](#6-password-encoding)
7. [JWT Authentication (Complete Implementation)](#7-jwt-authentication-complete-implementation)
8. [OAuth2 & OpenID Connect](#8-oauth2--openid-connect)
9. [Method-Level Security](#9-method-level-security)
10. [CORS & CSRF](#10-cors--csrf)
11. [Security Testing](#11-security-testing)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Spring Security Architecture

Spring Security is a powerful, customizable **authentication and authorization** framework.

### Core Components:
```
HTTP Request
    │
    ▼
┌──────────────────────────────────────────────┐
│              Security Filter Chain            │
│  ┌────────────────────────────────────────┐  │
│  │ UsernamePasswordAuthenticationFilter   │  │
│  │ BasicAuthenticationFilter              │  │
│  │ BearerTokenAuthenticationFilter (JWT)  │  │
│  │ ...30+ filters...                      │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
    │
    ▼
AuthenticationManager
    │
    ▼
AuthenticationProvider(s)
    │       └── DaoAuthenticationProvider
    │           ├── UserDetailsService
    │           └── PasswordEncoder
    ▼
SecurityContext (stores Authentication)
    │
    ▼
Controller / Service
```

---

## 2. Authentication vs Authorization

| | Authentication | Authorization |
|---|---|---|
| **Question** | Who are you? | What can you do? |
| **Process** | Verify identity (username+password, JWT) | Check permissions (roles, access rules) |
| **Result** | Authentication object in SecurityContext | Access granted or AccessDeniedException |
| **Spring class** | `AuthenticationManager` | `AccessDecisionManager` / `AuthorizationManager` |
| **Exception** | `AuthenticationException` → 401 | `AccessDeniedException` → 403 |

---

## 3. Security Filter Chain

Spring Security is implemented as a chain of servlet filters. Key filters:

| Filter | Purpose |
|---|---|
| `SecurityContextPersistenceFilter` | Loads/saves SecurityContext from session |
| `UsernamePasswordAuthenticationFilter` | Handles form login (POST /login) |
| `BasicAuthenticationFilter` | Handles HTTP Basic auth |
| `BearerTokenAuthenticationFilter` | Handles OAuth2/JWT Bearer tokens |
| `CsrfFilter` | CSRF protection |
| `CorsFilter` | CORS handling |
| `ExceptionTranslationFilter` | Translates security exceptions to HTTP responses |
| `AuthorizationFilter` | Final access control decision |

### SecurityFilterChain Bean (modern approach):
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }
}
```

---

## 4. HttpSecurity Configuration

### Stateless REST API Configuration:
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // Disable CSRF for stateless APIs
            .csrf(csrf -> csrf.disable())

            // Enable CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // Session management — stateless (no HttpSession)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/{id}/**").access(
                    new WebExpressionAuthorizationManager(
                        "#id == authentication.principal.id"))
                .anyRequest().authenticated()
            )

            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler())
            )

            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)

            .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://frontend.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // Strength 12 (FAANG level)
    }
}
```

---

## 5. User Details & UserDetailsService

### UserDetails:
```java
@Entity
@Table(name = "users")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User implements UserDetails {  // Implement UserDetails directly

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String email;
    private String password;

    @Enumerated(EnumType.STRING)
    private Role role;

    private boolean enabled;
    private boolean accountNonLocked;
    private boolean credentialsNonExpired;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return accountNonLocked; }

    @Override
    public boolean isCredentialsNonExpired() { return credentialsNonExpired; }

    @Override
    public boolean isEnabled() { return enabled; }
}
```

### UserDetailsService:
```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        return userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));
    }
}
```

---

## 6. Password Encoding

**NEVER store plain text passwords!**

```java
// BCrypt — adaptive, includes salt
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String encoded = encoder.encode("rawPassword");
boolean matches = encoder.matches("rawPassword", encoded);

// Argon2 (most modern, memory-hard)
PasswordEncoder argon2 = new Argon2PasswordEncoder(16, 32, 1, 65536, 10);

// DelegatingPasswordEncoder (supports multiple formats, migration)
PasswordEncoder delegating = PasswordEncoderFactories.createDelegatingPasswordEncoder();
// Stores: {bcrypt}$2a$10$...
// Allows migration between algorithms
```

### In Service:
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public User register(RegisterRequest request) {
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))  // Encode!
            .role(Role.USER)
            .enabled(true)
            .accountNonLocked(true)
            .credentialsNonExpired(true)
            .build();
        return userRepository.save(user);
    }
}
```

---

## 7. JWT Authentication (Complete Implementation)

### Dependencies:
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

### JwtService:
```java
@Service
public class JwtService {

    @Value("${app.security.jwt-secret}")
    private String secretKey;

    @Value("${app.security.jwt-expiration:86400000}")  // 24 hours default
    private long expirationMs;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims,
                                UserDetails userDetails) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs * 7))  // 7 days
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
}
```

### JwtAuthenticationFilter:
```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        // Skip if no Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7); // Remove "Bearer "

        try {
            final String username = jwtService.extractUsername(jwt);

            // Only authenticate if not already authenticated
            if (username != null &&
                    SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request));

                    // Set authentication in SecurityContext
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException ex) {
            // Invalid JWT — just continue without setting auth
            // ExceptionTranslationFilter will handle the 401
            log.warn("JWT validation failed: {}", ex.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

### AuthController:
```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@RequestBody @Valid RegisterRequest request) {
        return ResponseEntity.ok(authService.register(request));
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @RequestHeader("Refresh-Token") String refreshToken) {
        return ResponseEntity.ok(authService.refreshToken(refreshToken));
    }
}

@Service
@RequiredArgsConstructor
public class AuthService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authManager;

    public AuthResponse register(RegisterRequest req) {
        User user = User.builder()
            .username(req.getUsername())
            .email(req.getEmail())
            .password(passwordEncoder.encode(req.getPassword()))
            .role(Role.USER)
            .enabled(true).accountNonLocked(true).credentialsNonExpired(true)
            .build();
        userRepository.save(user);

        String token = jwtService.generateToken(user);
        String refresh = jwtService.generateRefreshToken(user);
        return new AuthResponse(token, refresh);
    }

    public AuthResponse login(LoginRequest req) {
        // This validates credentials and throws if invalid
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.getUsername(), req.getPassword()));

        User user = userRepository.findByUsername(req.getUsername()).orElseThrow();
        return new AuthResponse(
            jwtService.generateToken(user),
            jwtService.generateRefreshToken(user)
        );
    }
}
```

---

## 8. OAuth2 & OpenID Connect

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,email,profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                )
                .successHandler(oAuth2AuthenticationSuccessHandler)
            )
            .build();
    }
}
```

### Resource Server (OAuth2 JWT validation):
```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(jwtDecoder()))
            )
            .build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
            .build();
    }
}
```

---

## 9. Method-Level Security

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,   // @PreAuthorize, @PostAuthorize
    securedEnabled = true,   // @Secured
    jsr250Enabled = true     // @RolesAllowed
)
public class MethodSecurityConfig { }
```

### @PreAuthorize (most powerful — SpEL):
```java
@Service
public class UserService {

    // Role-based
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) { }

    // Multiple roles
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<User> getAllUsers() { }

    // Access method arguments
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public User getUserById(Long userId) { }

    // Complex expression
    @PreAuthorize("isAuthenticated() and #user.email == authentication.name")
    public void updateUser(User user) { }

    // Custom permission evaluator
    @PreAuthorize("hasPermission(#id, 'User', 'READ')")
    public User getUser(Long id) { }
}
```

### @PostAuthorize:
```java
// Check AFTER method executes (can inspect return value)
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { }
```

### @PostFilter / @PreFilter:
```java
// Filter collection after method returns
@PostFilter("filterObject.owner == authentication.name")
public List<Document> getDocuments() { }

// Filter collection before method is called
@PreFilter("filterObject.owner == authentication.name")
public void processDocuments(List<Document> docs) { }
```

### Custom Permission Evaluator:
```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject,
                                 Object permission) {
        if (auth == null || targetDomainObject == null || !(permission instanceof String)) {
            return false;
        }
        String type = targetDomainObject.getClass().getSimpleName();
        return hasPrivilege(auth, type, permission.toString().toUpperCase());
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId,
                                 String targetType, Object permission) {
        if (auth == null || targetType == null || !(permission instanceof String)) {
            return false;
        }
        return hasPrivilege(auth, targetType.toUpperCase(), permission.toString().toUpperCase());
    }

    private boolean hasPrivilege(Authentication auth, String targetType, String permission) {
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals(targetType + "_" + permission));
    }
}
```

---

## 10. CORS & CSRF

### CSRF:
CSRF attacks trick authenticated users into making unintended requests. Spring Security enables CSRF protection by default using **Synchronizer Token Pattern**.

```java
// For REST APIs with JWT (stateless) — disable CSRF
http.csrf(csrf -> csrf.disable())

// For traditional web apps — keep enabled
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .ignoringRequestMatchers("/api/webhooks/**")
)
```

### CORS:
```java
http.cors(cors -> cors.configurationSource(request -> {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("https://*.example.com"));
    config.setAllowedMethods(List.of("*"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    return config;
}))
```

---

## 11. Security Testing

```java
@WebMvcTest(UserController.class)
class UserControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturn401WhenNotAuthenticated() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "john", roles = "USER")
    void shouldReturn200WhenAuthenticated() throws Exception {
        mockMvc.perform(get("/api/users/me"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "john", roles = "USER")
    void shouldReturn403WhenNotAdmin() throws Exception {
        mockMvc.perform(delete("/api/admin/users/1"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void shouldReturn204WhenAdminDeletes() throws Exception {
        mockMvc.perform(delete("/api/admin/users/1"))
            .andExpect(status().isNoContent());
    }

    // Test with custom JWT
    @Test
    void shouldReturn200WithValidJwt() throws Exception {
        String token = jwtService.generateToken(testUser);
        mockMvc.perform(get("/api/users")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
}
```

---

## 12. Interview Questions & Answers

---

### Q1: How does Spring Security authenticate a user?
**A:** 
1. Request passes through `UsernamePasswordAuthenticationFilter` (form) or `BearerTokenAuthenticationFilter` (JWT)
2. Filter creates an `Authentication` object with credentials
3. Passes to `AuthenticationManager` (default: `ProviderManager`)
4. `ProviderManager` delegates to `DaoAuthenticationProvider`
5. `DaoAuthenticationProvider` calls `UserDetailsService.loadUserByUsername()`
6. Compares passwords using `PasswordEncoder.matches()`
7. If successful, returns fully populated `Authentication`
8. Stored in `SecurityContextHolder`

---

### Q2: What is SecurityContextHolder?
**A:** Thread-local storage for the `SecurityContext`, which holds the current `Authentication` object. By default uses `ThreadLocal` strategy — each request thread has its own SecurityContext. It's cleared after the request completes. Strategies: `THREAD_LOCAL`, `INHERITED_THREAD_LOCAL`, `GLOBAL`.

---

### Q3: What is the difference between authentication and authorization in Spring Security?
**A:** Authentication = verifying identity (who you are). Authorization = determining permissions (what you can do). Authentication happens first (filter chain), populates `SecurityContext`. Authorization checks roles/permissions on secured resources (`@PreAuthorize`, URL rules). Different exceptions: `AuthenticationException` (401) vs `AccessDeniedException` (403).

---

### Q4: How does JWT work for authentication?
**A:** Client sends credentials → server validates, creates JWT (Header.Payload.Signature) signed with secret → returns JWT to client → client stores JWT (localStorage/cookie) → sends `Authorization: Bearer <token>` header on every request → server validates token signature and expiry, extracts username, loads UserDetails, sets SecurityContext → no session needed (stateless).

---

### Q5: What is BCrypt and why is it used?
**A:** BCrypt is an adaptive password hashing algorithm. It: incorporates a random salt (no rainbow table attacks), is intentionally slow (configurable cost factor), adapts to hardware improvements by increasing cost. BCrypt's strength parameter lets you increase work factor as hardware gets faster. Never use MD5/SHA1 for passwords.

---

### Q6: What is CSRF? When should you disable it?
**A:** CSRF (Cross-Site Request Forgery) tricks authenticated users into submitting malicious requests. Spring Security prevents it via the Synchronizer Token Pattern. **Disable for stateless REST APIs** using JWT/OAuth2 — no cookies/sessions means no CSRF risk. **Keep enabled for browser-based form submissions** using sessions/cookies.

---

### Q7: What is @PreAuthorize and how does it differ from URL-based security?
**A:** URL-based security (in `HttpSecurity`) is coarse-grained — applies to URL patterns. `@PreAuthorize` is fine-grained — applies to individual methods with SpEL expressions, can access method arguments and return values, works anywhere in the service layer (not just controllers). `@PreAuthorize` is more expressive and closer to business logic.

---

### Q8: What is the Spring Security filter chain?
**A:** It's an ordered chain of servlet filters (`SecurityFilterChain`) that intercepts HTTP requests. Each filter performs a specific security task (CSRF, authentication, authorization). The chain is managed by `FilterChainProxy` which delegates to the appropriate `SecurityFilterChain` based on request matchers. You can have multiple chains for different URL patterns.

---

### Q9: How do you implement role hierarchy?
**A:**
```java
@Bean
public RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy("ROLE_ADMIN > ROLE_MANAGER > ROLE_USER");
    return hierarchy;
}
// Now ADMIN automatically has USER and MANAGER authorities
```

---

### Q10: What is the difference between @Secured and @PreAuthorize?
**A:** `@Secured` is simpler — only supports role names (Strings), no SpEL. `@PreAuthorize` supports full SpEL expressions, method argument access, custom permission evaluators. `@PreAuthorize` is generally preferred for its flexibility. Both require `@EnableMethodSecurity`.

---

*End of Chapter 6 — Spring Security*
