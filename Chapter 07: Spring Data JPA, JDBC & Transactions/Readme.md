# Chapter 7: Spring Data JPA, JDBC & Transactions
> 📘 FAANG Interview Ready | Textbook Level Coverage

---

## Table of Contents
1. [Spring Data Overview](#1-spring-data-overview)
2. [Spring JDBC & JdbcTemplate](#2-spring-jdbc--jdbctemplate)
3. [Spring Data JPA — Repositories](#3-spring-data-jpa--repositories)
4. [Entity Relationships & Mapping](#4-entity-relationships--mapping)
5. [JPQL & Named Queries](#5-jpql--named-queries)
6. [Spring Data Specifications (Dynamic Queries)](#6-spring-data-specifications-dynamic-queries)
7. [Projections & DTOs](#7-projections--dtos)
8. [Auditing](#8-auditing)
9. [Spring Transactions](#9-spring-transactions)
10. [Connection Pooling — HikariCP](#10-connection-pooling--hikaricp)
11. [Caching](#11-caching)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Spring Data Overview

Spring Data provides a unified data access framework:

```
Spring Data
├── Spring Data JPA           → JPA (Hibernate)
├── Spring Data MongoDB       → MongoDB
├── Spring Data Redis         → Redis
├── Spring Data Elasticsearch → Elasticsearch
├── Spring Data R2DBC         → Reactive relational DBs
└── Spring Data JDBC          → Simple JDBC (no JPA)
```

### Layered Architecture:
```
Controller → Service → Repository (Spring Data) → Database
```

---

## 2. Spring JDBC & JdbcTemplate

`JdbcTemplate` eliminates boilerplate JDBC code (opening/closing connections, handling exceptions).

### Traditional JDBC vs JdbcTemplate:
```java
// Traditional JDBC — boilerplate nightmare
Connection conn = null;
PreparedStatement ps = null;
try {
    conn = dataSource.getConnection();
    ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    ps.setLong(1, id);
    ResultSet rs = ps.executeQuery();
    if (rs.next()) {
        return mapResultSet(rs);
    }
} catch (SQLException e) {
    throw new RuntimeException(e);
} finally {
    // close ps, conn...
}

// JdbcTemplate — clean!
User user = jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE id = ?",
    userRowMapper,
    id
);
```

### JdbcTemplate Complete Usage:
```java
@Repository
@RequiredArgsConstructor
public class UserJdbcRepository {

    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    private final RowMapper<User> userRowMapper = (rs, rowNum) -> {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
        user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
        return user;
    };

    // Query single row
    public Optional<User> findById(Long id) {
        try {
            User user = jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userRowMapper, id);
            return Optional.ofNullable(user);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    // Query multiple rows
    public List<User> findAll() {
        return jdbcTemplate.query("SELECT * FROM users", userRowMapper);
    }

    // Query scalar value
    public long count() {
        return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM users", Long.class);
    }

    public String getNameById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT name FROM users WHERE id = ?", String.class, id);
    }

    // Insert with generated key
    public Long insert(User user) {
        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(
                "INSERT INTO users (name, email) VALUES (?, ?)",
                Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, user.getName());
            ps.setString(2, user.getEmail());
            return ps;
        }, keyHolder);
        return keyHolder.getKey().longValue();
    }

    // Update
    public int update(User user) {
        return jdbcTemplate.update(
            "UPDATE users SET name=?, email=? WHERE id=?",
            user.getName(), user.getEmail(), user.getId());
    }

    // Delete
    public int delete(Long id) {
        return jdbcTemplate.update("DELETE FROM users WHERE id = ?", id);
    }

    // Batch insert
    public void batchInsert(List<User> users) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            users, users.size(),
            (ps, user) -> {
                ps.setString(1, user.getName());
                ps.setString(2, user.getEmail());
            });
    }

    // Named parameters (more readable)
    public List<User> findByNameAndEmail(String name, String email) {
        String sql = "SELECT * FROM users WHERE name = :name AND email = :email";
        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("name", name)
            .addValue("email", email);
        return namedJdbcTemplate.query(sql, params, userRowMapper);
    }

    // Transaction using DataSourceTransactionManager
    public void transferData(Long fromId, Long toId) {
        // Use @Transactional instead — covered in Transaction section
    }
}
```

### SimpleJdbcInsert:
```java
SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
    .withTableName("users")
    .usingGeneratedKeyColumns("id");

Map<String, Object> params = new HashMap<>();
params.put("name", user.getName());
params.put("email", user.getEmail());

Number id = insert.executeAndReturnKey(params);
```

---

## 3. Spring Data JPA — Repositories

### Repository Hierarchy:
```
Repository<T, ID>
    └── CrudRepository<T, ID>          (save, findById, findAll, delete, count)
            └── PagingAndSortingRepository  (findAll with Pageable/Sort)
                    └── JpaRepository<T, ID>  (flush, saveAndFlush, deleteInBatch)
```

### Basic Repository:
```java
@Entity
@Table(name = "users",
       indexes = {
           @Index(name = "idx_user_email", columnList = "email", unique = true),
           @Index(name = "idx_user_name", columnList = "name")
       })
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}
```

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query from method name
    Optional<User> findByEmail(String email);
    List<User> findByNameContainingIgnoreCase(String name);
    List<User> findByRoleAndEnabled(Role role, boolean enabled);
    boolean existsByEmail(String email);
    long countByRole(Role role);
    void deleteByEmail(String email);

    // Sorted
    List<User> findByRoleOrderByNameAsc(Role role);

    // Paginated
    Page<User> findByRole(Role role, Pageable pageable);

    // Custom JPQL
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.enabled = true")
    Optional<User> findActiveByEmail(@Param("email") String email);

    // Custom native SQL
    @Query(value = "SELECT * FROM users WHERE YEAR(created_at) = :year",
           nativeQuery = true)
    List<User> findByCreatedYear(@Param("year") int year);

    // Modifying query
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.enabled = false WHERE u.lastLogin < :cutoff")
    int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);

    // Fetch with JOIN to avoid N+1
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
}
```

---

## 4. Entity Relationships & Mapping

### @OneToMany / @ManyToOne:
```java
@Entity
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // Many orders → One user (owning side)
    @ManyToOne(fetch = FetchType.LAZY)  // LAZY is best practice
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "order",
               cascade = CascadeType.ALL,
               orphanRemoval = true,
               fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();

    // Helper methods for bidirectional consistency
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}
```

### @ManyToMany:
```java
@Entity
public class Student {

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### @OneToOne:
```java
@Entity
public class User {
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id", referencedColumnName = "id")
    private UserProfile profile;
}
```

### FetchType — Critical Concept:
| | EAGER | LAZY |
|---|---|---|
| When loaded | Immediately with parent | Only when accessed |
| Default for | `@ManyToOne`, `@OneToOne` | `@OneToMany`, `@ManyToMany` |
| Performance | Can cause over-fetching | Risk of LazyInitializationException |
| Recommendation | Use sparingly | Always prefer (default for collections) |

### N+1 Problem:
```java
// N+1 Problem: 1 query for users + N queries for each user's orders
List<User> users = userRepository.findAll(); // 1 query
for (User u : users) {
    System.out.println(u.getOrders().size()); // N queries!
}

// Fix 1: JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();

// Fix 2: @EntityGraph
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<User> findAll();

// Fix 3: Batch fetching
@BatchSize(size = 20)
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

---

## 5. JPQL & Named Queries

### JPQL (Java Persistence Query Language):
```java
// Entity names, not table names!
// Field names, not column names!

// Basic select
@Query("SELECT u FROM User u")
@Query("SELECT u FROM User u WHERE u.email = ?1")
@Query("SELECT u FROM User u WHERE u.email = :email")

// Aggregate functions
@Query("SELECT COUNT(u) FROM User u WHERE u.role = :role")
@Query("SELECT AVG(o.total) FROM Order o WHERE o.user.id = :userId")

// Join
@Query("SELECT u FROM User u JOIN u.orders o WHERE o.status = :status")

// Subquery
@Query("SELECT u FROM User u WHERE u.id IN " +
       "(SELECT o.user.id FROM Order o WHERE o.total > :minTotal)")

// Constructor expression (DTO projection)
@Query("SELECT new com.example.dto.UserSummary(u.id, u.name, COUNT(o)) " +
       "FROM User u LEFT JOIN u.orders o GROUP BY u.id, u.name")
List<UserSummary> getUserSummaries();
```

### Named Queries:
```java
@Entity
@NamedQueries({
    @NamedQuery(name = "User.findByRole",
                query = "SELECT u FROM User u WHERE u.role = :role"),
    @NamedQuery(name = "User.findActiveUsers",
                query = "SELECT u FROM User u WHERE u.enabled = true ORDER BY u.name")
})
public class User { }

// Usage in repository (Spring Data finds them by entity name + method name)
List<User> findByRole(@Param("role") Role role);  // Uses User.findByRole
```

---

## 6. Spring Data Specifications (Dynamic Queries)

Specifications implement the **Specification pattern** using JPA Criteria API:

```java
// Entity must implement Specification<T>
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> { }

// Specification building blocks
public class UserSpecifications {

    public static Specification<User> hasName(String name) {
        return (root, query, cb) -> {
            if (name == null) return null;  // or cb.conjunction()
            return cb.like(cb.lower(root.get("name")),
                           "%" + name.toLowerCase() + "%");
        };
    }

    public static Specification<User> hasRole(Role role) {
        return (root, query, cb) ->
            role == null ? null : cb.equal(root.get("role"), role);
    }

    public static Specification<User> isEnabled() {
        return (root, query, cb) -> cb.isTrue(root.get("enabled"));
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }
}

// Usage — compose specifications
public List<User> searchUsers(UserSearchCriteria criteria) {
    Specification<User> spec = Specification.where(null);

    if (criteria.getName() != null) {
        spec = spec.and(UserSpecifications.hasName(criteria.getName()));
    }
    if (criteria.getRole() != null) {
        spec = spec.and(UserSpecifications.hasRole(criteria.getRole()));
    }
    if (criteria.isOnlyEnabled()) {
        spec = spec.and(UserSpecifications.isEnabled());
    }

    return userRepository.findAll(spec);
}
```

---

## 7. Projections & DTOs

### Interface Projection (lightest):
```java
// Spring Data creates a proxy implementing this interface
public interface UserSummaryProjection {
    Long getId();
    String getName();
    String getEmail();

    // SpEL in projection
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}

Page<UserSummaryProjection> findBy(Pageable pageable);
```

### Class-based DTO Projection:
```java
public record UserDTO(Long id, String name, String email) {}

// Constructor expression
@Query("SELECT new com.example.dto.UserDTO(u.id, u.name, u.email) FROM User u")
List<UserDTO> findAllAsDTO();

// Or using interface (Spring Data infers from return type)
List<UserDTO> findByRole(Role role);  // Maps to UserDTO constructor
```

---

## 8. Auditing

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorAware")
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(
            SecurityContextHolder.getContext().getAuthentication()
        ).map(Authentication::getName);
    }
}

@Entity
@EntityListeners(AuditingEntityListener.class)
public class User {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

---

## 9. Spring Transactions

### @Transactional — Core Concepts:

```java
@Service
public class OrderService {

    @Transactional  // Default: REQUIRED, RUNTIME exception rollback
    public Order placeOrder(OrderRequest request) {
        Order order = createOrder(request);
        paymentService.processPayment(order);  // If this throws → rollback
        inventoryService.deductStock(order);
        notificationService.sendConfirmation(order);  // If this throws → rollback
        return order;
    }
}
```

### Transaction Attributes:

```java
@Transactional(
    propagation = Propagation.REQUIRED,       // Join existing or create new
    isolation = Isolation.READ_COMMITTED,     // Isolation level
    readOnly = false,                         // Optimize reads when true
    timeout = 30,                             // 30 seconds timeout
    rollbackFor = {BusinessException.class},  // Also rollback on checked
    noRollbackFor = {LoggingException.class}  // Don't rollback on these
)
public void complexOperation() { }
```

### Transaction Propagation:

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction; create if none |
| `REQUIRES_NEW` | Always create new transaction; suspend existing |
| `NESTED` | Create nested transaction (savepoint) within existing |
| `SUPPORTS` | Join existing if present; run non-transactional if none |
| `NOT_SUPPORTED` | Suspend existing; run non-transactional |
| `MANDATORY` | Must join existing; throw if none |
| `NEVER` | Must NOT be in transaction; throw if one exists |

```java
@Service
public class AuditService {
    // Even if parent rolls back, audit log is saved
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(String action) {
        auditRepository.save(new AuditLog(action, LocalDateTime.now()));
    }
}
```

### Isolation Levels:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ_UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible |
| READ_COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible |
| REPEATABLE_READ | ❌ Prevented | ❌ Prevented | ✅ Possible |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented |

```java
// Service-level override
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalBankTransfer() { }
```

### Rollback Rules:

By default, Spring rolls back on **unchecked exceptions** (RuntimeException) only!

```java
// Checked exceptions — NO rollback by default!
@Transactional
public void riskyOperation() throws IOException {
    repository.save(entity);
    throw new IOException("Network error");  // Transaction COMMITS! Bug!
}

// Fix: explicit rollback for checked exceptions
@Transactional(rollbackFor = IOException.class)
public void riskyOperation() throws IOException { ... }

// Or use unchecked exception wrapper
@Transactional
public void riskyOperation() {
    try {
        repository.save(entity);
        networkService.call();
    } catch (IOException e) {
        throw new RuntimeException("Network error", e);  // Now rolls back
    }
}
```

### Programmatic Transaction Management:
```java
@Service
@RequiredArgsConstructor
public class ManualTransactionService {

    private final TransactionTemplate transactionTemplate;
    private final PlatformTransactionManager transactionManager;

    // Using TransactionTemplate (preferred)
    public User createUserManually(CreateUserRequest request) {
        return transactionTemplate.execute(status -> {
            try {
                User user = userRepository.save(new User(request));
                profileRepository.save(new Profile(user));
                return user;
            } catch (Exception e) {
                status.setRollbackOnly();  // Mark for rollback
                throw e;
            }
        });
    }

    // Using PlatformTransactionManager (verbose)
    public void directTransaction() {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            // ... database operations ...
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

---

## 10. Connection Pooling — HikariCP

HikariCP is the **fastest** Java connection pool (Spring Boot default):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
    driver-class-name: org.postgresql.Driver
    hikari:
      # Pool size
      maximum-pool-size: 20        # Max connections in pool
      minimum-idle: 5              # Min idle connections
      idle-timeout: 600000         # 10 min idle before closing
      max-lifetime: 1800000        # 30 min max connection lifetime
      connection-timeout: 30000    # 30 sec wait to get connection

      # Connection validation
      connection-test-query: SELECT 1  # Validation query
      keepalive-time: 60000           # Ping every 60 sec

      # Pool naming
      pool-name: MyHikariPool

      # Leak detection
      leak-detection-threshold: 2000  # Warn if held > 2 sec
```

---

## 11. Caching

### Enable Caching:
```java
@SpringBootApplication
@EnableCaching
public class Application { }
```

### Cache Annotations:
```java
@Service
public class UserService {

    // Cache result — key is method argument by default
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    // Composite cache key
    @Cacheable(value = "users",
               key = "#name + ':' + #role",
               condition = "#name != null")
    public List<User> findByNameAndRole(String name, Role role) { ... }

    // Update cache on save
    @CachePut(value = "users", key = "#result.id")
    public User save(User user) {
        return userRepository.save(user);
    }

    // Evict single entry
    @CacheEvict(value = "users", key = "#id")
    public void deleteById(Long id) {
        userRepository.deleteById(id);
    }

    // Evict all
    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() { }

    // Combine multiple cache operations
    @Caching(
        cacheable = @Cacheable("users"),
        evict = @CacheEvict(value = "userLists", allEntries = true)
    )
    public User createUser(CreateUserRequest req) { ... }
}
```

### Redis Cache Configuration:
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users", config.entryTtl(Duration.ofHours(1)))
            .build();
    }
}
```

---

## 12. Interview Questions & Answers

---

### Q1: What is the difference between JdbcTemplate and Spring Data JPA?
**A:** `JdbcTemplate` is low-level — you write SQL, map result sets manually. Good for complex queries, stored procedures, or when you need full SQL control. `Spring Data JPA` is high-level — repositories auto-implement CRUD, derived queries from method names, JPQL. Less control, more productivity. JPA adds overhead (entity tracking, L1/L2 cache, lazy loading). Use JPA for CRUD-heavy apps; JdbcTemplate for complex reporting or legacy schemas.

---

### Q2: What is the N+1 problem and how do you solve it?
**A:** N+1 = 1 query to get N parent entities + N separate queries to get each entity's children. Example: fetch 100 users, then for each user fetch their orders = 101 queries. Solutions:
1. `JOIN FETCH` in JPQL: `SELECT u FROM User u LEFT JOIN FETCH u.orders`
2. `@EntityGraph` on repository method
3. `@BatchSize(size=N)` — Hibernate fetches N associations per query
4. DTO projections with JOIN query
5. Two separate queries (find parents, then find all children WHERE id IN (...))

---

### Q3: What is the difference between CascadeType.ALL and orphanRemoval?
**A:** `CascadeType.ALL` propagates all operations (PERSIST, MERGE, REMOVE, REFRESH, DETACH) from parent to children. `orphanRemoval = true` automatically deletes child entities when they're removed from the parent collection. Use both together to fully manage child lifecycle through parent. Warning: `CascadeType.REMOVE` deletes children when parent is deleted; `orphanRemoval` also deletes when removed from collection.

---

### Q4: Explain @Transactional propagation types.
**A:** Key ones:
- `REQUIRED` — joins existing or creates new (default, safe)
- `REQUIRES_NEW` — always new transaction (suspends outer); use for audit logs that must commit even if outer rolls back
- `NESTED` — savepoint within outer transaction; rollback to savepoint on failure; outer can still commit
- `NOT_SUPPORTED` — suspends transaction; use for long-running reads that shouldn't hold locks
- `MANDATORY` — must join existing; throws if called outside transaction (enforce contract)

---

### Q5: When does @Transactional NOT work (self-invocation problem)?
**A:** Same as AOP — calling a `@Transactional` method from within the same class using `this` bypasses the Spring proxy. Fix: inject `self` via `@Autowired`, use `ApplicationContext.getBean()`, or use AspectJ mode (`@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)`).

---

### Q6: What is optimistic vs pessimistic locking?
**A:** 
- **Optimistic** (`@Version`): assumes conflicts are rare; marks entity with version number; fails with `OptimisticLockException` if version changed since read. No DB locks held during transaction.
- **Pessimistic** (`@Lock(LockModeType.PESSIMISTIC_WRITE)`): immediately locks row with `SELECT FOR UPDATE`; other transactions wait. Prevents conflicts but reduces concurrency.

```java
@Entity
public class Account {
    @Version
    private Long version;  // Optimistic lock
}

@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);  // Pessimistic lock
```

---

### Q7: What is the difference between first-level and second-level cache in Hibernate?
**A:** **L1 Cache** (Session/EntityManager cache): per-transaction, always enabled, stores entities loaded in current session. `entityManager.find()` twice returns same instance. **L2 Cache** (SessionFactory cache): shared across transactions, optional, provider-specific (EhCache, Redis, Infinispan). Reduces database hits for frequently accessed, rarely changing data. Configure with `@Cacheable` and `@Cache` on entities.

---

### Q8: What is the difference between save() and saveAndFlush() in Spring Data JPA?
**A:** `save()` persists entity and schedules SQL for the next flush. `saveAndFlush()` immediately flushes to DB (executes SQL) while staying in same transaction. Use `saveAndFlush()` when you need to trigger DB constraints immediately, or when another part of the transaction reads what was just written.

---

### Q9: Explain the JPA entity lifecycle states.
**A:** 
- **Transient**: New object, not associated with EntityManager
- **Managed/Persistent**: Tracked by EntityManager; changes auto-persisted on flush
- **Detached**: Was managed, now EntityManager closed/cleared; changes not tracked
- **Removed**: Marked for deletion; DELETE on flush

```java
User user = new User("John");           // Transient
entityManager.persist(user);            // Managed
entityManager.detach(user);             // Detached
entityManager.merge(user);              // Back to Managed
entityManager.remove(user);             // Removed
```

---

### Q10: What are JPA inheritance strategies?
**A:** 
- **SINGLE_TABLE**: All subclasses in one table with discriminator column. Best performance, but nullable columns. `@Inheritance(strategy = SINGLE_TABLE)`
- **TABLE_PER_CLASS**: Each concrete class has its own table. Poor polymorphic queries (UNION).
- **JOINED**: Superclass + subclass tables joined by PK/FK. Normalized, but multiple JOINs.

---

*End of Chapter 7 — Spring Data JPA, JDBC & Transactions*
