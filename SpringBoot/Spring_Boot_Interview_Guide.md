# Spring Boot Microservices — REST API Complete Interview Deep Dive

> Covers Controllers · Services · Repositories · DTOs · Exception Handling · Versioning · Pagination

---

## #1 — REST API Design & Best Practices

---

### Core Principles — REST Constraints

**REST** = Representational State Transfer — an architectural style defined by 6 constraints:

| Constraint | What it means |
|---|---|
| 1. Client-Server | UI and backend are separated — independent evolution |
| 2. Stateless | No session stored on server — each request is self-contained |
| 3. Cacheable | Responses must declare whether they can be cached |
| 4. Uniform Interface | Standard HTTP methods & URIs across all resources |
| 5. Layered System | Client cannot tell if it communicates with server or proxy |
| 6. Code on Demand | *(Optional)* Server may send executable code to client |

---

### HTTP Methods — Correct Usage

| Method | Action | Idempotent | Body | Success Code |
|---|---|---|---|---|
| `GET` | Read / fetch | Yes | No | `200 OK` |
| `POST` | Create resource | No | Yes | `201 Created` |
| `PUT` | Full replace | Yes | Yes | `200 OK` |
| `PATCH` | Partial update | No | Yes | `200 OK` |
| `DELETE` | Remove resource | Yes | No | `204 No Content` |

---

### Spring Controller — All HTTP Methods

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    // GET — Read, Idempotent, Cacheable
    @GetMapping
    public ResponseEntity<List<UserDTO>> getAllUsers(
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        return ResponseEntity.ok(userService.getAllUsers(page, size, sortBy));
    }

    // GET by ID
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUserById(@PathVariable Long id) {
        return userService.findById(id)
                .map(ResponseEntity::ok)
                .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    // POST — Create, NOT idempotent
    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserDTO created = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}").buildAndExpand(created.getId()).toUri();
        return ResponseEntity.created(location).body(created); // 201 Created
    }

    // PUT — Full Update, Idempotent
    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> updateUser(
            @PathVariable Long id, @Valid @RequestBody UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }

    // PATCH — Partial Update
    @PatchMapping("/{id}")
    public ResponseEntity<UserDTO> patchUser(
            @PathVariable Long id, @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(userService.patch(id, updates));
    }

    // DELETE — Idempotent
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build(); // 204 No Content
    }
}
```

---

### HTTP Status Codes — Must Know

| Code | Meaning | When to use |
|---|---|---|
| `200 OK` | Success | GET, PUT, PATCH success |
| `201 Created` | Resource created | POST success — also set Location header |
| `204 No Content` | Success, no body | DELETE success |
| `400 Bad Request` | Client error | Invalid input, validation failed |
| `401 Unauthorized` | Not authenticated | Missing or invalid token |
| `403 Forbidden` | No permission | Authenticated but lacks access |
| `404 Not Found` | Missing resource | Resource does not exist |
| `409 Conflict` | State conflict | Duplicate email / concurrent update |
| `422 Unprocessable` | Semantic error | Field value invalid (e.g. age < 0) |
| `429 Too Many Requests` | Rate limited | Client exceeded request quota |
| `500 Internal Server Error` | Unexpected failure | Unhandled exception on server |
| `503 Service Unavailable` | Server down | Overloaded or in maintenance |

---

### Layered Architecture — Production Standard

```
Request
  │
  ▼
┌─────────────────────────────────────┐
│         Controller Layer            │  ← HTTP handling, request parsing, validation
│         @RestController             │
└──────────────┬──────────────────────┘
               │ DTO
               ▼
┌─────────────────────────────────────┐
│          Service Layer              │  ← Business logic, transaction coordination
│          @Service                   │
└──────────────┬──────────────────────┘
               │ Entity
               ▼
┌─────────────────────────────────────┐
│        Repository Layer             │  ← Data access, JPQL/SQL queries
│        @Repository / JpaRepository  │
└──────────────┬──────────────────────┘
               │
               ▼
        Database (MySQL / PostgreSQL)
```

> **Flow:** Request → Controller (DTO) → Service (Entity) → Repository → Database

---

### DTOs, Entity & Mapper

```java
// ── Request DTO (input) ──────────────────────────────────────────────────────
@Getter @Setter
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50)
    private String name;

    @NotBlank
    @Email(message = "Valid email required")
    private String email;

    @NotNull @Min(18) @Max(100)
    private Integer age;

    @Pattern(regexp = "^\\+?[1-9]\\d{9,14}$")
    private String phone;
}

// ── Response DTO (output) — NO sensitive data! ──────────────────────────────
@Getter @Builder
public class UserDTO {
    private Long          id;
    private String        name;
    private String        email;
    private int           age;
    private String        phone;
    private LocalDateTime createdAt;
    // ⚠  NO password field — never expose sensitive data!
}

// ── Entity ───────────────────────────────────────────────────────────────────
@Entity
@Table(name = "users")
@Getter @Setter
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    private int    age;
    private String phone;
    private String password; // ← never include in DTO!

    @CreationTimestamp private LocalDateTime createdAt;
    @UpdateTimestamp  private LocalDateTime updatedAt;
}

// ── Mapper ───────────────────────────────────────────────────────────────────
@Component
public class UserMapper {

    public UserDTO toDTO(User user) {
        return UserDTO.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .age(user.getAge())
                .phone(user.getPhone())
                .createdAt(user.getCreatedAt())
                .build();
    }

    public User toEntity(CreateUserRequest req) {
        User user = new User();
        user.setName(req.getName());
        user.setEmail(req.getEmail());
        user.setAge(req.getAge());
        user.setPhone(req.getPhone());
        return user;
    }
}
```

---

### Service & Repository

```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper     userMapper;

    @Transactional(readOnly = true)
    public Page<UserDTO> getAllUsers(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        return userRepository.findAll(pageable).map(userMapper::toDTO);
    }

    @Transactional(readOnly = true)
    public Optional<UserDTO> findById(Long id) {
        return userRepository.findById(id).map(userMapper::toDTO);
    }

    public UserDTO create(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.getEmail()))
            throw new DuplicateResourceException("Email already registered");
        User user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        return userMapper.toDTO(userRepository.save(user));
    }

    public void delete(Long id) {
        if (!userRepository.existsById(id))
            throw new ResourceNotFoundException("User", id);
        userRepository.deleteById(id);
    }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    boolean     existsByEmail(String email);
    Optional<User> findByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.age >= :minAge")
    List<User>  findUsersAboveAge(@Param("minAge") int minAge);
}
```

---

## Global Exception Handling — Production Must-Have

### Standard Error Response Shape

```java
@Getter @Builder
public class ErrorResponse {
    private int                  status;
    private String               error;
    private String               message;
    private String               path;
    private LocalDateTime        timestamp;
    private Map<String, String>  fieldErrors; // per-field validation errors
}
```

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 400 — @Valid validation failures
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));

        return ResponseEntity.badRequest().body(ErrorResponse.builder()
                .status(400).error("Validation Failed")
                .message("Input validation failed")
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now())
                .fieldErrors(fieldErrors).build());
    }

    // 404 — Resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        return ResponseEntity.status(404).body(ErrorResponse.builder()
                .status(404).error("Not Found").message(ex.getMessage())
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now()).build());
    }

    // 409 — Duplicate resource
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(
            DuplicateResourceException ex, HttpServletRequest request) {
        return ResponseEntity.status(409).body(ErrorResponse.builder()
                .status(409).error("Conflict").message(ex.getMessage())
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now()).build());
    }

    // 500 — Catch-all (never expose internals!)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error: ", ex); // log full stack trace internally
        return ResponseEntity.status(500).body(ErrorResponse.builder()
                .status(500).error("Internal Server Error")
                .message("Something went wrong") // safe generic message to client
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now()).build());
    }
}

// Custom domain exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id);
    }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

---

## API Versioning Strategies

| Strategy | Example | Pros / Cons |
|---|---|---|
| URI Versioning | `/api/v1/users` | Most common, visible, easy to test. Breaks REST purity. |
| Header Versioning | `API-Version: 1` | Clean URLs, flexible. Harder to test in browser. |
| Accept Header | `Accept: application/vnd.app.v1+json` | Most RESTful. Complex to implement. |

> **Recommendation:** Use URI versioning (`/api/v1/`) — most visible and widely adopted. Keep at least one previous version active and deprecate with a `Sunset` response header.

### URI Versioning — Code Example

```java
// v1 — original contract
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

// v2 — breaking change (new response shape)
@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }

// Header versioning alternative
@GetMapping(headers = "API-Version=1")
public ResponseEntity<UserDTOV1> getUserV1() { ... }

@GetMapping(headers = "API-Version=2")
public ResponseEntity<UserDTOV2> getUserV2() { ... }
```

---

## Pagination & Filtering — Standard Pattern

```java
@GetMapping
public ResponseEntity<PagedResponse<UserDTO>> getUsers(
        @RequestParam(defaultValue = "0")   int     page,
        @RequestParam(defaultValue = "10")  int     size,
        @RequestParam(defaultValue = "id")  String  sortBy,
        @RequestParam(defaultValue = "ASC") String  direction,
        @RequestParam(required = false)     String  name,
        @RequestParam(required = false)     Integer minAge) {

    Sort sort = direction.equalsIgnoreCase("DESC")
                ? Sort.by(sortBy).descending()
                : Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, size, sort);
    Page<UserDTO> users = userService.search(name, minAge, pageable);

    return ResponseEntity.ok(PagedResponse.<UserDTO>builder()
            .content(users.getContent())
            .page(users.getNumber())
            .size(users.getSize())
            .totalElements(users.getTotalElements())
            .totalPages(users.getTotalPages())
            .last(users.isLast())
            .build());
}

// Paged Response Wrapper
@Getter @Builder
public class PagedResponse<T> {
    private List<T> content;
    private int     page;
    private int     size;
    private long    totalElements;
    private int     totalPages;
    private boolean last;
}
```

---

## Top Interview Q&A

**Q: What is the difference between PUT and PATCH?**

> PUT replaces the entire resource — missing fields become null (full update). PATCH applies a partial update — only the provided fields change, the rest stay as-is.

---

**Q: Why use DTOs instead of exposing Entities directly?**

> Entities expose the DB schema, may contain sensitive fields (e.g. password), and tightly couple the API to the database structure. DTOs give full control over what is exposed and allow the API and DB to evolve independently.

---

**Q: How do you handle versioning in production?**

> URI versioning (`/api/v1/`) is most visible and widely adopted. Maintain at least one previous version for backward compatibility. Deprecate with a `Sunset` response header to warn clients.

---

**Q: What is idempotency and which HTTP methods are idempotent?**

> An operation is idempotent if calling it multiple times produces the same result. `GET`, `PUT`, `DELETE`, and `HEAD` are idempotent. `POST` is NOT — calling it twice creates two separate resources.

---

**Q: How do you secure REST APIs in Spring Boot?**

> JWT tokens in the `Authorization` header, HTTPS only, request validation with `@Valid`, rate limiting, CORS configuration, and never exposing stack traces or internal error details in error responses.

---

## REST API Production Checklist

- ✅ Use nouns in URIs, not verbs — `/users` not `/getUsers`
- ✅ Plural resource names — `/users` not `/user`
- ✅ Correct HTTP methods and status codes for every endpoint
- ✅ Request validation with `@Valid` on all POST and PUT bodies
- ✅ Global exception handling via `@RestControllerAdvice`
- ✅ DTOs everywhere — never expose JPA entities directly
- ✅ Pagination on all list endpoints — never return unbounded lists
- ✅ API versioning strategy defined and documented
- ✅ HTTPS only — never accept plain HTTP in production
- ✅ Rate limiting to protect against abuse
- ✅ API documentation via Swagger / OpenAPI
- ✅ Correlation ID header for distributed request tracing

---

*Suresh Sunuguri · Senior Software Engineer | Java · Spring Boot · Microservices*
